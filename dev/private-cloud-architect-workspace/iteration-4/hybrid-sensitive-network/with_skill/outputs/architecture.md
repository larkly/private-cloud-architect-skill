# Hybrid Network Architecture: Norwegian Defense Contractor with AWS Connectivity

## Classification and Compliance Context

**On-premises classification level**: BEGRENSET (restricted) per Sikkerhetsloven (Norwegian Security Act, 2018)
**Applicable frameworks**:
- Sikkerhetsloven and its regulations (Virksomhetsikkerhetsforskriften, Sikkerhetsgraderte anskaffelser)
- NSM Grunnprinsipper for IKT-sikkerhet (primary ICT security baseline)
- GDPR and NIS2 (Norway is EEA member)
- NATO interoperability requirements (Norway is a founding NATO member)

**AWS region**: eu-north-1 (Stockholm)
**Connectivity**: AWS Direct Connect dedicated link to eu-north-1

**Fundamental constraint**: BEGRENSET data must remain in Norway at all times. AWS services handle only non-classified data. The on-premises network is the trust anchor; AWS is treated as untrusted.

---

## Architecture Principles

Mapped to NSM Grunnprinsipper categories:

### Identifisere (Identify)
1. All assets across both environments are inventoried in NetBox (on-prem DCIM/IPAM) and AWS Config (cloud side).
2. Data classification is enforced at the application layer: every data object is tagged BEGRENSET or UGRADERT before it leaves the application.
3. Risk assessment covers the hybrid boundary as a distinct attack surface.

### Beskytte (Protect)
4. The on-prem network sets the security bar. Cloud connectivity must meet BEGRENSET handling requirements at the boundary.
5. Never extend the BEGRENSET boundary to AWS. Create a well-defined exchange zone where data crosses between sensitivity levels.
6. Encrypt everything in transit with NSM-approved or industry-standard cryptography (IPsec, TLS 1.3).
7. Treat AWS as untrusted. Design as if the cloud side is compromised.
8. Minimize the attack surface: only expose the minimum required services and ports across the hybrid boundary.

### Oppdage (Detect)
9. Unified observability across both environments with full traffic flow visibility.
10. All hybrid boundary traffic is logged, inspected, and retained per Sikkerhetsloven requirements.

### Handtere (Respond)
11. Ability to sever the hybrid link within minutes. On-prem continues operating independently.
12. Incident response playbooks span both environments.

---

## Network Architecture

### Zone Model

```
+============================================================================+
|                          ON-PREMISES (NORWAY)                              |
|                                                                            |
|  +------------------------------+    +-------------------------------+     |
|  |     BEGRENSET ZONE           |    |     EXCHANGE ZONE             |     |
|  |     (Restricted Network)     |    |     (UGRADERT / Non-classified)|    |
|  |                              |    |                               |     |
|  |  Kubernetes cluster          |    |  Security Gateway Appliance   |     |
|  |  Internal APIs               |--->|  (NSM-approved or hardened    |     |
|  |  Databases                   |<---|   firewall/proxy)             |     |
|  |  BEGRENSET workloads         |    |                               |     |
|  |                              |    |  API Gateway (Kong/APISIX)    |     |
|  |  Network: 10.10.0.0/16      |    |  Data sanitization service    |     |
|  |  Cilium default-deny         |    |  Audit logging               |     |
|  +------------------------------+    |                               |     |
|                                      |  Network: 10.20.0.0/16       |     |
|                                      +-------------------------------+     |
|                                                   |                        |
|                                      +-------------------------------+     |
|                                      |  DIRECT CONNECT TERMINATION   |     |
|                                      |  NSM-approved VPN appliance   |     |
|                                      |  (IPsec over Direct Connect)  |     |
|                                      +-------------------------------+     |
|                                                   |                        |
+===================================================|========================+
                                                    | AWS Direct Connect
                                                    | (dedicated 1/10 Gbps)
                                                    | eu-north-1 (Stockholm)
+===================================================|========================+
|                              AWS                   |                        |
|                                      +-------------------------------+     |
|                                      |  Direct Connect Gateway       |     |
|                                      |  + Virtual Private Gateway    |     |
|                                      +-------------------------------+     |
|                                                   |                        |
|  +-------------------------------+   +-------------------------------+     |
|  |  HYBRID VPC                   |   |  ISOLATED VPC                 |     |
|  |  (On-prem connected)          |   |  (No on-prem connectivity)    |     |
|  |                               |   |                               |     |
|  |  Private subnets only         |   |  SageMaker training jobs      |     |
|  |  No internet gateway          |   |  S3 buckets (non-classified)  |     |
|  |  VPC Endpoints for S3,        |   |  Internet Gateway (for ML     |     |
|  |   SageMaker, CloudWatch,      |   |   model downloads only)       |     |
|  |   STS, KMS                    |   |                               |     |
|  |                               |   |  Network: 10.40.0.0/16       |     |
|  |  Transit Gateway link to      |---|                               |     |
|  |   Isolated VPC (controlled)   |   +-------------------------------+     |
|  |                               |                                         |
|  |  Network: 10.30.0.0/16       |                                         |
|  +-------------------------------+                                         |
|                                                                            |
+============================================================================+
```

