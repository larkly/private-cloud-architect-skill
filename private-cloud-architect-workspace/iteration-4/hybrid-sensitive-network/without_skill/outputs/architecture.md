# Hybrid Network Architecture: Norwegian Defense Contractor (BEGRENSET / AWS)

## 1. Executive Summary

This document describes a hybrid network architecture connecting an on-premises BEGRENSET-classified Kubernetes environment in Norway with AWS services (S3, SageMaker) in eu-north-1 (Stockholm). The design enforces data sovereignty under Sikkerhetsloven (the Norwegian Security Act) and NSM's guidelines for ICT security, ensuring that BEGRENSET-graded information never leaves Norwegian soil or transits to AWS, while still enabling controlled use of AWS cloud services for non-classified workloads.

---

## 2. Regulatory and Classification Context

### 2.1 Sikkerhetsloven (Security Act)

- Sections 6-1 through 6-5 govern information security for classified and security-graded information.
- BEGRENSET (Restricted) is the lowest classification grade but still requires protection against unauthorized access.
- Data graded BEGRENSET must be processed and stored within systems approved by the owning organization, with physical and logical controls.
- Cross-border transfer of classified information requires explicit authorization from NSM.

### 2.2 NSM Guidelines

- NSM's "Grunnprinsipper for IKT-sikkerhet" (Basic Principles for ICT Security) mandates defense-in-depth, least privilege, and network segmentation.
- NSM's cloud guidance states that BEGRENSET data may only reside in infrastructure under Norwegian jurisdiction and control.
- All interconnections between security domains must pass through approved gateways with logging and inspection.

### 2.3 Key Constraint

**BEGRENSET data must never enter AWS.** AWS is used exclusively for non-classified (UGRADERT) workloads. The architecture must enforce this at the network, application, and data layers.

---

## 3. Network Architecture

### 3.1 High-Level Topology

```
                        NORWAY (On-Prem, BEGRENSET)                    AWS eu-north-1 (UGRADERT only)
                   +------------------------------------+          +--------------------------------+
                   |                                    |          |                                |
                   |  +-----------+   +--------------+  |          |  +----------+  +------------+ |
                   |  | BEGRENSET |   | DMZ / Data   |  |  Direct  |  | Transit  |  | Workload   | |
                   |  | K8s       |-->| Diode &      |--|--Connect-|->| Gateway  |->| VPC        | |
                   |  | Cluster   |   | Sanitization |  |  (DX)    |  |          |  | (S3, SM)   | |
                   |  +-----------+   | Gateway      |  |          |  +----------+  +------------+ |
                   |       |          +--------------+  |          |       |                        |
                   |       v                |           |          |       v                        |
                   |  +-----------+   +-----------+    |          |  +-----------+                 |
                   |  | On-prem   |   | Monitoring |    |          |  | AWS       |                 |
                   |  | Security  |   | Collector  |    |          |  | Monitoring|                 |
                   |  | SIEM      |   +-----------+    |          |  | (CW, VPC  |                 |
                   |  +-----------+                    |          |  |  Flow)    |                 |
                   +------------------------------------+          +--------------------------------+
```

### 3.2 Network Zones

| Zone | Location | Classification | Purpose |
|------|----------|---------------|---------|
| **Zone 1: BEGRENSET Core** | On-prem Norway | BEGRENSET | Kubernetes cluster, databases, internal APIs |
| **Zone 2: Sanitization DMZ** | On-prem Norway | UGRADERT (outbound) | Data diode/gateway for classification enforcement |
| **Zone 3: Transit** | Direct Connect + AWS Transit GW | UGRADERT | Encrypted transport between on-prem and AWS |
| **Zone 4: AWS Workload VPC** | AWS eu-north-1 | UGRADERT | S3 buckets, SageMaker endpoints, PrivateLink |

### 3.3 Direct Connect Configuration

