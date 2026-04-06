# Hybrid Cloud Architecture: Nutanix On-Premises with AWS Cloud Bursting for HIPAA-Compliant Healthcare E-Commerce

## Document Information

| Field | Value |
|---|---|
| Author | Private Cloud Architect |
| Date | 2026-03-20 |
| Version | 1.0 |
| Classification | Internal - Confidential |
| Review Cycle | Quarterly |

---

## 1. Executive Summary

This document defines a hybrid cloud architecture that extends an existing 80-node Nutanix HCI environment to AWS for seasonal cloud bursting. The organization is a healthcare e-commerce company experiencing 10x traffic spikes during a 6-week Black Friday / holiday window. The current approach of permanently over-provisioning on-premises hardware for peak capacity is financially wasteful -- roughly 10 months of idle compute per year.

The proposed architecture retains Nutanix as the steady-state platform (where TCO is favorable and HIPAA controls are fully under organizational control) and bursts stateless and semi-stateful workloads to AWS during peak periods. Kubernetes serves as the workload abstraction layer across both environments, Ansible provides configuration management and orchestration, and a dedicated hybrid networking layer ensures HIPAA compliance end-to-end.

**Key outcomes:**
- Eliminate 60-70% of over-provisioned on-premises hardware (estimated 30-40 nodes worth of idle capacity)
- Maintain full HIPAA compliance across both environments with a single governance model
- Achieve burst capacity within 15 minutes of triggering scale-out
- Reduce annualized infrastructure TCO by an estimated 35-45%
- Establish a Kubernetes-based workload portability layer that prevents lock-in to either environment

---

## 2. Current State Assessment

### 2.1 On-Premises Infrastructure

**Compute:**
- ~80 Nutanix nodes (mixed NX-series and Dell XC)
- Nutanix AHV hypervisor with Prism Central management
- Estimated steady-state utilization: 25-35% (compute), scaling to 80-95% during peak
- Peak period: approximately 6 weeks (late November through early January)
- Remaining 46 weeks: significant over-provisioning

**Workload Profile (Inferred for Healthcare E-Commerce):**
- Web frontends (product catalog, search, checkout)
- API gateway and microservices tier
- Order processing and fulfillment engines
- Payment processing (PCI-DSS scope, likely overlaps with HIPAA)
- Patient/customer data management (PHI -- Protected Health Information)
- Inventory and warehouse management integrations
- Analytics and reporting
- Email/notification services

**Automation & Tooling:**
- Ansible is the primary automation tool (mature adoption)
- Kubernetes adoption is in early stages
- Likely existing CI/CD pipelines (to be integrated)

### 2.2 Traffic and Capacity Profile

| Metric | Steady State (46 weeks) | Peak (6 weeks) | Ratio |
|---|---|---|---|
| Traffic volume | Baseline | 10x baseline | 10:1 |
| Required compute | ~25-30 nodes equivalent | ~80+ nodes (current full capacity) | ~3:1 hardware |
| Utilization efficiency | 25-35% | 80-95% | -- |
| Cost of idle hardware | High (10 months) | N/A | -- |

### 2.3 Compliance Requirements

- **HIPAA** (Health Insurance Portability and Accountability Act) -- non-negotiable
  - PHI must be encrypted at rest and in transit
  - Access controls and audit logging for all PHI access
  - Business Associate Agreements (BAAs) required with all vendors handling PHI
  - Breach notification procedures
  - Minimum necessary access principle
  - Risk assessments and security management process
- **PCI-DSS** likely applies (e-commerce with payment processing)
- **State-level health data privacy laws** may apply depending on customer geography

---

## 3. Architecture Design

### 3.1 High-Level Architecture

```
                          +-----------------------+
                          |   Global Load         |
                          |   Balancer / DNS      |
                          |   (Route 53 +         |
                          |    weighted routing)   |
                          +----------+------------+
                                     |
                    +----------------+----------------+
                    |                                 |
          +---------v----------+          +-----------v---------+
          |  ON-PREMISES       |          |  AWS (Burst)        |
          |  NUTANIX CLUSTER   |          |  HIPAA-Compliant    |
          |                    |          |                     |
          | +----------------+ |          | +----------------+  |
          | | Kubernetes     | |  <-----> | | EKS Cluster    |  |
          | | (NKE or RKE2)  | |  Hybrid  | | (HIPAA-ready)  |  |
          | +----------------+ |  Network | +----------------+  |
          | | Web | API | Svc| |          | | Web | API | Svc|  |
          | +----------------+ |          | +----------------+  |
          |                    |          |                     |
          | +----------------+ |          | +----------------+  |
          | | Stateful Data  | |  Sync    | | Caches / Read  |  |
          | | (Primary DBs,  | | -------> | | Replicas /     |  |
          | |  PHI Store)    | |          | | Session Store  |  |
          | +----------------+ |          | +----------------+  |
          +--------------------+          +---------------------+
                    |                                 |
                    +------------ AWS Direct ----------+
                              Connect / VPN
```

### 3.2 Design Principles

1. **Data gravity on-premises**: PHI and primary databases remain on Nutanix. AWS burst nodes access data via secure APIs or read replicas -- PHI never resides uncontrolled in AWS.
2. **Stateless workloads burst first**: Web frontends, API gateways, product catalog reads, search, and static asset serving are ideal burst candidates.
3. **Kubernetes as the abstraction layer**: Identical container images deploy to both on-premises Kubernetes and AWS EKS, eliminating environment-specific application changes.
4. **Compliance as code**: HIPAA controls are codified in Ansible playbooks, Kubernetes policies, and AWS CloudFormation/Terraform -- not manual checklists.
5. **Automated scale-out and scale-in**: Burst activation is automated based on traffic thresholds, with human approval gates for initial activation each season.
6. **Encrypt everything**: TLS 1.3 for all transit, AES-256 for all storage, no exceptions in either environment.

### 3.3 Workload Placement Strategy

| Workload | Steady-State Location | Peak Burst Location | Rationale |
|---|---|---|---|
| Product catalog / search | On-prem K8s | On-prem + AWS EKS | Stateless, read-heavy, ideal for bursting |
| Web frontend / CDN | On-prem K8s | On-prem + AWS EKS + CloudFront | Static assets via CloudFront, dynamic via EKS |
| API gateway | On-prem K8s | On-prem + AWS EKS | Stateless, routes to backend services |
| Checkout / cart service | On-prem K8s | On-prem + AWS EKS | Session state in Redis (replicated) |
| Payment processing | On-prem K8s | On-prem ONLY | PCI-DSS scope minimization; keep on-prem |
| Order processing | On-prem K8s | On-prem + AWS EKS (non-PHI parts) | Split: PHI operations on-prem, notifications/fulfillment can burst |
| Primary databases (PostgreSQL/MySQL) | On-prem Nutanix VMs | On-prem ONLY (read replicas in AWS) | Data gravity, PHI residency, latency |
| PHI data store | On-prem Nutanix | On-prem ONLY | HIPAA -- full organizational control |
| Redis / session cache | On-prem K8s | On-prem + AWS ElastiCache | Replicated for burst; no PHI in session |
| Elasticsearch / search index | On-prem K8s | On-prem + AWS OpenSearch | Read replica / index copy for burst traffic |
| Email / notification | On-prem | On-prem + AWS SES | Stateless, easily burst |
| Analytics / reporting | On-prem | On-prem ONLY | Accesses PHI, keep on-prem |
| CI/CD pipelines | On-prem | On-prem | No reason to burst |
| Monitoring / observability | On-prem (primary) | On-prem + AWS (local collectors) | Federated Prometheus model |

---

## 4. On-Premises Architecture (Nutanix)