### Zone Descriptions

#### 1. BEGRENSET Zone (On-Prem, 10.10.0.0/16)

The core on-premises Kubernetes cluster handling BEGRENSET workloads.

- **Platform**: Kubernetes (RKE2 or Talos Linux for hardened, immutable nodes)
- **CNI**: Cilium with default-deny NetworkPolicy and identity-based policy enforcement
- **Storage**: Rook-Ceph for persistent volumes, encrypted at rest (LUKS/dm-crypt)
- **Identity**: FreeIPA or Keycloak integrated with organizational directory; MFA mandatory
- **Secrets**: HashiCorp Vault (self-hosted) or Sealed Secrets
- **All BEGRENSET data resides here. It never leaves this zone in classified form.**

#### 2. Exchange Zone (On-Prem, 10.20.0.0/16)

A physically separate network segment on-prem that bridges the BEGRENSET zone and the AWS connection. This is the critical architectural element.

- **Security gateway**: Hardened firewall appliance (OPNsense with Suricata IDS/IPS, or NSM-approved appliance if mandated) enforcing:
  - Stateful inspection of all traffic
  - Application-layer protocol validation
  - Content inspection to prevent BEGRENSET data leakage
  - Allowlist-only rules (no default-permit)
- **API gateway**: Kong or Apache APISIX performing:
  - Authentication of all requests (mTLS + JWT/OAuth2)
  - Request/response payload inspection and sanitization
  - Rate limiting and throttling
  - Request logging with full payload capture for audit
- **Data sanitization service**: Custom application that:
  - Validates outbound data has UGRADERT classification tag
  - Strips any BEGRENSET metadata before data leaves Norway
  - Enforces data loss prevention (DLP) rules
  - Logs every data transfer with classification, timestamp, source, destination, user, and approval reference
- **Firewall rules between BEGRENSET zone and Exchange zone**:
  - Only specific API endpoints on specific ports are reachable (e.g., TCP 8443 to the API gateway)
  - No direct database access from Exchange zone
  - No SSH/management traffic across this boundary (separate management VLAN)

#### 3. Direct Connect Termination (On-Prem)

- **NSM-approved VPN appliance** terminates IPsec over the Direct Connect link
  - Even though Direct Connect is a private link, it transits non-Norwegian infrastructure (Stockholm). IPsec encryption is mandatory.
  - Use AES-256-GCM with IKEv2; verify crypto selection against NSM's current approved list
  - If NSM requires specific VPN appliances for BEGRENSET-adjacent connectivity, deploy those (consult NSM directly)
- **BGP peering** over the VPN tunnel to AWS Virtual Private Gateway
  - Advertise only the Exchange Zone prefix (10.20.0.0/16) to AWS
  - Never advertise the BEGRENSET zone prefix (10.10.0.0/16)
  - Implement BGP prefix filters on both sides to prevent route leaks

#### 4. Hybrid VPC (AWS, 10.30.0.0/16)

The AWS VPC that has connectivity back to on-prem via Direct Connect.