```
Direct Connect Link: 1 Gbps or 10 Gbps dedicated connection
Location:  Digiplex/Green Mountain Norway PoP --> AWS eu-north-1

Primary VIF:   Private Virtual Interface (private VIF)
               - VLAN tag: allocated by provider
               - BGP ASN (on-prem): 64512 (private ASN)
               - BGP ASN (AWS): 7224
               - Advertised prefixes from on-prem: DMZ subnet ONLY
                 (e.g., 10.200.0.0/24 - sanitization gateway)
               - DO NOT advertise BEGRENSET subnets to AWS

Redundancy:    Second DX connection via different PoP for failover
               (or DX SiteLink if available)

Encryption:    MACsec (IEEE 802.1AE) on the DX link for layer-2
               encryption, plus IPsec overlay for defense-in-depth
```

**Critical: The on-prem router must only advertise the DMZ/sanitization subnet to AWS via BGP. BEGRENSET network prefixes (e.g., 10.100.0.0/16) must never appear in BGP announcements to AWS.**

### 3.4 AWS-Side Network Design

```
AWS Transit Gateway
  |
  +-- DX Gateway (attached to Transit GW)
  |
  +-- Workload VPC (10.250.0.0/16)
  |     |
  |     +-- Private Subnet: SageMaker endpoints (no IGW)
  |     +-- Private Subnet: VPC Endpoints for S3 (Gateway endpoint)
  |     +-- Private Subnet: Interface endpoints (STS, KMS, CW Logs)
  |     +-- NO public subnets, NO Internet Gateway, NO NAT Gateway
  |
  +-- Inspection VPC (optional, for AWS Network Firewall)
        |
        +-- AWS Network Firewall with stateful rules
        +-- All traffic from DX routed through inspection first
```

**VPC Configuration:**

- **No internet access**: The workload VPC has no IGW or NAT GW. All AWS service access is via VPC Endpoints (PrivateLink).
- **S3 access**: Gateway VPC Endpoint for S3, restricted by bucket policy to the VPC endpoint only.
- **SageMaker**: Interface VPC Endpoint for SageMaker API and runtime, so training jobs stay within the VPC.
- **DNS**: Route 53 Resolver inbound/outbound endpoints for hybrid DNS resolution between on-prem and AWS.

### 3.5 On-Prem Sanitization Gateway (The Critical Component)

This is the enforcement point that prevents BEGRENSET data from reaching AWS.

```
BEGRENSET K8s Cluster
        |
        | (internal network, 10.100.0.0/16)
        v
+----------------------------------+
| SANITIZATION GATEWAY (DMZ)       |
| 10.200.0.0/24                    |
|                                  |
| Components:                      |
| 1. Data Classification Engine    |
|    - Scans all outbound payloads |
|    - Checks for BEGRENSET marks  |
|    - Regex for Norwegian PII,    |
|      defense keywords, markings  |
|    - ML-based DLP (on-prem)      |
|                                  |
| 2. API Gateway / Reverse Proxy   |
|    - Allowlist of permitted API  |
|      calls to AWS                |
|    - Request/response inspection |
|    - Mutual TLS termination      |
|    - Schema validation           |
|                                  |
| 3. Data Diode (optional HW)     |
|    - For highest assurance:      |
|      hardware-enforced one-way   |
|      data flow for specific      |
|      channels                    |
|                                  |
| 4. Logging Agent                 |
|    - Full packet capture of all  |
|      cross-boundary traffic      |
|    - Feeds to on-prem SIEM       |
+----------------------------------+
        |
        | (DMZ network, 10.200.0.0/24)
        v
   DX Router --> AWS
```

**Sanitization Rules:**

1. All outbound data must pass classification check before leaving Zone 2.
2. Any payload containing BEGRENSET markings (`BEGRENSET`, `RESTRICTED`, document classification headers) is blocked and an alert is raised.
3. Only pre-approved API operations are permitted (e.g., `s3:PutObject` to specific buckets, `sagemaker:CreateTrainingJob`).
4. Response data from AWS is inspected on return to ensure no injection or unexpected data.
5. Maximum payload sizes enforced; bulk data exfiltration prevention.

---

## 4. Data Flow Patterns

### 4.1 Pattern A: Non-Classified Training Data to SageMaker