### 4.1 Nutanix Cluster Right-Sizing

With burst capacity moving to AWS, the on-premises cluster can be right-sized for steady-state plus a reasonable buffer (not 10x peak).

**Proposed target sizing:**
- Retain ~45-55 nodes (reduce from 80 by not replacing aging hardware during refresh cycles)
- Steady-state compute: 30-35 nodes for production workloads
- Buffer: 5-10 nodes for failover, maintenance windows, and minor traffic spikes
- Kubernetes nodes: 8-12 nodes dedicated to on-prem K8s cluster
- Management / infrastructure: 3-5 nodes (Prism, monitoring, Ansible, CI/CD)

**Hardware refresh strategy:**
- Do NOT renew or replace 25-35 nodes that serve only peak capacity
- Redirect capital savings toward AWS burst costs and Kubernetes platform investment
- Standardize remaining fleet on a single model during next refresh (reduce mixed NX/Dell XC complexity)

### 4.2 Kubernetes On-Premises

**Platform selection: RKE2 (recommended) or Nutanix Kubernetes Engine (NKE)**

RKE2 is recommended over NKE for this hybrid architecture because:
- FIPS 140-2 validated crypto modules (relevant for HIPAA)
- CIS-hardened by default
- Identical Kubernetes API surface as EKS (standard upstream K8s)
- Rancher integration for multi-cluster management across on-prem and EKS
- No additional Nutanix licensing cost
- Strong Ansible integration via kubernetes.core collection

If NKE is already in use or preferred for operational simplicity, it is a viable alternative -- the key requirement is standard Kubernetes API compatibility with EKS.

**On-prem K8s cluster design:**

```
Control Plane (3 nodes - HA):
  - Dedicated Nutanix VMs (4 vCPU, 16GB RAM each)
  - etcd on NVMe-backed Nutanix Volumes
  - kube-vip for control plane VIP

Worker Nodes (8-12 nodes):
  - Nutanix VMs or bare-metal (via AHV passthrough)
  - 16-32 vCPU, 64-128GB RAM each (size based on workload density)
  - Nutanix CSI driver for persistent volumes
  - Node labels: tier=frontend, tier=backend, tier=data

Ingress:
  - NGINX Ingress Controller or Cilium Gateway API
  - MetalLB for bare-metal load balancer IPs (if not using Nutanix LB)

Networking:
  - Cilium CNI (eBPF-based, supports network policies for HIPAA segmentation)
  - Cilium cluster mesh for future multi-cluster on-prem expansion

Storage:
  - Nutanix CSI driver for persistent volumes (backed by Nutanix Storage)
  - StorageClasses: nutanix-ssd (NVMe/SSD tier), nutanix-standard (hybrid tier)
```

### 4.3 Stateful Services On-Premises

**Databases:**
- Primary PostgreSQL/MySQL on Nutanix VMs (not containerized for now)
- Managed via Ansible (provisioning, configuration, patching, backups)
- Nutanix data protection for VM-level snapshots
- Streaming replication to read replicas (on-prem and AWS)

**PHI Data Store:**
- Remains exclusively on Nutanix
- Nutanix encryption at rest enabled (AES-256, Nutanix-managed or external KMS)
- Nutanix Flow microsegmentation policies restricting access to PHI VLANs
- Audit logging for all data access

**Redis (Session / Cache):**
- Redis Sentinel or Redis Cluster on-premises on K8s
- Session data design: no PHI in session tokens (use opaque references)
- Replicated to AWS ElastiCache during burst periods

---

## 5. AWS Burst Architecture

### 5.1 AWS Account Structure

```
AWS Organization
+-- Management Account (billing, organization policies)
+-- Security Account (GuardDuty, Security Hub, CloudTrail aggregation)
+-- Shared Services Account (Transit Gateway, Direct Connect termination, ECR)
+-- Production Burst Account (EKS, ALB, ElastiCache, etc.)
+-- Non-Prod Account (testing burst architecture, DR drills)
```

All accounts operate under AWS BAA (Business Associate Agreement) for HIPAA. Only HIPAA-eligible services are used.

### 5.2 AWS HIPAA Compliance Foundation

**Pre-requisites (one-time setup):**

1. **AWS BAA**: Execute a Business Associate Agreement with AWS (available through AWS Artifact)
2. **HIPAA-eligible services only**: Restrict usage to AWS HIPAA-eligible services via Service Control Policies (SCPs)
3. **AWS GovCloud evaluation**: For maximum HIPAA assurance, evaluate whether GovCloud is warranted. For most healthcare e-commerce, standard commercial AWS regions with BAA are sufficient. GovCloud adds cost and operational complexity -- reserve it for scenarios involving government healthcare contracts.