- **Private subnets only** -- no public subnets, no Internet Gateway, no NAT Gateway
- **VPC Endpoints (PrivateLink)** for all required AWS services:
  - `com.amazonaws.eu-north-1.s3` (Gateway endpoint)
  - `com.amazonaws.eu-north-1.sagemaker.api` (Interface endpoint)
  - `com.amazonaws.eu-north-1.sagemaker.runtime` (Interface endpoint)
  - `com.amazonaws.eu-north-1.monitoring` (Interface endpoint, for CloudWatch)
  - `com.amazonaws.eu-north-1.logs` (Interface endpoint, for CloudWatch Logs)
  - `com.amazonaws.eu-north-1.sts` (Interface endpoint)
  - `com.amazonaws.eu-north-1.kms` (Interface endpoint)
- **Security Groups**: Allow only traffic from on-prem Exchange Zone CIDR (10.20.0.0/16) and deny all else
- **No workloads run here permanently** -- this VPC is a transit and API access layer
- **AWS Transit Gateway** connects Hybrid VPC to Isolated VPC with route table controls

#### 5. Isolated VPC (AWS, 10.40.0.0/16)

Where actual AWS workloads (S3, SageMaker) operate.

- **S3 buckets**:
  - Bucket policy restricts access to Hybrid VPC endpoint only (aws:sourceVpce condition)
  - Server-side encryption with AWS KMS (customer-managed key)
  - Versioning and access logging enabled
  - S3 Object Lock for compliance retention if required
  - **Only non-classified data** -- training datasets, model artifacts, large non-sensitive files
- **SageMaker**:
  - Training jobs run in VPC mode (SageMaker attaches ENIs in the Isolated VPC)
  - No direct internet access for training instances; model artifacts pulled from S3
  - SageMaker execution role follows least privilege (IAM policy restricts to specific S3 paths and KMS keys)
  - Training data is non-classified, pre-sanitized on-prem before upload
- **Transit Gateway route table**: Only allows traffic from Hybrid VPC; no direct on-prem routes

---

## Data Flow Architecture

### Flow 1: On-prem uploads non-classified training data to S3

```
BEGRENSET Zone App                    Exchange Zone                     AWS
      |                                     |                             |
      |  1. App prepares UGRADERT           |                             |
      |     training dataset                |                             |
      |                                     |                             |
      |  2. Sanitization service            |                             |
      |     validates classification        |                             |
      |     tag = UGRADERT                  |                             |
      |                                     |                             |
      |  3. API GW authenticates --------->  |                             |
      |     (mTLS + token)                  |                             |
      |                                     |  4. IPsec tunnel            |
      |                                     |----------------------------> |
      |                                     |     to Hybrid VPC           |
      |                                     |                             |
      |                                     |  5. S3 PutObject via        |
      |                                     |     VPC Endpoint            |
      |                                     |                      [S3 bucket]
```

### Flow 2: AWS SageMaker training job requests data from on-prem API

```
SageMaker (Isolated VPC)       Hybrid VPC              Exchange Zone        BEGRENSET Zone
      |                           |                          |                    |
      |  1. Training callback     |                          |                    |
      |     needs feature data    |                          |                    |
      |                           |                          |                    |
      |  2. Transit GW route ---> |                          |                    |
      |                           |  3. IPsec tunnel ------> |                    |
      |                           |                          |                    |
      |                           |     4. API GW validates  |                    |
      |                           |        request (mTLS,    |                    |
      |                           |        payload check)    |                    |
      |                           |                          |                    |
      |                           |     5. Sanitization svc  |  6. Fetch from    |
      |                           |        confirms response |     on-prem API   |
      |                           |        is UGRADERT       |<------------------>|
      |                           |                          |                    |
      |  7. Response returns <----|<-------------------------|                    |
      |     (non-classified only) |                          |                    |
```

### Flow 3: Data that must NOT flow

- BEGRENSET-tagged database records must never transit the Exchange Zone outbound
- Personnel records, clearance data, classified project data: blocked by DLP rules in the sanitization service
- Direct Kubernetes API access from AWS: blocked at firewall (management plane is isolated)
- Any traffic from AWS to 10.10.0.0/16: no route exists; BGP never advertises this prefix