```
1. Data scientist tags dataset as UGRADERT in on-prem data catalog
2. Two-person approval workflow confirms classification
3. Dataset exported from BEGRENSET cluster to sanitization gateway
4. Gateway scans dataset:
   - DLP engine checks for BEGRENSET content
   - Metadata headers verified as UGRADERT
   - File checksums logged
5. Dataset pushed to S3 via DX (encrypted with customer-managed KMS key)
6. SageMaker training job runs in VPC-mode (no internet)
7. Model artifacts stored in S3
8. Model metadata (not weights) returned to on-prem for evaluation
```

### 4.2 Pattern B: On-Prem API Called by AWS Service

```
1. SageMaker or Lambda in AWS needs to call on-prem API
2. Request routes: AWS VPC --> Transit GW --> DX --> On-prem DMZ
3. Sanitization gateway receives request:
   - Validates source (AWS VPC CIDR, mTLS certificate)
   - Checks request against API allowlist
   - Proxies to internal K8s service ONLY if permitted
4. K8s service processes request using BEGRENSET data internally
5. Response generated with ONLY non-classified output
6. Gateway scans response before forwarding to AWS
7. Full request/response logged to SIEM
```

### 4.3 Pattern C: Blocked Flow (BEGRENSET Data Attempted Egress)

```
1. Application attempts to send BEGRENSET-marked data to S3
2. Sanitization gateway DLP engine detects classification marking
3. Request is BLOCKED
4. Alert sent to:
   - On-prem SIEM (immediate)
   - Security operations team (immediate)
   - Sikkerhetssjef (security officer) notification
5. Incident logged with full payload capture for forensics
6. Source pod/service in K8s flagged for review
```

---

## 5. Security Controls

### 5.1 Network-Level Controls

| Control | Implementation | Layer |
|---------|---------------|-------|
| Network segmentation | Physically separate BEGRENSET and DMZ networks; firewall between zones | L2/L3 |
| BGP prefix filtering | Only DMZ prefix advertised to AWS; BEGRENSET prefixes never leave on-prem | L3 |
| MACsec on DX | Layer 2 encryption on Direct Connect | L2 |
| IPsec overlay | Site-to-site VPN over DX for defense-in-depth | L3 |
| AWS Security Groups | Restrictive SGs allowing only on-prem DMZ CIDR | L4 |
| AWS Network Firewall | Stateful inspection in inspection VPC | L4/L7 |
| On-prem firewall | Palo Alto / Stormshield (NSM-approved) between all zones | L3-L7 |
| No internet egress | AWS VPC has no IGW/NAT; on-prem BEGRENSET has no internet | L3 |

### 5.2 Identity and Access

```
On-Prem:
  - Active Directory / LDAP for BEGRENSET users
  - Smart card / BankID authentication for admin access
  - RBAC in Kubernetes with namespace isolation per project

AWS:
  - Dedicated AWS Organization, no connection to other accounts
  - IAM roles with minimal permissions
  - STS temporary credentials only (no long-lived keys)
  - AWS IAM Identity Center (SSO) federated from on-prem IdP
  - SCPs (Service Control Policies) restricting:
    * Regions: ONLY eu-north-1 allowed
    * Services: ONLY S3, SageMaker, KMS, CloudWatch, VPC, DX
    * No public S3 buckets (s3:PutBucketPublicAccessBlock enforced)
```

### 5.3 Encryption

| Scope | Mechanism | Key Management |
|-------|-----------|----------------|
| Data at rest (on-prem) | LUKS full-disk, etcd encryption | On-prem HSM (e.g., Thales Luna) |
| Data at rest (AWS S3) | SSE-KMS with customer-managed CMK | AWS KMS (eu-north-1), key policy restricts to VPC endpoint |
| Data in transit (DX) | MACsec + IPsec | Pre-shared keys rotated quarterly |
| Data in transit (app layer) | mTLS between gateway and AWS endpoints | Certificates from on-prem PKI |
| SageMaker training | VPC-mode, encrypted volumes | KMS CMK |