**Service Control Policies (SCPs):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHIPAAServices",
      "Effect": "Deny",
      "Action": [
        "lightsail:*",
        "gamelift:*",
        "other-non-hipaa-services:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyUnencryptedStorage",
      "Effect": "Deny",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "EnforceRegionRestriction",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "sts:*",
        "organizations:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

**AWS Config Rules for HIPAA:**
- Enable AWS Config with HIPAA conformance pack
- Enforce encryption on all EBS volumes, S3 buckets, RDS instances
- Require VPC flow logs on all VPCs
- Enforce CloudTrail logging in all regions
- Detect and alert on any public S3 buckets or security groups

**CloudTrail:**
- Organization-wide CloudTrail with log file validation
- Logs shipped to S3 with KMS encryption and sent to on-prem SIEM
- 7-year retention (HIPAA requires 6 years)

### 5.3 AWS Network Architecture

```
+------------------------------------------------------------------+
|  AWS VPC (Production Burst)                                       |
|  CIDR: 10.100.0.0/16                                             |
|                                                                   |
|  +--------------------+  +--------------------+                   |
|  | Private Subnet AZ-a|  | Private Subnet AZ-b|                  |
|  | 10.100.1.0/24      |  | 10.100.2.0/24      |                  |
|  | EKS Worker Nodes   |  | EKS Worker Nodes   |                  |
|  +--------------------+  +--------------------+                   |
|                                                                   |
|  +--------------------+  +--------------------+                   |
|  | Private Subnet AZ-a|  | Private Subnet AZ-b|                  |
|  | 10.100.10.0/24     |  | 10.100.11.0/24     |                  |
|  | ElastiCache, Data  |  | ElastiCache, Data  |                  |
|  +--------------------+  +--------------------+                   |
|                                                                   |
|  +--------------------+  +--------------------+                   |
|  | Public Subnet AZ-a |  | Public Subnet AZ-b |                  |
|  | 10.100.100.0/24    |  | 10.100.101.0/24    |                  |
|  | ALB, NAT Gateway   |  | ALB, NAT Gateway   |                  |
|  +--------------------+  +--------------------+                   |
|                                                                   |
|  Transit Gateway Attachment --> TGW --> Direct Connect Gateway     |
+------------------------------------------------------------------+
          |
          | AWS Direct Connect (dedicated 10Gbps)
          | + Site-to-Site VPN (backup/failover)
          |
+------------------------------------------------------------------+
|  On-Premises Data Center                                          |
|  CIDR: 10.0.0.0/16 (example)                                     |
|  Nutanix Cluster + Kubernetes                                     |
+------------------------------------------------------------------+
```

**Hybrid connectivity:**

| Connection | Type | Bandwidth | Purpose | SLA |
|---|---|---|---|---|
| Primary | AWS Direct Connect (dedicated) | 10 Gbps | Production traffic, DB replication, inter-cluster | 99.99% (redundant) |
| Secondary | AWS Direct Connect (second circuit, different location) | 10 Gbps | Redundancy | 99.99% |
| Tertiary | Site-to-Site VPN over internet | Up to 1.25 Gbps per tunnel | Failover if both DX circuits fail | Best-effort |

**Direct Connect setup:**
- Two Direct Connect connections at two different DX locations for resilience
- Private VIF to Transit Gateway (enables future multi-VPC expansion)
- BGP routing with MED/local-preference for primary/secondary failover
- BFD (Bidirectional Forwarding Detection) for fast failover (~1 second)
- MACsec encryption on Direct Connect for HIPAA transit encryption (Layer 2)
- IPsec VPN overlay on Direct Connect for defense-in-depth (optional but recommended for HIPAA)

**DNS architecture:**
- Route 53 for public-facing DNS
- Route 53 Private Hosted Zones for AWS internal DNS
- On-prem DNS (e.g., PowerDNS, Infoblox, or BIND) for on-prem resolution
- Route 53 Resolver endpoints for hybrid DNS resolution:
  - Inbound endpoint: AWS resources resolve on-prem DNS names
  - Outbound endpoint: On-prem resources resolve AWS private DNS names
- Split-horizon DNS for services accessible from both environments

### 5.4 AWS EKS Cluster Design

**EKS cluster configuration:**

```yaml
# Cluster specification (conceptual)
cluster:
  name: healthcare-ecom-burst
  version: "1.29"  # or latest stable
  region: us-east-1
  encryption:
    secrets: true
    kms_key: arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID
  logging:
    api: true
    audit: true
    authenticator: true
    controllerManager: true
    scheduler: true
  networking:
    vpc_cni: true
    service_cidr: 172.20.0.0/16
    pod_cidr: # managed by VPC CNI (pods get VPC IPs)
  access:
    endpoint_private: true
    endpoint_public: false  # private-only API endpoint
    # Access via Direct Connect or VPN only
```

**Node groups:**

| Node Group | Instance Type | Min | Max (Peak) | Purpose |
|---|---|---|---|---|
| system | m6i.xlarge | 2 | 3 | CoreDNS, ingress controllers, monitoring agents |
| web-frontend | c6i.2xlarge | 0 | 40 | Product pages, search UI, static rendering |
| api-backend | c6i.4xlarge | 0 | 30 | API services, business logic |
| cache-client | r6i.2xlarge | 0 | 10 | Services requiring high memory for local caching |

- All node groups use **Cluster Autoscaler** or **Karpenter** for automatic scaling
- During off-peak (46 weeks): min=0 for burst node groups, only system nodes run
- During peak: autoscaler provisions nodes based on pending pod requests
- **Spot instances** for stateless web/API tiers (50-70% cost savings):
  - Use diversified allocation strategy across instance families (c6i, c6a, c5, m6i)
  - On-Demand for system nodes and any pods requiring guaranteed availability
  - Spot for web-frontend and api-backend (applications must handle graceful termination)

**EKS add-ons and components:**
- AWS Load Balancer Controller (ALB for ingress)
- External DNS (sync Route 53 with K8s ingress/services)
- Cluster Autoscaler or Karpenter
- AWS Secrets Manager CSI driver (for secrets injection)
- Fluent Bit (log shipping to CloudWatch and on-prem)
- AWS Distro for OpenTelemetry (ADOT) collector
- Gatekeeper / Kyverno for policy enforcement (matching on-prem policies)

### 5.5 AWS Data Tier (Burst Period)

**Read replicas (provisioned during burst activation):**

| Service | AWS Implementation | Purpose | PHI Exposure |
|---|---|---|---|
| PostgreSQL read replica | RDS PostgreSQL (encrypted, private subnet) | Offload read queries from on-prem primary | Minimal -- product catalog data, no direct PHI |
| Redis cache | ElastiCache Redis (encryption at rest + transit) | Session store, page caching, cart data | No PHI in cache (enforced by application design) |
| Search index | OpenSearch Service (encrypted, VPC-only) | Product search for burst traffic | No PHI -- product data only |
| Static assets | S3 + CloudFront | Images, CSS, JS | No PHI |

**Critical design constraint**: PHI data (patient information, health records, prescription data) is **never replicated to AWS**. The application architecture must separate PHI-accessing operations from non-PHI operations. API calls requiring PHI are routed back to on-premises services over Direct Connect.

**Data flow for PHI-adjacent operations during burst:**

```
Customer browser --> AWS ALB --> EKS (product browsing, cart) --> AWS ElastiCache/RDS
                                                                   (no PHI)

Customer browser --> AWS ALB --> EKS (checkout w/ health info) --> Direct Connect
                                --> On-Prem K8s (PHI service) --> On-Prem DB
                                <-- Response back through same path
```

---

## 6. Kubernetes Workload Portability

### 6.1 Container Image Strategy

**Single image registry with mirroring:**

```
On-Prem Harbor Registry (source of truth)
    |
    +--> AWS ECR (mirror, synced via CI/CD pipeline)
    |
    +--> On-Prem K8s nodes (pull from Harbor)
    +--> AWS EKS nodes (pull from ECR)
```

- Harbor on-premises as the primary registry (image signing with cosign/Sigstore)
- AWS ECR as a mirror for burst workloads
- CI/CD pipeline pushes to Harbor, then replicates to ECR
- Image scanning in both registries (Harbor Trivy scanner + ECR scanning)
- Images are identical -- no environment-specific builds

### 6.2 Kubernetes Manifests and Helm Charts

**Environment abstraction via Helm values:**

```yaml
# values-onprem.yaml
global:
  environment: onprem
  registry: harbor.internal.company.com
ingress:
  class: nginx
  annotations:
    # on-prem specific annotations
storage:
  class: nutanix-ssd
database:
  host: postgres-primary.internal.company.com
  port: 5432
redis:
  host: redis-cluster.internal.company.com
  port: 6379

# values-aws-burst.yaml
global:
  environment: aws-burst
  registry: ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
ingress:
  class: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
storage:
  class: gp3
database:
  host: postgres-read-replica.XXXXX.us-east-1.rds.amazonaws.com
  port: 5432
redis:
  host: redis-burst.XXXXX.cache.amazonaws.com
  port: 6379
```

**GitOps with ArgoCD:**

```
ArgoCD (on-prem, primary) manages:
  +-- On-prem K8s cluster (ApplicationSet)
  +-- AWS EKS cluster (ApplicationSet, activated during burst)

Repository structure:
  apps/
    product-catalog/
      base/          # Kustomize base (shared)
      overlays/
        onprem/      # On-prem kustomization
        aws-burst/   # AWS burst kustomization
    api-gateway/
      base/
      overlays/
        onprem/
        aws-burst/
    ...
  infrastructure/
    onprem/
    aws/
```

- ArgoCD runs on-premises (single management plane)
- EKS cluster registered as a remote cluster in ArgoCD
- Burst applications deployed via ApplicationSet generator with cluster selector
- During off-peak: AWS ApplicationSet disabled, no workloads running on EKS

### 6.3 Kubernetes Network Policies (HIPAA Segmentation)

```yaml
# Example: Deny all ingress by default (both environments)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: healthcare-ecom
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow frontend to talk to API only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: healthcare-ecom
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
          protocol: TCP

---
# In AWS: Allow API pods to reach on-prem PHI services via Direct Connect
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-onprem-phi
  namespace: healthcare-ecom
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.50.0/24  # On-prem PHI service VLAN
      ports:
        - port: 443
          protocol: TCP
```

### 6.4 Multi-Cluster Management with Rancher

**Rancher** (on-premises) provides a unified management plane:

- Single pane of glass for both on-prem RKE2 and AWS EKS
- Centralized RBAC with LDAP/AD/SAML federation
- Cluster monitoring and alerting (Rancher Monitoring based on Prometheus/Grafana)
- Cluster-level CIS benchmark scanning
- Backup/restore via Rancher Backup operator

---

## 7. HIPAA Compliance Architecture

### 7.1 HIPAA Technical Safeguards Matrix

| Safeguard | On-Premises Implementation | AWS Implementation |
|---|---|---|
| **Access Control** | Nutanix Prism RBAC, K8s RBAC, FreeIPA/AD | AWS IAM, EKS RBAC, IAM Roles for Service Accounts (IRSA) |
| **Audit Controls** | Nutanix audit logs, K8s audit logging, Wazuh SIEM | CloudTrail, VPC Flow Logs, EKS audit logs, shipped to on-prem SIEM |
| **Integrity Controls** | Nutanix data protection checksums, container image signing (cosign) | S3 object lock, ECR image scanning, KMS-encrypted EBS |
| **Transmission Security** | TLS 1.3 everywhere, Cilium encrypted overlay, IPsec for DC interconnects | TLS 1.3, MACsec on Direct Connect, ALB with ACM certificates, VPC encryption |
| **Encryption at Rest** | Nutanix software encryption (AES-256), LUKS for K8s node disks | KMS-managed keys, EBS encryption, RDS encryption, ElastiCache encryption |
| **PHI Access Logging** | Application-level audit logs, database query logging | N/A (PHI does not reside in AWS) |
| **Automatic Logoff** | Session timeout policies in application, K8s RBAC token expiry | Same application policies, IAM session duration limits |
| **Emergency Access** | Break-glass procedures documented, Ansible vault for emergency creds | IAM break-glass role with CloudTrail alerting |

### 7.2 Business Associate Agreement (BAA) Chain

```
Your Organization (Covered Entity)
    |
    +-- AWS (Business Associate -- BAA via AWS Artifact)
    |     +-- S3, EKS, RDS, ElastiCache, CloudWatch, etc.
    |
    +-- Nutanix (Business Associate -- BAA required if Nutanix support accesses PHI)
    |
    +-- Direct Connect Provider (Business Associate -- if they can access data)
    |
    +-- Any SaaS tools that touch PHI (monitoring, logging, etc.)
```

**Action items:**
- Execute AWS BAA through AWS Artifact (no-cost, self-service)
- Review existing Nutanix BAA status
- Ensure Direct Connect / colocation provider BAA is in place
- Audit all third-party tools for PHI exposure

### 7.3 HIPAA Risk Assessment Integration

The hybrid architecture introduces specific risks that must be assessed:

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| PHI accidentally deployed to AWS | Medium | High | Application architecture separates PHI services; K8s admission controller blocks PHI-labeled deployments on AWS cluster; network policies prevent AWS pods from reaching PHI databases |
| Direct Connect failure during peak | Low | High | Redundant DX circuits + VPN failover; AWS workloads degrade gracefully (non-PHI operations continue, PHI operations queue) |
| Unauthorized access to AWS resources | Medium | High | SCPs, IAM least-privilege, private EKS endpoint, no public subnets for workloads, VPC flow log monitoring |
| Encryption key compromise | Low | Critical | Separate KMS keys per environment, automatic key rotation, HSM-backed keys for PHI encryption on-prem |
| Audit log tampering | Low | High | CloudTrail log file validation, S3 object lock, on-prem SIEM ingestion with integrity checking |
| Burst infrastructure not ready for peak | Medium | High | Pre-season readiness testing (see Section 10), automated infrastructure provisioning |

### 7.4 PHI Data Boundary Enforcement

**Kubernetes Admission Controller (OPA Gatekeeper / Kyverno):**

```yaml
# Kyverno policy: Block deployments with PHI label on AWS cluster
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-phi-on-aws
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-phi-workloads
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
                - Job
      preconditions:
        all:
          - key: "{{ request.object.metadata.labels.\"data-classification\" || '' }}"
            operator: Equals
            value: "phi"
      validate:
        message: "PHI workloads are not permitted on the AWS burst cluster. Deploy to on-premises only."
        deny: {}
```

This policy is deployed **only** on the AWS EKS cluster. Any deployment labeled `data-classification: phi` is rejected.

---

## 8. Ansible Automation

### 8.1 Ansible Automation Scope

Since the team is already heavily invested in Ansible, it serves as the primary automation layer for infrastructure provisioning and configuration management. Kubernetes workload deployment is handled by ArgoCD (GitOps), while Ansible handles everything underneath.

**Ansible role structure:**

```
ansible/
  inventory/
    onprem/
      hosts.yml                # Nutanix VMs, K8s nodes, infra
      group_vars/
        all.yml
        nutanix.yml
        kubernetes.yml
    aws/
      aws_ec2.yml              # Dynamic inventory plugin
      group_vars/
        all.yml
        eks.yml

  playbooks/
    # On-premises
    nutanix-cluster-config.yml
    k8s-cluster-bootstrap.yml
    k8s-cluster-addons.yml
    database-provision.yml
    monitoring-deploy.yml
    security-hardening.yml

    # AWS burst
    aws-burst-activate.yml      # Full burst environment provisioning
    aws-burst-deactivate.yml    # Tear down burst environment
    aws-network-setup.yml       # VPC, subnets, TGW (one-time)
    aws-eks-provision.yml       # EKS cluster creation
    aws-data-tier-provision.yml # RDS replicas, ElastiCache, OpenSearch
    aws-dns-update.yml          # Route 53 weighted routing update

    # Hybrid
    hybrid-connectivity-test.yml
    hipaa-compliance-audit.yml
    dr-failover-test.yml
    burst-readiness-check.yml

  roles/
    common/                     # Shared across on-prem and AWS
    nutanix-vm/
    k8s-hardening/
    cilium/
    monitoring-agent/
    aws-eks/
    aws-elasticache/
    aws-rds-replica/
    hipaa-controls/

  collections/
    requirements.yml:
      - kubernetes.core
      - amazon.aws
      - community.general
      - nutanix.ncp            # Nutanix Ansible collection
```

### 8.2 Burst Activation Playbook (Key Workflow)

```yaml
# playbooks/aws-burst-activate.yml
---
- name: Activate AWS Burst Environment for Peak Season
  hosts: localhost
  gather_facts: false
  vars:
    burst_season: "{{ lookup('env', 'BURST_SEASON') | default('holiday-2026') }}"
    approval_ticket: "{{ lookup('env', 'APPROVAL_TICKET') }}"

  pre_tasks:
    - name: Verify approval ticket exists
      assert:
        that:
          - approval_ticket | length > 0
        fail_msg: "Burst activation requires an approved change ticket"

    - name: Run pre-burst connectivity checks
      include_role:
        name: hybrid-connectivity-check

    - name: Verify Direct Connect circuits are healthy
      include_role:
        name: aws-direct-connect-health

  tasks:
    - name: Provision EKS managed node groups
      include_role:
        name: aws-eks
        tasks_from: scale-up-node-groups
      vars:
        node_groups:
          web-frontend:
            min: 2
            desired: 5
            max: 40
          api-backend:
            min: 2
            desired: 4
            max: 30
          cache-client:
            min: 1
            desired: 2
            max: 10

    - name: Provision RDS read replica
      include_role:
        name: aws-rds-replica
      vars:
        source_db: postgres-primary
        replica_instance_class: db.r6i.2xlarge
        encrypted: true
        kms_key_id: "{{ aws_kms_key_arn }}"

    - name: Provision ElastiCache Redis cluster
      include_role:
        name: aws-elasticache
      vars:
        cluster_mode: enabled
        node_type: cache.r6g.xlarge
        num_node_groups: 3
        replicas_per_node_group: 2
        at_rest_encryption: true
        in_transit_encryption: true

    - name: Sync container images to ECR
      include_role:
        name: ecr-image-sync
      vars:
        source_registry: harbor.internal.company.com
        images: "{{ burst_application_images }}"

    - name: Deploy ArgoCD ApplicationSet for AWS burst
      kubernetes.core.k8s:
        kubeconfig: "{{ onprem_kubeconfig }}"
        state: present
        src: "{{ playbook_dir }}/../manifests/argocd/aws-burst-appset.yml"

    - name: Update Route 53 weighted routing
      include_role:
        name: aws-dns-update
      vars:
        routing_policy: weighted
        onprem_weight: 50
        aws_weight: 50
        # Gradually shift traffic -- start 50/50, adjust based on monitoring

    - name: Run HIPAA compliance validation on AWS environment
      include_role:
        name: hipaa-controls
        tasks_from: validate-aws

  post_tasks:
    - name: Send activation notification
      include_role:
        name: notification
      vars:
        message: "AWS burst environment activated for {{ burst_season }}. Ticket: {{ approval_ticket }}"
        channels: ["slack-ops", "email-oncall"]
```

### 8.3 Burst Deactivation Playbook

```yaml
# playbooks/aws-burst-deactivate.yml
---
- name: Deactivate AWS Burst Environment (Post-Peak)
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Shift all traffic to on-premises
      include_role:
        name: aws-dns-update
      vars:
        routing_policy: weighted
        onprem_weight: 100
        aws_weight: 0

    - name: Wait for in-flight requests to drain
      pause:
        minutes: 15

    - name: Verify no active sessions on AWS
      include_role:
        name: aws-session-drain-check

    - name: Remove ArgoCD AWS burst applications
      kubernetes.core.k8s:
        kubeconfig: "{{ onprem_kubeconfig }}"
        state: absent
        src: "{{ playbook_dir }}/../manifests/argocd/aws-burst-appset.yml"

    - name: Scale down EKS node groups to zero
      include_role:
        name: aws-eks
        tasks_from: scale-down-node-groups

    - name: Destroy RDS read replica
      include_role:
        name: aws-rds-replica
        tasks_from: destroy

    - name: Destroy ElastiCache cluster
      include_role:
        name: aws-elasticache
        tasks_from: destroy

    - name: Run cost report for burst period
      include_role:
        name: aws-cost-report

    - name: Send deactivation notification
      include_role:
        name: notification
      vars:
        message: "AWS burst environment deactivated. Cost report attached."
```

---

## 9. Networking and Traffic Management

### 9.1 Global Traffic Distribution

**During steady state (46 weeks):**
- All traffic routes to on-premises via existing DNS/load balancer
- AWS EKS has zero worker nodes (only EKS control plane persists, ~$0.10/hr)
- Direct Connect circuits remain active (used for monitoring, replication testing, DR)

**During burst (6 weeks):**

```
                    DNS (Route 53 Weighted Routing)
                    /                              \
                   /                                \
        On-Prem (weight: 40-60)          AWS (weight: 40-60)
              |                                |
        On-Prem LB                        AWS ALB
              |                                |
        On-Prem K8s                       AWS EKS
        (all services)                    (stateless services)
              |                                |
        On-Prem DB (primary)              AWS (read replicas)
              |                                |
              +--- Direct Connect Link --------+
              (PHI requests, DB replication, monitoring)
```

**Traffic routing strategy:**
1. Route 53 health checks on both endpoints
2. Weighted routing initially 50/50, adjustable via Ansible
3. Latency-based routing as alternative (users geographically closer to AWS region get routed there)
4. Automatic failover: if on-prem health check fails, all traffic shifts to AWS (degraded mode -- no PHI operations)

### 9.2 Latency Considerations

| Path | Expected Latency | Acceptable? |
|---|---|---|
| User --> AWS ALB --> EKS (non-PHI) | 10-50ms | Yes |
| User --> On-prem LB --> On-prem K8s | 10-50ms | Yes |
| AWS EKS --> Direct Connect --> On-prem (PHI API call) | 5-15ms (DX) | Yes, with caching |
| AWS EKS --> On-prem DB (read) | 5-15ms (DX) | Acceptable for read-after-write |

Direct Connect latency is typically 2-10ms depending on colocation proximity. This is acceptable for synchronous API calls but should be minimized through:
- Aggressive caching of non-PHI data on AWS side
- Asynchronous patterns for non-time-critical PHI operations
- Read replicas on AWS for non-PHI data queries

### 9.3 Firewall and Security Groups

**On-premises (Nutanix Flow or perimeter firewall):**
- Allow Direct Connect subnet to reach specific service ports only
- Deny all AWS-originated traffic to PHI database VLANs except from designated API gateway pods
- Allow monitoring traffic (Prometheus federation, log shipping)

**AWS Security Groups:**

```
ALB Security Group:
  Inbound: 443/tcp from 0.0.0.0/0 (HTTPS)
  Outbound: 8080/tcp to EKS Worker SG

EKS Worker Security Group:
  Inbound: 8080/tcp from ALB SG
  Inbound: All from EKS Worker SG (inter-pod)
  Inbound: 443/tcp from EKS Control Plane SG
  Outbound: 443/tcp to 0.0.0.0/0 (ECR, AWS APIs)
  Outbound: 5432/tcp to RDS SG (read replicas)
  Outbound: 6379/tcp to ElastiCache SG
  Outbound: All to 10.0.0.0/16 (on-prem via DX)

RDS Security Group:
  Inbound: 5432/tcp from EKS Worker SG only

ElastiCache Security Group:
  Inbound: 6379/tcp from EKS Worker SG only

No public IPs on any EKS worker node or data tier resource.
```

---

## 10. Observability and Monitoring

### 10.1 Unified Observability Architecture

```
+------------------+          +------------------+
| On-Prem K8s      |          | AWS EKS          |
|                  |          |                  |
| Prometheus       |          | Prometheus       |
| (local scrape)   |          | (local scrape)   |
|       |          |          |       |          |
| Fluent Bit ------|----------|-> Fluent Bit     |
|       |          |          |       |          |
| OTel Collector   |          | OTel Collector   |
+--------+---------+          +--------+---------+
         |                             |
         +----------+    +-------------+
                    |    |
              +-----v----v------+
              | On-Prem Central |
              | Monitoring      |
              |                 |
              | Thanos/Mimir    |  (long-term metrics storage)
              | Loki            |  (log aggregation)
              | Grafana         |  (dashboards)
              | Alertmanager    |  (alerting)
              +-----------------+
```

**Key metrics to monitor:**
- Burst cluster health (EKS node count, pod scheduling latency)
- Direct Connect circuit utilization and latency
- Cross-environment request latency (AWS pod --> on-prem service)
- PHI boundary violations (any unexpected traffic patterns)
- Cost accumulation in AWS (real-time cost tracking)
- Kubernetes pod autoscaler events (HPA/VPA scaling decisions)
- Application error rates by environment

### 10.2 Alerting Strategy

| Alert | Severity | Action |
|---|---|---|
| Direct Connect both circuits down | P1 - Critical | Verify VPN failover active; page on-call |
| AWS burst cost exceeding daily threshold | P2 - Warning | Review Karpenter provisioning, check for runaway scaling |
| PHI-classified pod scheduled on AWS | P1 - Critical | Immediate investigation -- admission controller should prevent this |
| Cross-environment latency > 50ms | P2 - Warning | Check DX circuit health, review traffic patterns |
| EKS node group at max capacity | P2 - Warning | Evaluate increasing max node count |
| HIPAA compliance check failure | P1 - Critical | Investigate immediately, potential breach reporting |
| Database replication lag > 30s | P2 - Warning | Check DX bandwidth, DB load |

---

## 11. Security Architecture

### 11.1 Identity and Access Management

**Federated identity model:**

```
On-Prem Identity Provider (AD/FreeIPA/Keycloak)
    |
    +-- LDAP/SAML --> Nutanix Prism (admin access)
    +-- OIDC --> On-prem Kubernetes (RBAC)
    +-- OIDC --> Rancher (multi-cluster management)
    +-- SAML --> AWS IAM Identity Center (SSO)
         |
         +-- IAM Roles --> EKS cluster access
         +-- IAM Roles --> AWS console access
         +-- IAM Roles --> Service accounts (IRSA)
```

- Single identity source for both environments
- No local AWS IAM users (all access via SSO federation)
- EKS uses IAM Roles for Service Accounts (IRSA) -- pods get scoped AWS permissions
- MFA required for all human access to both environments
- Break-glass procedures documented for emergency access

### 11.2 Secrets Management

**HashiCorp Vault (on-premises) as central secrets store:**

- Vault cluster on-premises (HA with Raft storage on Nutanix)
- Kubernetes auth method for both on-prem K8s and EKS
- AWS auth method for EC2/Lambda (if needed)
- Dynamic database credentials (PostgreSQL, Redis)
- PKI secrets engine for internal TLS certificates
- Transit secrets engine for application-level encryption
- Audit logging enabled, shipped to SIEM

**For teams preferring AWS-native:**
- AWS Secrets Manager for AWS-specific secrets (RDS passwords, API keys)
- External Secrets Operator on EKS to sync Vault secrets to K8s Secrets
- Never store PHI-related secrets in AWS -- keep in on-prem Vault

### 11.3 Certificate Management

- cert-manager on both Kubernetes clusters
- On-prem: ACME with Let's Encrypt or internal CA (step-ca, Vault PKI)
- AWS: ACM for ALB certificates, cert-manager with Vault PKI for pod-to-pod TLS
- Mutual TLS (mTLS) between environments for service-to-service calls over Direct Connect
- Certificate rotation automated, no manual certificate management

### 11.4 Container Security

- Image scanning in CI pipeline (Trivy, Grype)
- Image signing with cosign (Sigstore)
- Admission controller verifies image signatures before deployment
- Pod Security Standards enforced (restricted profile)
- No privileged containers
- Read-only root filesystems where possible
- Runtime security monitoring with Falco on both clusters
- Regular CIS Kubernetes benchmark scans

---

## 12. Cost Analysis

### 12.1 Current State Cost (Estimated Annual)

| Item | Annual Cost (Estimated) |
|---|---|
| 80 Nutanix nodes (hardware amortized over 5yr) | $800,000 - $1,200,000 |
| Nutanix software licensing (80 nodes) | $320,000 - $480,000 |
| Data center (power, cooling, space for 80 nodes) | $200,000 - $300,000 |
| Network infrastructure | $100,000 - $150,000 |
| Staff (infrastructure team) | Existing headcount |
| **Total (estimated)** | **$1,420,000 - $2,130,000** |

### 12.2 Proposed Hybrid State Cost (Estimated Annual)

| Item | Annual Cost (Estimated) |
|---|---|
| 50 Nutanix nodes (reduced fleet, hardware amortized) | $500,000 - $750,000 |
| Nutanix software licensing (50 nodes) | $200,000 - $300,000 |
| Data center (reduced power/cooling/space) | $140,000 - $200,000 |
| AWS Direct Connect (2x 10Gbps, dedicated) | $36,000 - $48,000 |
| AWS EKS control plane (always-on) | $876 (24/7 @ $0.10/hr) |
| AWS burst compute (6 weeks, ~60 instances avg) | $80,000 - $150,000 |
| AWS data tier during burst (RDS, ElastiCache, OpenSearch) | $15,000 - $30,000 |
| AWS networking (NAT Gateway, data transfer) | $10,000 - $25,000 |
| AWS monitoring and logging | $5,000 - $10,000 |
| Rancher licensing (if enterprise) or self-managed | $0 - $30,000 |
| Kubernetes platform investment (one-time, amortized) | $50,000 - $80,000 |
| Staff training (Kubernetes, AWS, one-time amortized) | $20,000 - $40,000 |
| **Total (estimated)** | **$1,057,000 - $1,664,000** |

### 12.3 Savings Analysis

| Scenario | Annual Savings | Percentage |
|---|---|---|
| Conservative estimate | $363,000 | ~25% |
| Mid-range estimate | $540,000 | ~35% |
| Optimistic estimate | $730,000 | ~45% |

**Key savings drivers:**
- 30 fewer Nutanix nodes to purchase, license, power, and cool
- AWS burst costs are usage-based (6 weeks vs. 52 weeks of owned hardware)
- Spot instances for 50-70% of burst compute (additional savings)
- No hardware depreciation on idle burst capacity

**Key cost additions:**
- Direct Connect circuits (always-on, but essential for reliability)
- AWS data transfer costs (monitor closely)
- Kubernetes platform build-out and training
- Ongoing AWS operational costs (monitoring, logging, security tools)

### 12.4 AWS Cost Optimization During Burst

- **Spot instances**: Use for stateless web/API tiers (50-70% savings vs On-Demand)
- **Savings Plans**: If burst compute is predictable, 1-year Compute Savings Plan for baseline burst
- **Right-sizing**: Start conservatively, let Karpenter optimize instance selection
- **Scale to zero**: Outside the 6-week burst, all burst resources are destroyed (except EKS control plane and DX)
- **Reserved capacity for Direct Connect**: Already amortized across 12 months
- **Data transfer**: Minimize data egress from AWS; use VPC endpoints for AWS service access

---

## 13. Implementation Roadmap

### Phase 0: Foundation (Weeks 1-4)

**Objective**: Establish hybrid connectivity and AWS landing zone.

| Task | Owner | Duration |
|---|---|---|
| Procure AWS Direct Connect circuits (lead time: 2-8 weeks) | Network team | Start immediately |
| Execute AWS BAA | Security/Legal | Week 1 |
| Set up AWS Organization, accounts, SCPs | Cloud team | Weeks 1-2 |
| Configure AWS IAM Identity Center with on-prem IdP | Identity team | Weeks 2-3 |
| Deploy AWS Config rules and HIPAA conformance pack | Security team | Weeks 3-4 |
| Enable CloudTrail, GuardDuty, Security Hub | Security team | Week 3 |
| Establish VPC, subnets, Transit Gateway | Network team | Weeks 2-3 |
| Set up VPN as interim connectivity (before DX is live) | Network team | Week 2 |

### Phase 1: Kubernetes On-Premises (Weeks 3-8)

**Objective**: Deploy production Kubernetes on Nutanix for steady-state workloads.

| Task | Owner | Duration |
|---|---|---|
| Deploy RKE2 cluster on Nutanix (3 control plane, 8 workers) | Platform team | Weeks 3-4 |
| Install Cilium CNI, configure network policies | Platform team | Week 4 |
| Deploy Nutanix CSI driver, configure StorageClasses | Platform team | Week 4 |
| Install ArgoCD for GitOps | Platform team | Week 5 |
| Deploy Rancher for multi-cluster management | Platform team | Week 5 |
| Set up Harbor container registry | Platform team | Week 5 |
| Deploy monitoring stack (Prometheus, Grafana, Loki, Alertmanager) | Platform team | Weeks 5-6 |
| Deploy cert-manager and Vault integration | Security team | Week 6 |
| Deploy Gatekeeper/Kyverno with HIPAA policies | Security team | Week 6 |
| Containerize first application (product catalog) | App team | Weeks 5-7 |
| Deploy first application to on-prem K8s | App/Platform team | Week 7 |
| Validate HIPAA controls on-prem | Security team | Week 8 |

### Phase 2: AWS Burst Infrastructure (Weeks 6-12)

**Objective**: Build the AWS burst environment and automation.

| Task | Owner | Duration |
|---|---|---|
| Direct Connect circuits active and tested | Network team | Week 6-8 (dependent on provider) |
| Provision EKS cluster (private endpoint) | Platform team | Weeks 7-8 |
| Register EKS in Rancher and ArgoCD | Platform team | Week 8 |
| Deploy EKS add-ons (ALB controller, External DNS, ADOT) | Platform team | Weeks 8-9 |
| Set up ECR with image replication from Harbor | Platform team | Week 9 |
| Deploy Kyverno PHI-blocking policy on EKS | Security team | Week 9 |
| Provision RDS read replica (test) | Platform team | Week 9 |
| Provision ElastiCache (test) | Platform team | Week 9 |
| Write Ansible burst-activate and burst-deactivate playbooks | Automation team | Weeks 8-11 |
| Configure Route 53 weighted routing | Network team | Week 10 |
| Deploy product catalog to EKS (first burst workload) | App/Platform team | Weeks 10-11 |
| End-to-end testing: on-prem + AWS | All teams | Weeks 11-12 |

### Phase 3: Application Migration and Testing (Weeks 10-18)

**Objective**: Containerize remaining burst-eligible applications and validate.

| Task | Owner | Duration |
|---|---|---|
| Containerize API gateway, search, cart, notifications | App teams | Weeks 10-16 |
| Implement PHI/non-PHI separation in application code | App teams | Weeks 10-14 |
| Deploy all burst-eligible apps to on-prem K8s | App/Platform team | Weeks 14-16 |
| Deploy all burst-eligible apps to EKS (test) | App/Platform team | Weeks 15-17 |
| Load testing: simulate 10x traffic across both environments | QA/Platform team | Weeks 16-18 |
| Chaos engineering: Direct Connect failure, node failures | SRE team | Weeks 17-18 |
| HIPAA compliance audit of full hybrid architecture | Security team | Week 18 |

### Phase 4: Production Burst (Pre-Peak Season)

**Objective**: Go live with hybrid burst for the next peak season.

| Task | Owner | Duration |
|---|---|---|
| Pre-season burst readiness check (Ansible playbook) | Platform team | T-4 weeks before peak |
| Activate burst environment (staged) | Platform team | T-2 weeks before peak |
| Gradual traffic shift: 90/10 --> 70/30 --> 50/50 | Platform/SRE team | T-1 week |
| Full burst operation | All teams | 6 weeks |
| Post-peak: deactivate burst, review costs, lessons learned | All teams | Peak + 1 week |

### Phase 5: Optimization and Steady-State (Ongoing)

| Task | Owner | Cadence |
|---|---|---|
| Hardware decommission planning (reduce to 50 nodes) | Infrastructure team | Next refresh cycle |
| Review AWS burst costs and optimize | FinOps/Platform team | Monthly during burst, quarterly otherwise |
| Update HIPAA risk assessment | Security team | Annually + after changes |
| Kubernetes platform upgrades | Platform team | Quarterly |
| Burst readiness drill | Platform/SRE team | Quarterly |
| Disaster recovery testing | SRE team | Semi-annually |

---

## 14. Disaster Recovery and Business Continuity

### 14.1 DR Scenarios

| Scenario | RTO | RPO | Strategy |
|---|---|---|---|
| On-prem K8s node failure | < 5 min | 0 | K8s self-healing, pod rescheduling |
| On-prem Nutanix node failure | < 15 min | 0 | Nutanix RF2/RF3 data protection, VM HA restart |
| On-prem cluster total loss | < 4 hours | < 15 min | Shift all traffic to AWS (degraded: no PHI ops), restore on-prem from Nutanix DR |
| AWS region failure during burst | < 5 min | 0 | Shift all traffic to on-prem (may require additional on-prem capacity -- keep some reserve) |
| Direct Connect failure | < 1 min (BFD) | 0 | Failover to second DX circuit or VPN |
| Both DX circuits + VPN failure | < 5 min | 0 | AWS operates independently for non-PHI; PHI operations queued until connectivity restored |

### 14.2 Degraded Mode Operations

When hybrid connectivity is lost, the architecture supports degraded-mode operation:

**AWS without on-prem connectivity:**
- Product browsing, search, cart: fully functional (cached data + read replicas)
- Checkout with health information: queued, not processed until connectivity restored
- Order placement: can accept orders, queue for fulfillment processing
- User presented with clear messaging about delayed processing

**On-prem without AWS:**
- Full functionality for all operations
- Increased load handled by on-prem capacity (may need to shed non-critical traffic)
- Route 53 health checks automatically shift all traffic to on-prem

---

## 15. Operational Runbooks (Summary)

### 15.1 Burst Season Activation Checklist

```
Pre-Activation (T-4 weeks):
[ ] Run burst-readiness-check.yml Ansible playbook
[ ] Verify Direct Connect circuit health (both circuits)
[ ] Verify AWS account limits sufficient (EC2, EBS, EIP quotas)
[ ] Update container images in ECR
[ ] Review and update Kubernetes manifests for current application versions
[ ] Verify HIPAA compliance controls on AWS (Config rules, SCPs)
[ ] Test RDS replication from on-prem to AWS
[ ] Confirm AWS BAA is current
[ ] Review AWS cost budget alerts

Activation (T-2 weeks):
[ ] Execute aws-burst-activate.yml with approved change ticket
[ ] Verify EKS nodes healthy and pods scheduling
[ ] Verify RDS read replica caught up
[ ] Verify ElastiCache cluster healthy
[ ] Validate application health checks on AWS
[ ] Begin gradual traffic shift (10% to AWS)
[ ] Monitor error rates, latency, cost

Ramp-Up (T-1 week):
[ ] Increase AWS traffic weight to 30%, then 50%
[ ] Monitor cross-environment latency
[ ] Verify autoscaler is responding to load
[ ] Confirm alerting is working for both environments

Peak Operation:
[ ] Daily cost review
[ ] Daily health check across both environments
[ ] Weekly HIPAA compliance scan
[ ] Incident response team briefed on hybrid architecture
```

### 15.2 Burst Season Deactivation Checklist

```
Post-Peak (Peak + 1 day):
[ ] Begin gradual traffic shift back to on-prem (AWS 30%, then 10%, then 0%)
[ ] Monitor on-prem capacity during shift-back
[ ] Allow 24h at 0% AWS traffic before teardown

Deactivation:
[ ] Execute aws-burst-deactivate.yml
[ ] Verify all AWS workloads terminated
[ ] Destroy RDS read replicas
[ ] Destroy ElastiCache clusters
[ ] Scale EKS node groups to 0
[ ] Verify EKS control plane remains (for quick reactivation)
[ ] Generate AWS cost report for burst period
[ ] Conduct lessons-learned review

Post-Season:
[ ] Document any issues encountered
[ ] Update runbooks and automation
[ ] Update cost projections for next season
[ ] Plan any hardware decommissions
```

---

## 16. Architecture Decision Records (ADRs)

### ADR-001: Kubernetes as Workload Abstraction Layer

**Status**: Accepted

**Context**: Need a portable workload format that runs identically on Nutanix (on-prem) and AWS (burst). Team is already starting Kubernetes adoption.

**Decision**: Use Kubernetes (RKE2 on-prem, EKS on AWS) as the common workload abstraction layer. Same container images, same Helm charts (with environment-specific values), same GitOps workflow via ArgoCD.

**Consequences**: Requires investment in Kubernetes platform skills; containerization of existing applications is a prerequisite; provides long-term portability beyond just AWS.

### ADR-002: PHI Data Remains On-Premises Only

**Status**: Accepted

**Context**: HIPAA compliance is non-negotiable. While AWS is HIPAA-eligible with BAA, minimizing PHI exposure in AWS reduces risk surface and audit scope.

**Decision**: All PHI data remains on-premises in Nutanix. AWS burst handles only non-PHI workloads (product catalog, search, cart, notifications). PHI-requiring operations route back to on-prem over Direct Connect.

**Consequences**: Application architecture must cleanly separate PHI and non-PHI operations; some checkout/order flows add latency from AWS-to-on-prem calls; significantly reduces HIPAA scope in AWS.

### ADR-003: RKE2 Over NKE for On-Premises Kubernetes

**Status**: Proposed

**Context**: Nutanix offers NKE (Nutanix Kubernetes Engine) which simplifies K8s on Nutanix. However, the hybrid architecture needs maximum portability.

**Decision**: Use RKE2 for on-premises Kubernetes. RKE2 provides FIPS-validated crypto, CIS hardening, and standard upstream Kubernetes API identical to EKS. Rancher manages both clusters.

**Consequences**: Slightly more operational overhead than NKE for initial setup; better portability and security posture; no additional Nutanix licensing for K8s.

**Alternative considered**: NKE is acceptable if the team strongly prefers Nutanix-managed Kubernetes and accepts the tighter coupling.

### ADR-004: AWS Direct Connect Over VPN-Only

**Status**: Accepted

**Context**: Hybrid connectivity between on-prem and AWS must support production traffic, database replication, and synchronous API calls during 10x peak load.

**Decision**: Two dedicated 10Gbps AWS Direct Connect circuits at different DX locations, with site-to-site VPN as tertiary failover.

**Consequences**: Higher cost than VPN-only (~$36-48K/year); provides consistent low-latency (~2-10ms) and predictable bandwidth; essential for database replication and synchronous PHI API calls during peak.

### ADR-005: ArgoCD for GitOps Over Flux

**Status**: Accepted

**Context**: Need a GitOps tool that manages multiple clusters (on-prem + AWS) from a single instance.

**Decision**: ArgoCD as the GitOps engine, running on-premises, managing both on-prem K8s and EKS clusters. ApplicationSets provide dynamic cluster targeting.

**Consequences**: ArgoCD's multi-cluster support and UI are mature; Flux is a strong FLOSS alternative but ArgoCD's ApplicationSet controller simplifies the burst activation/deactivation workflow.

### ADR-006: Spot Instances for Burst Compute

**Status**: Accepted

**Context**: AWS burst runs for only 6 weeks; cost optimization is critical for ROI.

**Decision**: Use Spot Instances for 50-70% of burst compute (stateless web and API workloads). On-Demand for system nodes and any workloads requiring guaranteed availability. Karpenter for instance diversification.

**Consequences**: Applications must handle graceful termination (2-minute warning); Karpenter diversifies across instance families to reduce interruption risk; estimated 50-70% compute cost savings during burst.

---

## 17. Risk Register

| ID | Risk | Probability | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | PHI leaks to AWS environment | Low | Critical | Admission controllers, network policies, application design, regular audits | Security team |
| R2 | Direct Connect insufficient bandwidth during peak | Medium | High | 2x 10Gbps circuits, monitor utilization, pre-provision additional if trending high | Network team |
| R3 | Kubernetes platform complexity overwhelms team | Medium | Medium | Phased rollout, training investment, Rancher for simplified management, hire/contract K8s expertise | Management |
| R4 | AWS costs exceed projections | Medium | Medium | Budget alerts, Karpenter right-sizing, Spot instances, weekly cost reviews during burst | FinOps |
| R5 | Application not cleanly separable into PHI/non-PHI | Medium | High | Invest in application refactoring early; this is a prerequisite for the architecture | App team |
| R6 | Spot instance interruptions during peak checkout | Medium | Medium | Diversified instance families, on-demand for critical paths, pod disruption budgets | Platform team |
| R7 | Latency on AWS-to-on-prem PHI calls unacceptable | Low | Medium | Aggressive caching, async patterns, DX circuit quality, performance testing pre-season | Platform/App team |
| R8 | Vendor lock-in to AWS | Low | Low | Kubernetes abstraction layer; could burst to Azure/GCP with same containers; DX is the main coupling | Architecture team |

---

## 18. Success Criteria

| Metric | Target | Measurement |
|---|---|---|
| Peak traffic handled without on-prem over-provisioning | 10x baseline with < 50% on-prem hardware | Load test results, production metrics |
| HIPAA compliance maintained | Zero findings in audit | Annual HIPAA audit, continuous compliance scanning |
| Burst activation time | < 30 minutes from trigger to serving traffic | Ansible playbook execution time |
| Burst deactivation time | < 2 hours to full teardown | Ansible playbook execution time |
| AWS burst cost (annual) | < 50% of equivalent owned hardware cost | AWS Cost Explorer vs. hardware amortization |
| Cross-environment latency (DX) | < 15ms p99 | Prometheus metrics |
| Availability during peak | 99.95% | Uptime monitoring |
| Infrastructure-as-code coverage | > 95% | Ansible/Terraform code review |
| PHI data boundary violations | Zero | Admission controller logs, network flow logs |

---

## 19. Appendix

### A. AWS HIPAA-Eligible Services (Key Subset for This Architecture)

- Amazon EKS
- Amazon EC2
- Elastic Load Balancing (ALB, NLB)
- Amazon RDS (PostgreSQL, MySQL)
- Amazon ElastiCache (Redis)
- Amazon S3
- Amazon CloudFront
- Amazon Route 53
- AWS KMS
- AWS CloudTrail
- Amazon CloudWatch
- AWS Config
- Amazon GuardDuty
- AWS Security Hub
- AWS Secrets Manager
- Amazon ECR
- AWS Direct Connect
- Amazon VPC (and VPC Flow Logs)
- Amazon OpenSearch Service
- Amazon SES

Full list: https://aws.amazon.com/compliance/hipaa-eligible-services-reference/

### B. Key Ansible Collections Required

```yaml
# collections/requirements.yml
collections:
  - name: kubernetes.core
    version: ">=2.4.0"
  - name: amazon.aws
    version: ">=6.0.0"
  - name: community.aws
    version: ">=6.0.0"
  - name: community.general
    version: ">=7.0.0"
  - name: nutanix.ncp
    version: ">=1.9.0"
```

### C. Kubernetes Version Compatibility

| Component | Version | Notes |
|---|---|---|
| RKE2 (on-prem) | 1.29.x | Match EKS version for API compatibility |
| EKS (AWS) | 1.29 | Align with on-prem K8s version |
| Rancher | 2.8.x+ | Must support both K8s versions |
| ArgoCD | 2.10.x+ | Multi-cluster support |
| Cilium | 1.15.x+ | On-prem CNI |
| AWS VPC CNI | Latest | EKS default CNI |

### D. Network CIDR Allocation

| Environment | CIDR | Purpose |
|---|---|---|
| On-prem production | 10.0.0.0/16 | Nutanix VMs, K8s pods, services |
| On-prem K8s pods | 10.1.0.0/16 | Cilium pod CIDR |
| On-prem K8s services | 10.2.0.0/16 | K8s ClusterIP services |
| AWS VPC | 10.100.0.0/16 | All AWS resources |
| AWS EKS pods | VPC-native (10.100.x.x) | VPC CNI assigns VPC IPs to pods |
| AWS EKS services | 172.20.0.0/16 | K8s ClusterIP services |

No CIDR overlap between on-prem and AWS ranges to ensure clean routing over Direct Connect.