---

## Identity and Access Control

### On-Premises
- **FreeIPA** or **Keycloak** as the authoritative identity provider
- All personnel accessing BEGRENSET systems hold appropriate klarering (security clearance)
- MFA enforced for all access (hardware tokens preferred; TOTP minimum)
- RBAC mapped to need-to-know principle

### Hybrid Identity Federation
- On-prem Keycloak/FreeIPA federated to AWS IAM Identity Center via SAML 2.0
- AWS IAM roles scoped to specific services (S3, SageMaker) with least privilege
- No permanent AWS IAM users; all access via assumed roles with session duration limits
- AWS STS conditions:
  - `aws:SourceVpc` restricts role assumption to Hybrid VPC
  - `aws:PrincipalTag/classification` enforced in IAM policies

### Service-to-Service Authentication
- mTLS between Exchange Zone and AWS (certificate chain rooted in on-prem CA)
- cert-manager on Kubernetes issues and rotates certificates automatically
- API keys for S3 access rotated via Vault and injected as Kubernetes secrets

---

## AWS Account Structure

```
AWS Organization
|
+-- Security OU
|   +-- Log Archive account (CloudTrail, Config, VPC Flow Logs -- immutable)
|   +-- Security Tooling account (GuardDuty, Security Hub aggregation)
|
+-- Workload OU
    +-- Hybrid-Connected account (Hybrid VPC, Direct Connect termination)
    +-- ML-Workload account (Isolated VPC, SageMaker, S3)
```

- **AWS Organizations SCPs** enforce:
  - Deny all regions except eu-north-1 (`aws:RequestedRegion` condition)
  - Deny creation of public S3 buckets
  - Deny disabling CloudTrail or VPC Flow Logs
  - Deny creation of IAM users with console access
  - Require encryption on all storage services

---

## Monitoring and Observability

### Design Goal
A single operational view showing traffic flows, security events, and resource health across both on-prem and AWS, fulfilling the NSM Grunnprinsipper "Oppdage" (Detect) requirements.

### On-Premises Monitoring Stack

| Component | Tool | Purpose |
|---|---|---|
| Metrics | Prometheus + Thanos | Cluster metrics, node metrics, application metrics |
| Logs | Loki | Centralized log aggregation |
| Traces | Tempo or Jaeger | Distributed tracing for API calls |
| Dashboards | Grafana | Unified dashboards |
| Network flows | Suricata + pmacct/ntopng | Deep packet inspection and NetFlow/sFlow collection |
| SIEM | Wazuh | Security event correlation, alerting, compliance |
| Alerting | Alertmanager | Routing to on-call (PagerDuty/Opsgenie if non-classified, or self-hosted) |
| Runtime security | Falco or Tetragon | Kubernetes runtime behavior monitoring |
| Network monitoring | LibreNMS | Switch/router/firewall health and interface utilization |

### AWS Monitoring

| Component | Tool | Purpose |
|---|---|---|
| API audit | CloudTrail | All AWS API calls logged to Log Archive account |
| VPC traffic | VPC Flow Logs | All network flows in both VPCs |
| Metrics | CloudWatch | AWS service metrics (S3, SageMaker, Direct Connect) |
| Security posture | AWS Security Hub + GuardDuty | Threat detection and security findings |
| Compliance | AWS Config | Resource configuration compliance |
| Cost | AWS Cost Explorer + Budgets | Spend tracking and alerting |

### Unified Hybrid Observability

#### Metrics Federation
```
On-Prem Prometheus -----> Thanos Sidecar -----> Thanos Query (on-prem)
                                                       ^
                                                       |
AWS CloudWatch -----> CloudWatch Exporter -----> Prometheus (AWS) -----> Thanos Sidecar ---+
(runs in Hybrid VPC)   (Prometheus remote_write to on-prem Thanos       |
                        via Direct Connect)                              |
                                                                         v
                                                              Grafana (on-prem)
                                                              Single pane of glass
```