### 5.4 AWS Service Control Policies (SCPs)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideEuNorth1",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "eu-north-1"
        },
        "ForAnyValue:StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/OrganizationAdmin"
          ]
        }
      }
    },
    {
      "Sid": "DenyPublicS3",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:PutAccountPublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "true"
        }
      }
    },
    {
      "Sid": "DenyInternetGateway",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:CreateNatGateway"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5.5 S3 Bucket Policy (VPC Endpoint Restriction)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAccessOutsideVPCEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::defense-ugradert-training-data",
        "arn:aws:s3:::defense-ugradert-training-data/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0123456789abcdef0"
        }
      }
    }
  ]
}
```

---

## 6. Kubernetes On-Prem Architecture

### 6.1 Cluster Layout

```
BEGRENSET Kubernetes Cluster (e.g., RKE2 / Tanzu on bare metal)
|
+-- Namespace: begrenset-apps       (BEGRENSET workloads, no egress)
|   +-- NetworkPolicy: deny all egress except intra-namespace
|   +-- PodSecurityAdmission: restricted
|
+-- Namespace: sanitization         (gateway services)
|   +-- NetworkPolicy: allow egress ONLY to DMZ subnet (10.200.0.0/24)
|   +-- DLP sidecar containers on all pods
|   +-- API gateway (e.g., Kong/Envoy with custom DLP plugin)
|
+-- Namespace: monitoring           (Prometheus, Grafana, log collectors)
|   +-- NetworkPolicy: allow ingress from all namespaces (metrics scrape)
|   +-- No egress to DMZ or AWS
|
+-- Namespace: system               (cert-manager, vault, OPA/Gatekeeper)
    +-- OPA policies enforcing:
        - No BEGRENSET namespace pod can have egress to DMZ
        - All cross-namespace traffic must go through sanitization
        - Container images must come from on-prem registry only
```

### 6.2 OPA/Gatekeeper Policy Example

```rego
package kubernetes.admission

deny[msg] {
    input.request.object.kind == "NetworkPolicy"
    input.request.object.metadata.namespace == "begrenset-apps"
    some i
    cidr := input.request.object.spec.egress[i].to[_].ipBlock.cidr
    not startswith(cidr, "10.100.")
    msg := sprintf("BEGRENSET namespace cannot have egress to non-BEGRENSET CIDR: %v", [cidr])
}
```

---

## 7. Monitoring and Observability

### 7.1 Architecture Overview

```
+---------------------+       +--------------------+
|   ON-PREM SIEM      |       |   AWS MONITORING   |
|   (Splunk/Elastic)  |<------| (collected via DX) |
|                     |       |                    |
| Sources:            |       | Sources:           |
| - K8s audit logs    |       | - VPC Flow Logs    |
| - Gateway DLP logs  |       | - CloudTrail       |
| - Firewall logs     |       | - S3 access logs   |
| - DX BGP state      |       | - SageMaker logs   |
| - Packet captures   |       | - CW Metrics       |
| - NSM audit trail   |       | - Config changes   |
+---------------------+       +--------------------+
         |                              |
         v                              v
+------------------------------------------------+
|          UNIFIED DASHBOARD (On-Prem)           |
|          Grafana + Custom Dashboards            |
|                                                |
|  Panels:                                       |
|  - Cross-boundary traffic volume (bytes/pkt)   |
|  - DLP block events (real-time)                |
|  - DX link health (latency, BGP state)         |
|  - AWS resource usage (S3, SageMaker)          |
|  - Anomaly detection (unusual traffic flows)   |
|  - Classification breach attempts              |
+------------------------------------------------+
```

### 7.2 On-Prem Monitoring Stack

| Component | Tool | Purpose |
|-----------|------|---------|
| Metrics | Prometheus + Thanos | K8s cluster metrics, node metrics, custom sanitization gateway metrics |
| Logs | Fluentd/Fluent Bit --> Elasticsearch | Centralized logging from all namespaces |
| Traces | Jaeger/OpenTelemetry | Distributed tracing across on-prem services and cross-boundary calls |
| Network | Zeek (formerly Bro) + Suricata | Deep packet inspection on DMZ interfaces |
| SIEM | Splunk Enterprise / Elastic SIEM | Correlation, alerting, compliance reporting |
| Dashboards | Grafana | Unified visualization |

### 7.3 AWS Monitoring (Forwarded to On-Prem)

All AWS monitoring data is forwarded to the on-prem SIEM via Direct Connect. No AWS-side dashboards are used for security monitoring (single pane of glass on-prem).

```
AWS CloudWatch Logs --> CloudWatch Logs Subscription Filter
    --> Kinesis Data Firehose (VPC endpoint)
    --> S3 staging bucket (VPC endpoint)
    --> On-prem log collector pulls via DX