- **Prometheus in AWS** (lightweight, runs in Hybrid VPC) scrapes CloudWatch Exporter and VPC Flow Log metrics
- **remote_write** sends AWS metrics to on-prem Thanos over the encrypted Direct Connect tunnel
- **Thanos Query** federates on-prem and AWS metrics into a single queryable store
- **Grafana on-prem** provides unified dashboards; no Grafana instance in AWS

#### Log Aggregation
- AWS CloudTrail and VPC Flow Logs ship to S3 in the Log Archive account
- A Fluentd/Fluent Bit forwarder in Hybrid VPC reads from S3/CloudWatch Logs and forwards to on-prem Loki over the Direct Connect tunnel
- On-prem Loki stores all logs (both environments) with appropriate retention
- Wazuh agents parse AWS logs for security correlation alongside on-prem events

#### Traffic Flow Visibility Dashboard (Grafana)

**Dashboard: Hybrid Network Traffic Flows**

Panel 1: **Exchange Zone Throughput**
- Bytes in/out per second across the Exchange Zone boundary
- Source: Suricata eve.json via Loki, pmacct NetFlow data via Prometheus

Panel 2: **Direct Connect Link Health**
- BGP session state, link utilization, packet loss, latency
- Source: AWS CloudWatch `dx-*` metrics via CloudWatch Exporter, on-prem router SNMP via LibreNMS

Panel 3: **Cross-Boundary API Calls**
- Request rate, latency (p50/p95/p99), error rate for all API calls crossing the hybrid boundary
- Source: Kong/APISIX metrics (Prometheus), application-level OpenTelemetry traces

Panel 4: **Data Classification Flow**
- Volume of UGRADERT data transferred to/from AWS per hour
- Count of BEGRENSET data blocked by sanitization service (should be zero in normal operation; any non-zero triggers alert)
- Source: Sanitization service custom metrics via Prometheus

Panel 5: **AWS VPC Flow Summary**
- Top talkers, denied flows, unusual port access in Hybrid and Isolated VPCs
- Source: VPC Flow Logs parsed by Fluent Bit, aggregated in Loki

Panel 6: **SageMaker Job Status**
- Active training jobs, completion rate, resource utilization
- Source: CloudWatch SageMaker metrics via CloudWatch Exporter

Panel 7: **S3 Access Patterns**
- PutObject/GetObject rates, access from unexpected sources (alert trigger)
- Source: S3 server access logs and CloudTrail S3 data events

### Alerting Rules

| Alert | Condition | Severity | Response |
|---|---|---|---|
| BEGRENSET data leak attempt | Sanitization service blocks outbound BEGRENSET data | Critical | Immediate investigation, sever hybrid link if confirmed |
| Direct Connect down | BGP session lost for > 30s | High | Failover procedures, notify operations |
| Unauthorized AWS API call | CloudTrail event from unexpected principal or region | Critical | Investigate, potentially revoke credentials |
| Exchange Zone firewall rule violation | Denied traffic from unexpected source/destination | High | Investigate source |
| S3 bucket policy change | AWS Config rule non-compliant | Critical | Auto-remediate via AWS Config remediation or alert |
| VPC route table change | Unexpected route added (especially toward BEGRENSET CIDR) | Critical | Immediate investigation, auto-revert |
| SageMaker training in wrong VPC | SageMaker ENI appears outside Isolated VPC | Critical | Kill job, investigate |
| Certificate expiry < 30d | cert-manager certificate approaching expiry | Warning | Rotate certificate |
| Hybrid link latency > 50ms | Prometheus latency metric exceeds threshold | Warning | Investigate Direct Connect performance |

---

## Infrastructure as Code

### Tools
- **OpenTofu** for AWS infrastructure (VPCs, Direct Connect, IAM, S3, SageMaker configuration)
- **OpenTofu** or **Ansible** for on-prem Exchange Zone infrastructure
- **ArgoCD** for Kubernetes GitOps (BEGRENSET zone cluster)
- **Ansible** for on-prem appliance configuration (firewalls, VPN appliances, switches)