AWS CloudTrail --> S3 bucket --> On-prem pull via DX

VPC Flow Logs --> CloudWatch Logs --> Same pipeline as above

AWS Config --> SNS --> SQS (VPC endpoint) --> On-prem consumer via DX
```

**Forwarded AWS data sources:**

1. **VPC Flow Logs** (all subnets, all traffic including rejected): Shows every network flow to/from on-prem.
2. **CloudTrail** (management + data events for S3 and SageMaker): Shows all API calls, who did what.
3. **S3 Server Access Logs**: Object-level read/write audit trail.
4. **SageMaker Experiment Logs**: Training job inputs, outputs, parameters.
5. **AWS Config**: Tracks configuration drift (e.g., someone modifies a security group).
6. **GuardDuty findings**: AWS-native threat detection findings forwarded for correlation.

### 7.4 Cross-Boundary Traffic Dashboard (Grafana)

Key panels for the unified dashboard:

```
Row 1: DX Link Health
  - Panel: DX connection state (up/down, BGP peer status)
  - Panel: DX throughput (Mbps in/out, 5-min average)
  - Panel: DX latency (round-trip time from gateway to AWS TGW)

Row 2: Traffic Flow Analysis
  - Panel: Bytes transferred on-prem --> AWS (per hour, per day)
  - Panel: Bytes transferred AWS --> on-prem (per hour, per day)
  - Panel: Top 10 source/destination IP pairs crossing boundary
  - Panel: Connection count by protocol and port

Row 3: Security Events
  - Panel: DLP block events (count, last 24h, trend)
  - Panel: Classification breach attempts (critical alert)
  - Panel: Failed authentication attempts (mTLS failures)
  - Panel: Anomaly score (ML-based baseline deviation)

Row 4: AWS Service Usage
  - Panel: S3 objects uploaded (count/size per day)
  - Panel: SageMaker training hours consumed
  - Panel: IAM authentication events (success/fail)
  - Panel: AWS Config compliance score