### Repository Structure
```
infra/
  aws/
    tofu/
      environments/
        prod/
          hybrid-vpc/
          isolated-vpc/
          iam/
          direct-connect/
          monitoring/
  on-prem/
    ansible/
      playbooks/
        exchange-zone-firewall.yml
        vpn-appliance.yml
        monitoring-stack.yml
      inventory/
        netbox.yml          # Dynamic inventory from NetBox
    tofu/
      exchange-zone/
  kubernetes/
    argocd/
      apps/
        exchange-zone-api-gw/
        sanitization-service/
        monitoring/
        cilium-policies/
```

### Key OpenTofu Controls (AWS)

```hcl
# Enforce eu-north-1 only
provider "aws" {
  region              = "eu-north-1"
  allowed_account_ids = [var.workload_account_id]
}

# S3 bucket with encryption and access restrictions
resource "aws_s3_bucket" "training_data" {
  bucket = "defense-ml-training-data"
}

resource "aws_s3_bucket_policy" "restrict_to_vpce" {
  bucket = aws_s3_bucket.training_data.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "RestrictToVPCEndpoint"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = ["${aws_s3_bucket.training_data.arn}/*"]
      Condition = {
        StringNotEquals = {
          "aws:sourceVpce" = aws_vpc_endpoint.s3.id
        }
      }
    }]
  })
}

# No public access
resource "aws_s3_bucket_public_access_block" "training_data" {
  bucket                  = aws_s3_bucket.training_data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## Compliance Mapping

### Sikkerhetsloven Requirements

| Requirement | Implementation |
|---|---|
| Data sovereignty (BEGRENSET in Norway) | BEGRENSET zone is entirely on-prem in Norway; Exchange Zone prevents classified data egress; DLP rules enforce |
| Access control and need-to-know | FreeIPA/Keycloak RBAC, Kubernetes RBAC, AWS IAM least-privilege, all mapped to personnel clearances |
| Audit and accountability | Comprehensive logging in Wazuh/Loki; every cross-boundary action logged with user attribution |
| Cryptographic protection | IPsec (AES-256-GCM) on Direct Connect; TLS 1.3 for all API traffic; encryption at rest on both sides |
| Incident reporting | Wazuh SIEM with automated alerting; NorCERT reporting procedures documented |

### NSM Grunnprinsipper Mapping

| Principle | Category | Implementation |
|---|---|---|
| Asset inventory | Identifisere | NetBox (on-prem), AWS Config (cloud), automated discovery |
| Risk assessment | Identifisere | Hybrid boundary threat model documented; reviewed quarterly |
| Access control | Beskytte | Zero-trust: mTLS, identity-based Cilium policies, AWS IAM conditions |
| Network segmentation | Beskytte | Four distinct zones with firewall boundaries; BGP prefix isolation |
| Secure configuration | Beskytte | CIS Benchmarks (supplementary to NSM), automated with OpenSCAP |
| Patch management | Beskytte | Automated with Ansible (on-prem), AWS Systems Manager (cloud) |
| Continuous monitoring | Oppdage | Prometheus/Thanos/Grafana federation; Wazuh SIEM; VPC Flow Logs |
| Anomaly detection | Oppdage | GuardDuty (AWS), Falco/Tetragon (K8s), Suricata (network) |
| Security event logging | Oppdage | All zones log to centralized on-prem Loki/Wazuh; immutable CloudTrail |
| Incident response | Handtere | Playbooks covering hybrid scenarios; ability to isolate AWS in minutes |

---

## Disaster Recovery and Link Failure

### Direct Connect Failure
- **Primary**: AWS Direct Connect dedicated connection (eu-north-1)
- **Backup**: Site-to-site IPsec VPN over internet as failover (lower bandwidth, higher latency)
- **Failover**: BGP automatically reroutes via VPN tunnel when Direct Connect BGP session drops
- **On-prem continuity**: All BEGRENSET workloads continue operating independently; only AWS services (S3, SageMaker) become unavailable
- **RTO for hybrid link**: < 5 minutes (BGP convergence)

### AWS Region Failure
- S3 data is replicated to a second bucket in the same region (S3 replication within eu-north-1 AZs)
- SageMaker training jobs are idempotent and can be re-submitted
- No BEGRENSET data is at risk; on-prem operations are unaffected

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- Provision Exchange Zone network segment on-prem (separate VLAN/physical segment)
- Deploy and harden firewall appliance between BEGRENSET zone and Exchange Zone
- Configure Direct Connect with IPsec VPN overlay
- Provision AWS account structure with SCPs
- Deploy Hybrid VPC with VPC Endpoints

### Phase 2: Data Path (Weeks 5-8)
- Deploy API Gateway (Kong/APISIX) in Exchange Zone
- Build and deploy data sanitization service
- Configure S3 buckets with encryption and access policies
- Establish mTLS certificate chain
- Test data flow: on-prem to S3 with classification enforcement

### Phase 3: ML Workloads (Weeks 9-12)
- Configure SageMaker in Isolated VPC
- Build training data pipeline (on-prem sanitization, S3 upload, SageMaker job)
- Test SageMaker callback to on-prem APIs via Exchange Zone
- Validate no BEGRENSET data reaches AWS (penetration testing of DLP rules)

### Phase 4: Monitoring (Weeks 10-14, overlapping)
- Deploy Prometheus/Thanos federation
- Configure CloudWatch Exporter and Fluent Bit forwarder in Hybrid VPC
- Build Grafana dashboards (traffic flows, security events, cost)
- Configure Wazuh rules for hybrid boundary alerts
- Deploy Falco/Tetragon on Kubernetes
- Integration test: simulate BEGRENSET leak attempt, verify detection and alerting

### Phase 5: Accreditation and Hardening (Weeks 13-16)
- NSM security audit preparation
- Automated compliance scanning (OpenSCAP, Prowler for AWS, kube-bench for K8s)
- Penetration testing of hybrid boundary
- Documentation: architecture decision records, runbooks, incident response playbooks
- Personnel training
- NSM review and approval

---

## Key Architectural Decisions (ADRs)

### ADR-001: Exchange Zone as mandatory intermediary
**Decision**: All traffic between BEGRENSET zone and AWS must transit the on-prem Exchange Zone. No direct connectivity.
**Rationale**: Sikkerhetsloven requires that classified data remain under Norwegian control. The Exchange Zone provides content inspection, classification enforcement, and audit logging at the boundary.

### ADR-002: IPsec over Direct Connect
**Decision**: Encrypt all Direct Connect traffic with IPsec even though Direct Connect is a private link.
**Rationale**: Direct Connect transits infrastructure in Stockholm (outside Norway). For BEGRENSET-adjacent connectivity, encryption in transit is mandatory per NSM guidance. Belt-and-suspenders approach.

### ADR-003: No BEGRENSET data in AWS
**Decision**: AWS only handles UGRADERT (non-classified) data. BEGRENSET data is processed and stored exclusively on-prem.
**Rationale**: AWS eu-north-1 is not accredited for Norwegian BEGRENSET data. Even with encryption, storing BEGRENSET data outside Norway violates Sikkerhetsloven data sovereignty requirements.

### ADR-004: Metrics flow to on-prem, not the reverse
**Decision**: AWS metrics are forwarded to on-prem Thanos/Grafana. On-prem metrics are never sent to AWS.
**Rationale**: On-prem infrastructure metadata (hostnames, IPs, service topology) could reveal BEGRENSET operational details. Monitoring data flows inward only.

### ADR-005: FLOSS-first for on-prem components
**Decision**: Use FLOSS tools (Prometheus, Grafana, Loki, Cilium, Kong, Wazuh, Falco, OpenTofu) for on-prem infrastructure.
**Rationale**: Reduces licensing costs, avoids vendor lock-in, enables source code audit for security-sensitive components. NSM-approved appliances used where mandated (VPN, potentially firewall).

### ADR-006: Separate AWS accounts for hybrid and workload
**Decision**: Direct Connect terminates in a dedicated Hybrid-Connected account. SageMaker and S3 run in a separate ML-Workload account.
**Rationale**: Blast radius containment. Compromise of the ML workload account does not grant Direct Connect access. AWS Organizations SCPs enforce guardrails on both.