```

### 7.5 Alerting Rules

| Alert | Condition | Severity | Response |
|-------|-----------|----------|----------|
| DLP Block | Any BEGRENSET content detected in outbound traffic | CRITICAL | Immediate SOC response, block source pod |
| BGP Prefix Leak | BEGRENSET prefix appears in BGP announcements | CRITICAL | Auto-shutdown DX interface, page network team |
| DX Link Down | Both DX connections down > 5 min | HIGH | Failover verification, notify operations |
| Unusual Data Volume | Outbound data to AWS exceeds 2x 7-day rolling average | HIGH | SOC investigation |
| AWS Config Drift | Security group or VPC endpoint modified | HIGH | Auto-remediate via AWS Config rule |
| CloudTrail Gap | No CloudTrail events for > 15 min | MEDIUM | Verify CloudTrail health |
| mTLS Cert Expiry | Certificate expires within 30 days | MEDIUM | Auto-renewal trigger |
| SageMaker Internet Access | Training job attempts internet access | CRITICAL | Job terminated, alert raised |

### 7.6 Prometheus Metrics for Sanitization Gateway

```yaml
# Custom metrics exposed by the sanitization gateway
sanitization_gateway_requests_total{direction="outbound|inbound", status="allowed|blocked", reason="..."}
sanitization_gateway_dlp_scan_duration_seconds{scanner="regex|ml|metadata"}
sanitization_gateway_payload_bytes{direction="outbound|inbound"}
sanitization_gateway_classification_detections_total{classification="BEGRENSET|KONFIDENSIELT|UGRADERT"}
sanitization_gateway_active_connections{destination="aws|onprem"}
sanitization_gateway_mtls_handshake_failures_total{}
```

---

## 8. Compliance Mapping

| Requirement (NSM/Sikkerhetsloven) | Implementation |
|----------------------------------|----------------|
| Classified data must not leave approved systems | Sanitization gateway with DLP; BEGRENSET prefixes never advertised to AWS |
| Defense-in-depth | MACsec + IPsec + mTLS + DLP + firewall + Network Firewall + SCPs |
| Least privilege | K8s RBAC, AWS IAM minimal roles, SCP region/service restrictions |
| Logging and auditability | Full packet capture at boundary, CloudTrail, VPC Flow Logs, SIEM correlation |
| Incident response | Automated alerting, DLP block-and-alert, BGP auto-shutdown |
| Access control | mTLS, smart card auth, federated SSO, no long-lived AWS credentials |
| Data sovereignty | BEGRENSET data stays on-prem in Norway; AWS restricted to eu-north-1 (Stockholm) for UGRADERT only |
| Physical security | On-prem in NSM-approved facility; AWS DX via Norwegian PoP |
| Configuration management | OPA/Gatekeeper for K8s; AWS Config for cloud resources; drift detection |
| Periodic review | Quarterly access reviews; annual penetration test of boundary; NSM audit support |

---

## 9. Deployment Recommendations

### 9.1 Phase 1: Foundation (Weeks 1-6)
1. Establish DX connections (primary + redundant) with MACsec.
2. Deploy Transit Gateway and Workload VPC with no internet access.
3. Configure VPC Endpoints for S3, SageMaker, KMS, CloudWatch, STS.
4. Deploy SCPs and AWS Config rules.
5. Set up on-prem DMZ network segment with firewalls.

### 9.2 Phase 2: Sanitization Gateway (Weeks 4-10)
1. Deploy API gateway (Kong/Envoy) in sanitization namespace.
2. Implement DLP scanning engine with Norwegian defense keyword lists.
3. Configure mTLS between gateway and AWS endpoints.
4. Establish OPA/Gatekeeper policies in K8s.
5. Test with synthetic data containing BEGRENSET markings (must be blocked).

### 9.3 Phase 3: Monitoring (Weeks 8-14)
1. Deploy Prometheus, Grafana, and Elasticsearch on-prem.
2. Configure AWS log forwarding pipeline (CloudTrail, VPC Flow Logs, etc.).
3. Build unified Grafana dashboards.
4. Implement alerting rules and test with simulated incidents.
5. Deploy Zeek/Suricata on DMZ interfaces.

### 9.4 Phase 4: Validation (Weeks 12-16)
1. Penetration test of the boundary (internal red team + external NSM-approved firm).
2. DLP evasion testing (attempt to exfiltrate BEGRENSET data through encoding, steganography, etc.).
3. Failover testing (DX link failure, gateway failure).
4. NSM compliance audit preparation and documentation.
5. Tabletop exercise for classification breach incident.

---

## 10. Risk Residuals and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| DLP bypass via encoding/encryption of BEGRENSET data | Medium | High | ML-based DLP, mandatory decryption at gateway, behavioral analysis |
| AWS Stockholm region data request by foreign authority | Low | High | Data in AWS is UGRADERT only; SCP prevents region spread; legal review with AWS Enterprise agreement |
| DX link compromise | Low | High | MACsec + IPsec; physical security of PoP; BGP monitoring |
| Insider threat exfiltrating data via approved channel | Medium | High | Two-person approval for data transfers; anomaly detection on volumes; audit trail |
| AWS service misconfiguration exposing data | Medium | Medium | AWS Config auto-remediation; SCP guardrails; no internet access in VPC |

---

## 11. Summary

This architecture enforces a strict BEGRENSET/UGRADERT boundary at the on-premises sanitization gateway. AWS never sees classified data. The Direct Connect link provides private, encrypted connectivity, and the AWS environment is locked down with no internet access, VPC endpoints only, and restrictive SCPs. Monitoring converges on a single on-prem SIEM with full visibility across both environments. The design aligns with Sikkerhetsloven and NSM guidelines by keeping BEGRENSET data in Norway, enforcing defense-in-depth, and maintaining comprehensive audit trails.
