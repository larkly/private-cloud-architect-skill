# Hybrid Cloud Architecture: Nutanix On-Premises + AWS Burst for HIPAA-Compliant Healthcare E-Commerce

## Document Information

| Field | Value |
|-------|-------|
| Version | 1.0 |
| Date | 2026-03-20 |
| Classification | Internal / Confidential |
| Status | Architecture Proposal |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Assessment](#2-current-state-assessment)
3. [Requirements](#3-requirements)
4. [Target Architecture Overview](#4-target-architecture-overview)
5. [Network Architecture](#5-network-architecture)
6. [Compute and Orchestration Layer](#6-compute-and-orchestration-layer)
7. [Data Architecture](#7-data-architecture)
8. [HIPAA Compliance Framework](#8-hipaa-compliance-framework)
9. [Identity, Access, and Secrets Management](#9-identity-access-and-secrets-management)
10. [Burst Automation and Scaling Strategy](#10-burst-automation-and-scaling-strategy)
11. [Ansible Automation Architecture](#11-ansible-automation-architecture)
12. [Kubernetes Multi-Cluster Strategy](#12-kubernetes-multi-cluster-strategy)
13. [Observability and Monitoring](#13-observability-and-monitoring)
14. [Disaster Recovery and Business Continuity](#14-disaster-recovery-and-business-continuity)
15. [Security Architecture](#15-security-architecture)
16. [Cost Analysis](#16-cost-analysis)
17. [Migration and Implementation Roadmap](#17-migration-and-implementation-roadmap)
18. [Risks and Mitigations](#18-risks-and-mitigations)
19. [Appendices](#19-appendices)

---

## 1. Executive Summary

This document defines the hybrid cloud architecture for bursting healthcare e-commerce workloads from an on-premises Nutanix cluster (~80 nodes, NX and Dell XC series) to AWS during peak seasonal traffic (Black Friday / holiday season, approximately 6 weeks of 10x traffic). The architecture must maintain full HIPAA compliance across both environments at all times.

**Key design principles:**

- Nutanix remains the primary platform for steady-state workloads (the "home" for all services)
- AWS serves as an elastic burst target, not a permanent second data center
- All PHI/ePHI handling complies with HIPAA across both environments with a signed AWS BAA
- Kubernetes provides the workload portability layer between on-prem and cloud
- Ansible remains the infrastructure-as-code backbone, extended with AWS modules
- Burst capacity is automated and can be triggered gradually or rapidly based on traffic signals
- Hardware over-provisioning is eliminated, targeting a 40-60% reduction in idle on-prem capacity

**Expected outcomes:**

- Eliminate ~40 nodes of over-provisioned on-prem hardware (estimated $1.2-2M in avoided CapEx per refresh cycle)
- Achieve elastic scaling to 10x traffic within minutes during peak season
- Maintain continuous HIPAA compliance with unified audit and governance
- Reduce annual infrastructure cost by an estimated 30-45% versus current over-provisioned model

---

## 2. Current State Assessment

### 2.1 On-Premises Infrastructure

| Component | Details |
|-----------|---------|
| Platform | Nutanix AOS (assumed 6.x+) with AHV hypervisor |
| Hardware | ~80 nodes: mix of Nutanix NX series and Dell XC series |
| Management | Prism Central for unified management |
| Storage | Nutanix distributed storage fabric (hybrid or all-flash) |
| Networking | Assumed VLAN-segmented, likely with Nutanix Flow for microsegmentation |
| Automation | Ansible (heavy usage for provisioning, config management, deployments) |
| Containers | Kubernetes adoption in progress (likely Nutanix Kubernetes Engine / NKE or manual kubeadm) |
| Compliance | HIPAA controls implemented on-premises |

### 2.2 Current Pain Points

1. **Over-provisioned hardware**: Approximately 40-50% of on-prem capacity exists solely for the 6-week peak season, sitting idle the remaining 46 weeks
2. **Capital expenditure waste**: Each hardware refresh cycle includes capacity that is used <12% of the year
3. **Scaling ceiling**: Even with over-provisioning, there is a hard ceiling on burst capacity
4. **Lead time**: Adding physical capacity requires procurement cycles measured in weeks/months
5. **Kubernetes maturity**: Early-stage adoption means workloads are not yet fully portable

### 2.3 Workload Characterization

For the burst architecture, we need to categorize workloads:

| Tier | Examples | PHI Exposure | Burst Candidate | Notes |
|------|----------|--------------|-----------------|-------|
| Tier 1 - Stateless Web/API | Product catalog, search, CDN origin, cart API | Minimal/None | Excellent | First burst candidates |
| Tier 2 - Stateless with PHI | Patient profile API, order processing, Rx verification | Yes - ePHI | Good (with controls) | Requires HIPAA controls in AWS |
| Tier 3 - Stateful Services | Session stores, caches, message queues | Varies | Moderate | Redis, RabbitMQ/Kafka can run in both |
| Tier 4 - Databases | Primary transactional DBs, EHR integration | Yes - ePHI | Limited | Keep primary on-prem, read replicas in AWS |
| Tier 5 - Compliance/Audit | Audit logging, compliance reporting | Yes - metadata | No | Stays on-prem as system of record |

---

## 3. Requirements

### 3.1 Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | System must scale from 1x to 10x traffic within 15 minutes of trigger |
| FR-02 | Burst to AWS must be automated and repeatable, not a manual migration |
| FR-03 | All services running in AWS must have functional parity with on-prem versions |
| FR-04 | Data replication between on-prem and AWS must have RPO < 5 minutes for transactional data |
| FR-05 | Traffic routing must support gradual shift (canary/weighted) between on-prem and AWS |
| FR-06 | Burst-down (scale-in) must be automated to avoid lingering AWS costs |
| FR-07 | Ansible playbooks must work across both environments with environment-specific variables |

### 3.2 Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | End-to-end latency for cross-environment calls must be < 20ms (network RTT) |
| NFR-02 | 99.95% availability during peak season |
| NFR-03 | Full HIPAA compliance in both environments at all times |
| NFR-04 | All data in transit encrypted with TLS 1.2+ minimum (TLS 1.3 preferred) |
| NFR-05 | All data at rest encrypted with AES-256 |
| NFR-06 | Complete audit trail for all PHI access across both environments |
| NFR-07 | RTO < 30 minutes for any single-environment failure during peak |

### 3.3 HIPAA-Specific Requirements

| ID | Requirement |
|----|-------------|
| HIPAA-01 | Signed Business Associate Agreement (BAA) with AWS |
| HIPAA-02 | Only HIPAA-eligible AWS services may be used for PHI workloads |
| HIPAA-03 | Access controls must enforce minimum necessary standard |
| HIPAA-04 | Audit logs must be immutable and retained for 6+ years |
| HIPAA-05 | Breach notification procedures must cover both environments |
| HIPAA-06 | Risk assessment must cover the hybrid architecture |
| HIPAA-07 | Encryption in transit and at rest for all ePHI |
| HIPAA-08 | Workforce training must cover hybrid cloud procedures |

---

## 4. Target Architecture Overview

### 4.1 High-Level Architecture

```
                         +-----------------------+
                         |   Global Traffic Mgmt |
                         |  (Route 53 / F5 GTM)  |
                         +-----------+-----------+
                                     |
                    +----------------+----------------+
                    |                                 |
          +---------v----------+           +----------v---------+
          |   ON-PREMISES       |           |   AWS (Burst)       |
          |   Nutanix Cluster   |           |   us-east-1         |
          |                     |           |                     |
          | +--NKE K8s Cluster-+|           | +--EKS Cluster----+|
          | | Ingress (NGINX)  ||           | | ALB Ingress     ||
          | | App Pods         ||  <------> | | App Pods        ||
          | | Service Mesh     || AWS Direct| | Service Mesh    ||
          | +------------------+| Connect / | +------------------+|
          |                     | VPN       |                     |
          | +--VMs (Legacy)----+|           | +--EC2 (Legacy)---+|
          | | Non-K8s services ||           | | Non-K8s services||
          | +------------------+|           | +------------------+|
          |                     |           |                     |
          | +--Data Tier-------+|           | +--Data Tier------+|
          | | Primary DBs      || -------> | | Read Replicas   ||
          | | Redis Primary    ||  Repl.   | | Redis Replica   ||
          | | Kafka            ||           | | MSK / Kafka     ||
          | +------------------+|           | +------------------+|
          +---------------------+           +---------------------+
                    |                                 |
          +---------v---------+             +---------v---------+
          | Shared Services   |             | AWS Shared Svcs   |
          | - Vault           |  Federated  | - Secrets Manager |
          | - Ansible Tower   | ----------> | - SSM Param Store |
          | - Log Aggregation |             | - CloudWatch      |
          | - Prism Central   |             | - CloudTrail      |
          +-------------------+             +-------------------+
```

### 4.2 Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Burst target cloud | AWS | Largest HIPAA-eligible service catalog, mature BAA program, best healthcare ecosystem |
| Container orchestration | Kubernetes (NKE on-prem + EKS in AWS) | Workload portability, consistent deployment model, strong Ansible integration |
| Service mesh | Istio (with multi-cluster federation) | mTLS everywhere, traffic management, unified observability across clusters |
| Connectivity | AWS Direct Connect (primary) + IPsec VPN (backup) | Low latency, high bandwidth, encrypted by default |
| Global traffic management | Route 53 with health checks + on-prem GTM | Weighted routing for gradual burst, automated failover |
| Infrastructure as code | Ansible (existing) + Terraform (AWS resources) | Ansible for config/app, Terraform for AWS infrastructure lifecycle |
| Secrets management | HashiCorp Vault (primary) | Cross-environment secrets, dynamic credentials, audit logging |
| Container registry | Harbor (on-prem) mirrored to ECR (AWS) | Consistent images, vulnerability scanning, HIPAA audit trail |

---

## 5. Network Architecture

### 5.1 Connectivity Design

#### AWS Direct Connect

This is the backbone of the hybrid connectivity and is **mandatory** for this architecture given the latency and throughput requirements during peak burst.

| Parameter | Specification |
|-----------|---------------|
| Connection type | Dedicated Direct Connect |
| Port speed | 10 Gbps (primary) + 10 Gbps (secondary, diverse path) |
| VIF type | Private VIF (to VPC) + Transit VIF (if multi-VPC) |
| Encryption | MACsec (IEEE 802.1AE) on the Direct Connect link |
| BGP ASN | Private ASN for on-prem (e.g., 65000) |
| Redundancy | Two connections via different Direct Connect locations |
| Expected RTT | 1-5ms depending on geographic proximity |

**Important**: Direct Connect takes 2-8 weeks for physical provisioning. This must be initiated early in the project.

#### IPsec VPN (Backup)

| Parameter | Specification |
|-----------|---------------|
| Type | AWS Site-to-Site VPN over internet |
| Tunnels | 2 tunnels per VPN connection (AWS standard) |
| Encryption | AES-256, SHA-256, DH Group 20+ |
| Purpose | Backup path if Direct Connect fails, initial connectivity while DC is provisioned |
| Bandwidth | Limited by internet uplink (typically 1-2 Gbps) |

#### Network Topology (AWS Side)

```
                    +---------------------------+
                    |  AWS Transit Gateway       |
                    |  (Central routing hub)     |
                    +---+---+---+---+---+-------+
                        |   |   |   |   |
            +-----------+   |   |   |   +-----------+
            |               |   |   |               |
    +-------v------+ +-----v---v---v-----+ +-------v------+
    | Shared Svcs  | | Production VPC     | | Management   |
    | VPC          | | (HIPAA Workloads)  | | VPC          |
    | 10.100.0.0/16| | 10.200.0.0/16     | | 10.150.0.0/16|
    |              | |                     | |              |
    | - NAT GW     | | - EKS Cluster      | | - Bastion    |
    | - Endpoints  | | - EC2 Instances    | | - Ansible    |
    | - Route 53   | | - RDS Replicas     | | - Monitoring |
    | - ECR        | | - ElastiCache      | | - Vault      |
    +--------------+ | - MSK              | +--------------+
                     +---------------------+
```

### 5.2 VPC Design (Production)

| Subnet Tier | CIDR Example | AZ-a | AZ-b | AZ-c | Purpose |
|-------------|-------------|------|------|------|---------|
| Public | 10.200.1.0/24 | 10.200.1.0/26 | 10.200.1.64/26 | 10.200.1.128/26 | ALB, NAT Gateway |
| Private App | 10.200.10.0/22 | 10.200.10.0/24 | 10.200.11.0/24 | 10.200.12.0/24 | EKS worker nodes, EC2 |
| Private Data | 10.200.20.0/23 | 10.200.20.0/25 | 10.200.20.128/25 | 10.200.21.0/25 | RDS, ElastiCache, MSK |
| Private Mgmt | 10.200.30.0/24 | 10.200.30.0/26 | 10.200.30.64/26 | 10.200.30.128/26 | Monitoring, logging |

**IP address space planning**: Ensure no overlap between on-prem Nutanix subnets and AWS VPC CIDRs. Conduct a full IPAM audit before implementation.

### 5.3 DNS Architecture

```
On-Prem DNS                              AWS Route 53
+-----------------+                      +-------------------+
| Internal zones  |  Conditional         | Private hosted    |
| corp.internal   |  forwarding          | zones             |
| app.internal    | <------------------> | aws.app.internal  |
+-----------------+                      +-------------------+
        |                                         |
        +----------------+------------------------+
                         |
                  +------v-------+
                  | Route 53     |
                  | Public Zone  |
                  | example.com  |
                  | (Weighted    |
                  |  routing)    |
                  +--------------+
```

- **Split-horizon DNS**: Internal services resolve to local endpoints in each environment
- **Route 53 weighted routing**: Public traffic distributed between on-prem and AWS based on burst ratio
- **Health checks**: Route 53 health checks on both environments; automatic failover if one side is unhealthy
- **TTL**: Low TTL (60s) on weighted records during burst transitions for fast convergence

### 5.4 Firewall and Security Groups

#### On-Premises (Nutanix Flow)

- Microsegmentation policies for app-tier isolation
- Allow-list for Direct Connect/VPN subnets
- ePHI workloads in dedicated security zones
- East-west traffic inspection between tiers

#### AWS (Security Groups + NACLs)

| Security Group | Inbound | Outbound | Notes |
|----------------|---------|----------|-------|
| sg-alb | 443 from 0.0.0.0/0 | App tier SG, port 8080-8443 | TLS termination at ALB |
| sg-eks-nodes | ALB SG on app ports, on-prem CIDR on mesh ports | All egress to VPC CIDR, S3/ECR endpoints | Worker nodes |
| sg-data | App tier SG on DB/cache ports only | None (response traffic only) | RDS, ElastiCache |
| sg-mgmt | On-prem mgmt CIDR on SSH/22, 443 | All | Management access |

**NACL hardening**: Deny all traffic from non-RFC1918 addresses on private subnets. Explicit deny rules for known bad ranges.

---

## 6. Compute and Orchestration Layer

### 6.1 On-Premises (Nutanix)

#### Steady-State Capacity Planning

With the burst architecture in place, right-size the on-prem cluster:

| Current | Target | Savings |
|---------|--------|---------|
| ~80 nodes | ~45-50 nodes | 30-40 nodes decommissioned or not replaced at next refresh |
| Sized for 10x peak | Sized for 1.5-2x steady state (headroom for small spikes and burst ramp time) | ~$1.5M CapEx avoided per 4-year cycle |

The remaining on-prem nodes should handle:
- 100% of steady-state traffic (1x baseline)
- Initial spike absorption up to 2x while AWS burst ramps
- All Tier 5 compliance/audit workloads permanently
- Primary databases (Tier 4)

#### Nutanix Kubernetes Engine (NKE)

NKE provides the on-prem Kubernetes cluster:

| Parameter | Value |
|-----------|-------|
| Control plane | 3 master nodes (HA) |
| Worker nodes | 15-20 dedicated Nutanix VMs (scalable) |
| Node sizing | 8 vCPU / 32 GB RAM / 200 GB SSD per worker (adjust based on workload profiling) |
| CNI | Calico (default with NKE) or Cilium |
| Storage class | Nutanix CSI driver (Volumes for RWO, Files for RWX) |
| Registry | Harbor on Nutanix VM (image source of truth) |

### 6.2 AWS Compute (Burst Target)

#### EKS Cluster Configuration

| Parameter | Value |
|-----------|-------|
| EKS version | Match NKE Kubernetes version (within one minor version) |
| Node groups | Managed node groups with mixed instance types |
| Scaling | Karpenter for node auto-provisioning |
| Instance types | c6i.2xlarge / c6i.4xlarge (compute-optimized for web/API) |
| Spot vs On-Demand | 70% Spot (stateless tiers) / 30% On-Demand (PHI workloads) |
| GPU | Not required (no ML inference in burst scope) |

**Important Spot Instance consideration**: Stateless, horizontally-scaled services (Tier 1) are ideal for Spot. PHI-handling services (Tier 2) should run On-Demand to avoid disruption during compliance-sensitive transactions. Use Karpenter's node pool priorities to enforce this.

#### Karpenter Node Pool Configuration

```yaml
# Tier 1 - Stateless (Spot eligible)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: burst-stateless
spec:
  template:
    metadata:
      labels:
        workload-tier: stateless
        spot-eligible: "true"
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["c6i.2xlarge", "c6i.4xlarge", "c6a.2xlarge", "c6a.4xlarge", "m6i.2xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "2000"   # Max 2000 vCPUs for stateless tier
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 60s

---
# Tier 2 - PHI Workloads (On-Demand only)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: burst-phi
spec:
  template:
    metadata:
      labels:
        workload-tier: phi
        hipaa-compliant: "true"
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["c6i.2xlarge", "c6i.4xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: hipaa-encrypted
  limits:
    cpu: "500"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 300s
```

#### EC2 Node Class for HIPAA

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: hipaa-encrypted
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags:
        tier: private-app
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/prod-burst: owned
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true            # HIPAA: EBS encryption mandatory
        kmsKeyId: "alias/hipaa-ebs-key"
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 1     # Restrict IMDS to pod level
    httpTokens: required            # IMDSv2 only
  userData: |
    #!/bin/bash
    # CIS hardening, HIPAA audit logging
    /opt/scripts/harden-node.sh
```

### 6.3 Non-Kubernetes Workloads (Legacy VM Burst)

For services not yet containerized:

| Approach | Tools | Notes |
|----------|-------|-------|
| Ansible + Terraform | Terraform provisions EC2, Ansible configures | Existing Ansible roles work with minimal changes |
| AMI pipeline | Packer builds AMIs from Ansible roles | "Golden image" approach, faster launch times |
| Nutanix Move / AWS VM Import | Direct VM migration | Last resort; not recommended for burst due to slowness |

**Recommendation**: Prioritize containerizing Tier 1 and Tier 2 services before the first burst season. Use the AMI pipeline approach for any remaining VM-based services.

---

## 7. Data Architecture

### 7.1 Data Classification

| Classification | Examples | Storage On-Prem | Storage AWS | Sync Strategy |
|----------------|----------|-----------------|-------------|---------------|
| ePHI (Protected) | Patient records, Rx data, insurance info | Nutanix Volumes (encrypted) | RDS (encrypted, HIPAA-eligible) | Encrypted replication, on-prem primary |
| PII (Sensitive) | Names, addresses, payment data | Nutanix Volumes | RDS / DynamoDB | Same as ePHI |
| Business Data | Orders, inventory, pricing | Nutanix Volumes | RDS / DynamoDB | Async replication |
| Public Data | Product catalog, images, marketing | Nutanix Files / Object Store | S3 + CloudFront | CDN sync, eventual consistency OK |
| Transient Data | Sessions, carts, search indices | Redis on-prem | ElastiCache | Independent instances, no sync needed |

### 7.2 Database Strategy

#### Primary Databases (On-Premises)

The primary transactional databases **stay on-premises**. This is a deliberate choice:

1. **Data gravity**: The majority of writes originate from on-prem systems (order fulfillment, inventory management, EHR integrations)
2. **Compliance simplicity**: On-prem databases are already under existing HIPAA controls
3. **Complexity avoidance**: Multi-master across hybrid is fragile and risky for healthcare data

#### Read Replicas in AWS

For workloads that burst to AWS and need database access:

```
On-Prem (Nutanix)                    AWS
+------------------+                 +------------------+
| PostgreSQL 16    |  Logical        | RDS PostgreSQL   |
| Primary          | ------------->  | Read Replica     |
| (ePHI, writes)   |  Replication    | (reads only)     |
+------------------+  (encrypted)    +------------------+

+------------------+                 +------------------+
| MySQL 8.x        |  binlog         | Aurora MySQL     |
| Primary          | ------------->  | Read Replica     |
| (orders, catalog)|  replication    | (reads only)     |
+------------------+  (TLS)          +------------------+
```

**Replication configuration:**

| Parameter | Value |
|-----------|-------|
| Replication type | Logical replication (PostgreSQL) / binlog (MySQL) |
| Transport | TLS 1.3 over Direct Connect |
| Lag target | < 5 seconds under normal load, < 30 seconds under 10x peak |
| Monitoring | Custom Prometheus exporter for replication lag |
| Failover | Manual promotion only (no automatic failover to AWS) |

#### Write Path for AWS-Based Services

Services running in AWS that need to write data have two options:

**Option A: Write-back to on-prem (preferred for ePHI)**
- AWS services call on-prem API endpoints over Direct Connect
- Latency: 2-10ms per call (acceptable for most operations)
- Maintains single source of truth on-prem

**Option B: Local write with async sync (for non-PHI transactional data)**
- AWS services write to local RDS instance
- Change Data Capture (CDC) via Debezium streams changes back to on-prem
- Used for: cart operations, session data, non-PHI order metadata
- Conflict resolution: last-writer-wins with on-prem as tiebreaker

### 7.3 Caching Layer

| Component | On-Prem | AWS | Strategy |
|-----------|---------|-----|----------|
| Session cache | Redis Cluster (Nutanix VMs) | ElastiCache Redis (cluster mode) | Independent instances; sticky sessions route users to same environment |
| Application cache | Redis / Memcached | ElastiCache | Cache-aside pattern; each environment manages its own cache |
| CDN cache | N/A | CloudFront | Product images, static assets; origin on-prem or S3 |

### 7.4 Message Queue / Event Streaming

```
On-Prem                              AWS
+------------------+                 +------------------+
| Kafka Cluster    |  MirrorMaker2   | Amazon MSK       |
| (3+ brokers on   | ------------->  | (or self-managed |
|  Nutanix VMs)    |  (encrypted)    |  Kafka on EKS)   |
+------------------+                 +------------------+
```

- **MirrorMaker 2** replicates relevant topics from on-prem Kafka to AWS MSK
- Topic naming convention: `onprem.{topic}` for replicated topics, `aws.{topic}` for AWS-origin topics
- Consumer groups are independent per environment
- PHI-containing topics: encrypted at rest on both sides, access controlled via ACLs

### 7.5 Object Storage and Static Assets

| Asset Type | Primary | Burst | Sync |
|------------|---------|-------|------|
| Product images | Nutanix Objects | S3 + CloudFront | Rclone scheduled sync (hourly or on-change) |
| User uploads | Nutanix Files | S3 (pre-signed URLs) | Real-time sync via event-driven Lambda |
| Static web assets | Harbor/Nginx on-prem | S3 + CloudFront | CI/CD deploys to both |
| Compliance documents | Nutanix Files | NOT synced to AWS | Stays on-prem only |

---

## 8. HIPAA Compliance Framework

### 8.1 AWS BAA and Eligible Services

**Step 1**: Execute a Business Associate Agreement (BAA) with AWS. This is available through AWS Artifact in the AWS Management Console and must be accepted before deploying any PHI workloads.

**HIPAA-eligible AWS services to be used in this architecture:**

| Service | Use Case | PHI Exposure |
|---------|----------|--------------|
| EC2 | EKS worker nodes, legacy VMs | Yes (runs app code) |
| EKS | Container orchestration | Yes (orchestrates PHI workloads) |
| RDS (PostgreSQL/MySQL) | Database read replicas | Yes (ePHI data) |
| ElastiCache (Redis) | Session and application caching | Yes (session may contain PHI) |
| S3 | Static assets, logs, backups | Yes (audit logs, potential PHI in uploads) |
| MSK (Kafka) | Event streaming | Yes (PHI in event payloads) |
| CloudWatch | Logging and monitoring | Yes (logs may contain PHI) |
| CloudTrail | API audit logging | Yes (records PHI access) |
| KMS | Encryption key management | Yes (encrypts PHI) |
| Secrets Manager | Secrets storage | No (secrets, not PHI) |
| ALB | Load balancing | Transient (TLS termination) |
| Route 53 | DNS | No |
| Direct Connect | Network connectivity | Transient (encrypted) |
| GuardDuty | Threat detection | No (metadata only) |
| Security Hub | Compliance dashboard | No (findings only) |
| Config | Resource compliance | No (config data only) |
| ECR | Container registry | No (images, not PHI) |

**Services explicitly NOT to be used for PHI** (not HIPAA-eligible or not covered under standard BAA):
- Amazon Comprehend Medical (unless specifically added to BAA)
- Amazon Macie (use for detection, not for PHI storage)
- Any service not listed on the AWS HIPAA-eligible services page

### 8.2 HIPAA Safeguards Mapping

#### Administrative Safeguards

| HIPAA Requirement | Implementation |
|-------------------|----------------|
| Security Officer | Designated individual responsible for hybrid cloud security |
| Risk Assessment | Annual risk assessment covering both environments; updated when architecture changes |
| Workforce Training | Training includes hybrid cloud procedures, AWS console access policies |
| Access Management | Centralized IAM via Vault + AWS IAM Identity Center (SSO) |
| Contingency Plan | DR plan covers both environments (see Section 14) |
| Business Associate Agreements | AWS BAA signed; BAAs with any managed service providers |
| Security Incident Procedures | Unified incident response plan spanning both environments |

#### Physical Safeguards

| HIPAA Requirement | On-Prem | AWS |
|-------------------|---------|-----|
| Facility Access Controls | Existing DC physical security | AWS responsibility (shared responsibility model) |
| Workstation Use | Corporate policy | N/A (no physical workstations in AWS) |
| Device and Media Controls | Nutanix disk encryption, secure disposal | AWS handles physical media destruction (per BAA) |

#### Technical Safeguards

| HIPAA Requirement | Implementation |
|-------------------|----------------|
| Access Control (Unique User ID) | Individual IAM users/roles, no shared accounts; federated via SAML/OIDC |
| Access Control (Emergency Access) | Break-glass procedure with Vault emergency tokens |
| Access Control (Automatic Logoff) | Session timeouts on all management interfaces (15 min) |
| Audit Controls | CloudTrail + on-prem audit logs aggregated to central SIEM |
| Integrity Controls | Checksums on data replication, S3 object lock for audit logs |
| Transmission Security | TLS 1.2+ everywhere, MACsec on Direct Connect, IPsec on VPN |
| Encryption (at rest) | Nutanix self-encrypting drives + software encryption; AWS KMS with CMK for all services |

### 8.3 PHI Data Flow Audit

Every PHI data flow must be documented and approved. Key flows in this architecture:

| Flow ID | Source | Destination | Transport | Encryption | Approved |
|---------|--------|-------------|-----------|------------|----------|
| PHI-001 | On-prem DB | AWS RDS replica | Direct Connect + TLS 1.3 | In-transit: TLS; At-rest: KMS | Required |
| PHI-002 | User browser | AWS ALB | Internet + TLS 1.3 | TLS termination at ALB | Required |
| PHI-003 | AWS EKS pod | On-prem API | Direct Connect + mTLS (Istio) | mTLS + TLS | Required |
| PHI-004 | On-prem Kafka | AWS MSK | Direct Connect + TLS | In-transit: TLS; At-rest: KMS | Required |
| PHI-005 | AWS EKS pod | ElastiCache | VPC internal + TLS | In-transit: TLS; At-rest: KMS | Required |
| PHI-006 | Application logs | CloudWatch | Internal + TLS | In-transit: TLS; At-rest: KMS | Required - PII scrubbing required |

### 8.4 Compliance Monitoring and Audit

| Tool | Purpose | Environment |
|------|---------|-------------|
| AWS Config Rules | Continuous compliance checking for AWS resources | AWS |
| AWS Security Hub | HIPAA compliance standard dashboard | AWS |
| CloudTrail | API-level audit logging | AWS |
| Nutanix Flow | Network policy compliance | On-prem |
| HashiCorp Vault Audit | Secrets access logging | Both |
| SIEM (Splunk/Elastic) | Centralized log analysis and alerting | Both (aggregated) |
| Custom Ansible playbooks | Compliance scanning and remediation | Both |

---

## 9. Identity, Access, and Secrets Management

### 9.1 Identity Architecture

```
+------------------+
| Corporate IdP    |    SAML 2.0 / OIDC
| (Okta/Azure AD)  +------------------------+
+--------+---------+                        |
         |                                  |
    SAML |                           +------v--------+
         |                           | AWS IAM       |
+--------v---------+                 | Identity      |
| Vault            |                 | Center (SSO)  |
| (On-prem primary)|                 +-------+-------+
| + OIDC provider  |                         |
+--------+---------+                 +-------v-------+
         |                           | AWS IAM Roles |
         |                           | (Per-service) |
    +----v----+                      +---------------+
    | K8s RBAC|
    | (NKE)   |
    +---------+
```

### 9.2 AWS IAM Strategy

**Account structure** (AWS Organizations):

```
Root Account (Management)
  +-- Security Account (GuardDuty, Security Hub, CloudTrail org trail)
  +-- Shared Services Account (Transit Gateway, Direct Connect, ECR, Route 53)
  +-- Production Account (EKS, RDS, burst workloads) <-- HIPAA workloads here
  +-- Staging Account (Pre-production testing of burst)
  +-- Log Archive Account (Immutable audit logs - S3 with Object Lock)
```

**IAM Roles for EKS (IRSA - IAM Roles for Service Accounts):**

Every Kubernetes service account maps to an IAM role with least-privilege permissions:

```yaml
# Example: Order service needs RDS read and SQS access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-order-service
---
# Corresponding IAM policy (Terraform)
# {
#   "Effect": "Allow",
#   "Action": ["rds-db:connect"],
#   "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:*/order_svc_readonly"
# }
```

### 9.3 HashiCorp Vault Architecture

Vault serves as the cross-environment secrets management platform:

| Parameter | Value |
|-----------|-------|
| Deployment | HA cluster on Nutanix VMs (3 nodes, Raft storage) |
| Auth methods | OIDC (humans), Kubernetes (pods), AppRole (Ansible), AWS IAM (AWS services) |
| Secret engines | KV v2 (static secrets), Database (dynamic DB creds), PKI (TLS certs), AWS (dynamic IAM creds) |
| Audit | File audit device + syslog to SIEM |
| Seal | Auto-unseal via Nutanix KMS or AWS KMS (for DR) |

**Critical for HIPAA**: Vault provides:
- Dynamic, short-lived database credentials (rotated every 1 hour)
- Complete audit trail of every secret access
- Encryption as a service for ePHI fields
- PKI for mTLS certificates in the service mesh

#### Vault Access from AWS EKS

```
AWS EKS Pod --> IRSA (IAM Role) --> Vault AWS Auth Backend --> Vault Policy --> Secret
```

The Vault AWS auth method validates the pod's IAM identity and issues a Vault token with appropriate policies. No long-lived credentials are stored in AWS.

### 9.4 Certificate Management

| Certificate Type | Issuer | Automation | Rotation |
|------------------|--------|------------|----------|
| Public TLS (*.example.com) | Let's Encrypt or commercial CA | cert-manager in K8s | 90-day auto-renewal |
| Internal mTLS (service mesh) | Vault PKI engine (intermediate CA) | Istio CSR integration | 24-hour TTL, automatic |
| Database TLS | Vault PKI engine | Ansible role | 30-day rotation |
| Direct Connect MACsec | Pre-shared key | Manual (AWS managed) | Annual or on-demand |

---

## 10. Burst Automation and Scaling Strategy

### 10.1 Burst Lifecycle

The burst lifecycle has five phases:

```
Phase 1        Phase 2          Phase 3           Phase 4         Phase 5
PREPARE -----> WARM-UP -------> BURST-ACTIVE ---> SCALE-DOWN ---> DECOMMISSION
(Oct 1)        (Nov 1)          (Nov 15-Dec 31)   (Jan 1-15)     (Jan 15-31)
```

| Phase | Duration | Actions |
|-------|----------|---------|
| PREPARE | ~4 weeks | Validate AWS infra, update AMIs, sync data, run load tests |
| WARM-UP | ~2 weeks | Deploy services to AWS, start replication, canary 5% traffic |
| BURST-ACTIVE | ~6 weeks | Full burst active, auto-scaling based on traffic, 30-70% traffic to AWS |
| SCALE-DOWN | ~2 weeks | Gradually shift traffic back to on-prem, reduce AWS capacity |
| DECOMMISSION | ~2 weeks | Tear down AWS burst resources, stop replication, archive logs |

### 10.2 Scaling Triggers and Thresholds

#### Pre-Scheduled Scaling (Known Events)

Black Friday, Cyber Monday, and holiday dates are known in advance. Pre-scale AWS capacity:

```yaml
# Karpenter provisioner scaling schedule (via CronJob that adjusts HPA targets)
burst_schedule:
  black_friday_prep:
    date: "2026-11-26T00:00:00Z"
    min_replicas_multiplier: 5    # 5x normal
  black_friday:
    date: "2026-11-27T00:00:00Z"
    min_replicas_multiplier: 10   # 10x normal
  cyber_monday:
    date: "2026-11-30T00:00:00Z"
    min_replicas_multiplier: 8
  holiday_sustained:
    start: "2026-12-01T00:00:00Z"
    end: "2026-12-31T23:59:59Z"
    min_replicas_multiplier: 4
```

#### Dynamic Scaling (Real-Time)

| Metric | Source | Threshold | Action |
|--------|--------|-----------|--------|
| Request rate (RPS) | Istio / ALB metrics | > 2x baseline sustained 5 min | Increase AWS traffic weight by 10% |
| On-prem CPU utilization | Prism Central / Prometheus | > 70% sustained 5 min | Shift 20% traffic to AWS |
| P95 latency | Application metrics | > 500ms sustained 3 min | Shift additional traffic to AWS |
| Error rate | Application metrics | > 1% sustained 2 min | Alert + investigate (don't auto-scale on errors alone) |
| On-prem memory utilization | Prism Central / Prometheus | > 80% sustained 5 min | Shift 20% traffic to AWS |

### 10.3 Traffic Shifting Mechanism

#### Global Server Load Balancing (GSLB)

```python
# Simplified traffic shifting logic (implemented in automation controller)

def calculate_traffic_split(metrics):
    """
    Returns (onprem_weight, aws_weight) as percentages.
    """
    onprem_cpu = metrics['onprem_avg_cpu']
    onprem_rps = metrics['onprem_current_rps']
    onprem_capacity_rps = metrics['onprem_max_rps']

    # Base: all traffic on-prem
    if onprem_rps < onprem_capacity_rps * 0.6:
        return (100, 0)

    # Ramp: start shifting to AWS
    elif onprem_rps < onprem_capacity_rps * 0.8:
        aws_pct = int((onprem_rps / onprem_capacity_rps - 0.6) * 250)  # 0-50%
        return (100 - aws_pct, aws_pct)

    # Full burst: aggressive AWS usage
    else:
        aws_pct = min(70, int((onprem_rps / onprem_capacity_rps - 0.6) * 350))
        return (100 - aws_pct, aws_pct)

    # Never send more than 70% to AWS (on-prem remains primary)
```

**Implementation**: Route 53 weighted routing records, updated via AWS SDK calls from the automation controller. Changes propagate within 60 seconds (TTL).

#### Session Affinity During Burst

- Users are assigned to an environment (on-prem or AWS) at session start
- Sticky sessions via cookie (`X-Env: onprem|aws`) set at the GSLB layer
- If a user's environment becomes unhealthy, their next request is routed to the other environment
- Session data is in Redis (independent per environment), so environment switch means cold session -- mitigate with session serialization to a shared store for critical user state

### 10.4 Burst Readiness Checklist

Automated pre-burst validation (runs daily during PREPARE phase):

```yaml
burst_readiness_checks:
  infrastructure:
    - direct_connect_status: UP (both links)
    - vpn_backup_status: UP
    - eks_cluster_health: HEALTHY
    - rds_replica_lag: < 10s
    - ecr_images_synced: true
    - dns_health_checks: PASSING

  security:
    - aws_baa_status: ACTIVE
    - vault_seal_status: UNSEALED
    - certificates_expiry: > 90 days
    - security_hub_findings_critical: 0
    - iam_roles_audited: true
    - encryption_keys_rotated: < 90 days

  application:
    - all_services_deployed_aws: true
    - health_checks_passing_aws: true
    - smoke_tests_passing_aws: true
    - load_test_completed: true (>= 5x target)
    - rollback_tested: true

  compliance:
    - hipaa_config_rules_compliant: true
    - audit_logging_verified: true
    - phi_data_flow_approved: true
    - incident_response_plan_updated: true
```

---

## 11. Ansible Automation Architecture

### 11.1 Repository Structure

```
ansible/
+-- inventories/
|   +-- on-prem/
|   |   +-- hosts.yml              # Nutanix VMs (static + dynamic from Prism)
|   |   +-- group_vars/
|   |       +-- all.yml
|   |       +-- webservers.yml
|   |       +-- databases.yml
|   +-- aws/
|   |   +-- aws_ec2.yml            # Dynamic inventory plugin for EC2
|   |   +-- group_vars/
|   |       +-- all.yml
|   |       +-- tag_role_webserver.yml
|   |       +-- tag_role_database.yml
|   +-- hybrid/
|       +-- hosts.yml              # Cross-environment inventory (combines both)
|
+-- roles/
|   +-- common/                    # OS hardening, CIS benchmarks (works on both)
|   +-- hipaa-baseline/            # HIPAA-specific hardening
|   +-- app-deploy/                # Application deployment (environment-agnostic)
|   +-- monitoring-agent/          # Prometheus node exporter, log shipping
|   +-- nutanix-vm/                # Nutanix-specific provisioning (via Prism API)
|   +-- aws-ec2/                   # AWS-specific provisioning (via boto3)
|   +-- kubernetes-deploy/         # Helm chart deployment (works on NKE and EKS)
|   +-- database-replica/          # Database replication setup
|   +-- vault-agent/               # Vault agent configuration
|   +-- burst-controller/          # Burst automation logic
|
+-- playbooks/
|   +-- site.yml                   # Full site deployment
|   +-- deploy-app.yml             # Application deployment (both environments)
|   +-- burst-prepare.yml          # Phase 1: Prepare AWS infrastructure
|   +-- burst-warmup.yml           # Phase 2: Deploy services, start replication
|   +-- burst-activate.yml         # Phase 3: Enable traffic shifting
|   +-- burst-scaledown.yml        # Phase 4: Shift traffic back
|   +-- burst-decommission.yml     # Phase 5: Tear down AWS resources
|   +-- compliance-scan.yml        # HIPAA compliance scanning
|   +-- disaster-recovery.yml      # DR procedures
|
+-- collections/
|   +-- requirements.yml           # amazon.aws, community.general, kubernetes.core
|
+-- terraform/                     # AWS infrastructure (invoked by Ansible)
|   +-- modules/
|   |   +-- vpc/
|   |   +-- eks/
|   |   +-- rds/
|   |   +-- elasticache/
|   |   +-- direct-connect/
|   +-- environments/
|       +-- production/
|       +-- staging/
|
+-- ansible.cfg
+-- Makefile                       # Common operations
```

### 11.2 Key Ansible Patterns

#### Environment Abstraction

```yaml
# group_vars/all.yml (shared across environments)
app_name: "healthcare-ecommerce"
app_version: "{{ lookup('env', 'APP_VERSION') }}"
hipaa_mode: true
tls_min_version: "1.2"
encryption_algorithm: "AES-256"

# group_vars/on-prem/all.yml
environment: "on-prem"
database_host: "db-primary.corp.internal"
redis_host: "redis-cluster.corp.internal"
vault_addr: "https://vault.corp.internal:8200"
container_registry: "harbor.corp.internal"

# group_vars/aws/all.yml
environment: "aws"
database_host: "{{ rds_endpoint }}"
redis_host: "{{ elasticache_endpoint }}"
vault_addr: "https://vault.corp.internal:8200"  # Still on-prem, accessed via Direct Connect
container_registry: "{{ aws_account_id }}.dkr.ecr.us-east-1.amazonaws.com"
```

#### Burst Prepare Playbook

```yaml
# playbooks/burst-prepare.yml
---
- name: "Phase 1: Prepare AWS Burst Infrastructure"
  hosts: localhost
  connection: local
  vars:
    burst_season: "{{ lookup('env', 'BURST_SEASON') | default('2026-holiday') }}"

  tasks:
    - name: "Terraform: Provision AWS infrastructure"
      community.general.terraform:
        project_path: "{{ playbook_dir }}/../terraform/environments/production"
        state: present
        variables:
          burst_season: "{{ burst_season }}"
          eks_node_count_min: 5
          eks_node_count_max: 200
      register: tf_output

    - name: "Validate Direct Connect connectivity"
      amazon.aws.ec2_vpc_vpn_info:
        region: us-east-1
      register: vpn_status

    - name: "Configure database replication"
      include_role:
        name: database-replica
      vars:
        replica_endpoint: "{{ tf_output.outputs.rds_endpoint.value }}"
        source_host: "{{ hostvars['db-primary']['ansible_host'] }}"

    - name: "Sync container images to ECR"
      include_tasks: tasks/sync-images-to-ecr.yml
      loop: "{{ container_images }}"

    - name: "Deploy Kubernetes manifests to EKS"
      include_role:
        name: kubernetes-deploy
      vars:
        kubeconfig: "{{ tf_output.outputs.eks_kubeconfig.value }}"
        target_cluster: "aws-burst"

    - name: "Run compliance scan on AWS resources"
      include_tasks: tasks/aws-compliance-scan.yml

    - name: "Run burst readiness checks"
      include_tasks: tasks/burst-readiness-checks.yml

    - name: "Send readiness report"
      community.general.slack:
        token: "{{ vault_lookup('secret/slack/token') }}"
        channel: "#platform-ops"
        msg: "Burst infrastructure ready for {{ burst_season }}. Readiness: {{ readiness_result }}"
```

#### Kubernetes Deployment Role (Environment-Agnostic)

```yaml
# roles/kubernetes-deploy/tasks/main.yml
---
- name: "Set kubeconfig based on target"
  set_fact:
    active_kubeconfig: >-
      {{ kubeconfig_onprem if target_cluster == 'on-prem'
         else kubeconfig_aws }}

- name: "Deploy Helm charts"
  kubernetes.core.helm:
    kubeconfig: "{{ active_kubeconfig }}"
    release_name: "{{ item.name }}"
    chart_ref: "{{ item.chart }}"
    release_namespace: "{{ item.namespace }}"
    values:
      image:
        registry: "{{ container_registry }}"
        tag: "{{ app_version }}"
      environment: "{{ environment }}"
      hipaa:
        enabled: true
        encryption: true
        auditLogging: true
      resources:
        requests:
          cpu: "{{ item.cpu_request }}"
          memory: "{{ item.mem_request }}"
        limits:
          cpu: "{{ item.cpu_limit }}"
          memory: "{{ item.mem_limit }}"
  loop: "{{ application_charts }}"
```

### 11.3 Ansible Automation Platform (AAP) / AWX

| Configuration | Value |
|---------------|-------|
| Deployment | AWX on Nutanix VM (or AAP if licensed) |
| Execution environments | Custom EE with AWS SDK, Terraform, kubectl, Vault CLI |
| Credential management | Vault integration (external credential lookup) |
| RBAC | Teams aligned to burst operation roles |
| Schedules | Burst readiness checks (daily during prepare phase) |
| Webhooks | Git push triggers deployment pipelines |
| Notifications | Slack, PagerDuty, email for burst operations |

---

## 12. Kubernetes Multi-Cluster Strategy

### 12.1 Cluster Architecture

| Parameter | NKE (On-Prem) | EKS (AWS) |
|-----------|----------------|-----------|
| Kubernetes version | 1.29.x (match versions) | 1.29.x |
| Control plane | Self-managed (NKE) | AWS-managed |
| Worker nodes | 15-20 Nutanix VMs | 5-200 EC2 (Karpenter) |
| CNI | Calico | VPC CNI (aws-cni) |
| Service mesh | Istio 1.21+ | Istio 1.21+ |
| Ingress | NGINX Ingress Controller | AWS ALB Ingress Controller |
| Storage | Nutanix CSI | EBS CSI + EFS CSI |
| DNS | CoreDNS | CoreDNS |

### 12.2 Multi-Cluster Service Mesh (Istio)

Istio provides the critical cross-cluster communication layer:

```
NKE Cluster                              EKS Cluster
+---------------------------+            +---------------------------+
| istiod (control plane)    |  <------>  | istiod (control plane)    |
| Istio CA (shared root)    |  Multi-    | Istio CA (shared root)    |
+---------------------------+  cluster   +---------------------------+
|                           |  config    |                           |
| +-----+ mTLS  +-------+  |            | +-----+ mTLS  +-------+  |
| |Svc A|<----->|Svc B  |  |            | |Svc A|<----->|Svc B  |  |
| +-----+       +---+---+  |            | +-----+       +---+---+  |
|                   |       |            |                   |       |
+---------------------------+            +---------------------------+
                    |          mTLS                   |
                    +---------- over ----------------+
                               Direct Connect
```

**Multi-cluster Istio configuration:**

```yaml
# Istio multi-cluster: shared trust domain
# Both clusters share the same root CA (from Vault PKI)
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
spec:
  values:
    global:
      meshID: healthcare-mesh
      multiCluster:
        clusterName: nke-onprem  # or eks-aws
      network: network-onprem     # or network-aws
    pilot:
      env:
        EXTERNAL_ISTIOD: "false"
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    enableAutoMtls: true
    defaultConfig:
      holdApplicationUntilProxyStarts: true
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY  # Strict: only allow registered services
```

**Cross-cluster service discovery**: Istio's multi-cluster model enables services in EKS to discover and call services in NKE (and vice versa) transparently. Traffic between clusters flows over Direct Connect, encrypted with mTLS.

### 12.3 Namespace Strategy

```yaml
# Consistent namespaces across both clusters
namespaces:
  - name: production
    labels:
      istio-injection: enabled
      hipaa-compliant: "true"
      environment: production

  - name: production-phi
    labels:
      istio-injection: enabled
      hipaa-compliant: "true"
      phi-tier: "true"
      pod-security.kubernetes.io/enforce: restricted

  - name: monitoring
    labels:
      istio-injection: enabled

  - name: istio-system
    labels:
      app: istio
```

### 12.4 GitOps with ArgoCD

```
+-------------------+
| Git Repository    |
| (app manifests)   |
+--------+----------+
         |
    +----v----+
    | ArgoCD  |  (on-prem, manages both clusters)
    | Server  |
    +----+----+
         |
    +----+----+----+
    |              |
+---v---+     +---v---+
| NKE   |     | EKS   |
| Apps   |     | Apps   |
+-------+     +-------+
```

ArgoCD ApplicationSets for multi-cluster deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ecommerce-apps
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            burst-target: "true"
  template:
    metadata:
      name: '{{name}}-ecommerce'
    spec:
      project: production
      source:
        repoURL: https://git.corp.internal/platform/k8s-manifests.git
        targetRevision: main
        path: 'apps/ecommerce/overlays/{{metadata.labels.environment}}'
      destination:
        server: '{{server}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 12.5 Pod Security and HIPAA Hardening

```yaml
# Pod Security Standard: Restricted (for PHI workloads)
apiVersion: v1
kind: Pod
metadata:
  name: phi-service-example
  namespace: production-phi
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: harbor.corp.internal/ecommerce/phi-service:v1.2.3
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: phi-service-db  # Injected by Vault Agent
              key: password
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2000m"
          memory: "2Gi"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: vault-secrets
          mountPath: /vault/secrets
          readOnly: true
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
    - name: vault-secrets
      emptyDir:
        medium: Memory  # Secrets in tmpfs, never on disk
        sizeLimit: 10Mi
  serviceAccountName: phi-service
  automountServiceAccountToken: false  # Explicitly mount only when needed
```

### 12.6 Network Policies

```yaml
# Default deny all in PHI namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production-phi
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow specific service communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-service
  namespace: production-phi
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
        - namespaceSelector:
            matchLabels:
              name: production
      ports:
        - protocol: TCP
          port: 8443
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database-proxy
      ports:
        - protocol: TCP
          port: 5432
    - to:  # Allow DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

---

## 13. Observability and Monitoring

### 13.1 Unified Observability Stack

```
+---------------------------------------------------------------------+
|                     Unified Dashboards (Grafana)                     |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    |       |       |       |       |       |       |       |
+---v---+ +-v-----+ +-----v-+ +---v---+ +--v----+ +------v--+-----+
|Metrics| |Logging| |Tracing| |Alerts | |Compli-| |Business |     |
|Mimir/ | |Loki/  | |Tempo/ | |Alert- | |ance   | |Metrics  |     |
|Thanos | |Elastic| |Jaeger | |manager| |Dashbd | |Custom   |     |
+---+---+ +---+---+ +---+---+ +---+---+ +---+---+ +---+-----+     |
    |         |         |         |         |         |             |
+---v---------v---------v---------v---------v---------v-------------+
|                    Collection Layer                                 |
| On-Prem: Prometheus + Promtail + OTel Collector                    |
| AWS:     Prometheus (via AMP) + CloudWatch + OTel Collector        |
+-------------------------------------------------------------------+
```

### 13.2 Metrics

| Component | On-Prem Collection | AWS Collection | Storage |
|-----------|-------------------|----------------|---------|
| Infrastructure | Prometheus + node_exporter | CloudWatch + Prometheus (AMP) | Mimir/Thanos (on-prem) |
| Kubernetes | kube-state-metrics + cAdvisor | Same + EKS control plane metrics | Mimir/Thanos |
| Application | Prometheus client libraries | Same | Mimir/Thanos |
| Nutanix | Prism Central API exporter | N/A | Mimir/Thanos |
| Network | SNMP exporter (switches) | VPC Flow Logs + Transit GW metrics | Mimir + S3 |

**Key dashboards:**

1. **Burst Overview**: Traffic split, on-prem vs AWS request rates, latency comparison
2. **Capacity Planning**: On-prem utilization, AWS node count, scaling headroom
3. **HIPAA Compliance**: Encryption status, access anomalies, audit log health
4. **Cross-Environment Latency**: Direct Connect latency, cross-cluster service call latency
5. **Data Replication**: Database replica lag, Kafka mirror lag, S3 sync status
6. **Cost Tracker**: Real-time AWS spend vs budget, cost per request

### 13.3 Logging

**HIPAA logging requirements:**
- All PHI access must be logged
- Logs must be immutable and retained for 6+ years
- Logs must not contain PHI themselves (log sanitization required)

| Log Type | Collection | Storage | Retention |
|----------|------------|---------|-----------|
| Application logs | Promtail/Fluentbit sidecar | Loki (on-prem) + S3 (archive) | 1 year hot, 6 years cold |
| Kubernetes audit logs | K8s audit policy | Loki + S3 | 6 years |
| AWS CloudTrail | Automatic | S3 (Object Lock, WORM) in Log Archive account | 7 years |
| Vault audit logs | Vault file audit device | Loki + S3 | 7 years |
| Database query logs | PostgreSQL pg_audit / MySQL audit plugin | Loki + S3 | 6 years |
| Network flow logs | Nutanix Flow / VPC Flow Logs | S3 | 1 year |

**Log sanitization pipeline:**

```
Application --> Fluentbit (sidecar) --> PII/PHI Redaction Filter --> Loki/S3
                                        |
                                        +-- Patterns:
                                            - SSN: \d{3}-\d{2}-\d{4} --> [REDACTED-SSN]
                                            - Email in PHI context --> [REDACTED-EMAIL]
                                            - Patient ID --> [REDACTED-PATIENT-ID]
                                            - Credit card --> [REDACTED-CC]
```

### 13.4 Distributed Tracing

OpenTelemetry instrumentation across all services:

- **On-prem**: OTel Collector --> Tempo (on-prem)
- **AWS**: OTel Collector --> AWS X-Ray (HIPAA eligible) + Tempo (on-prem via DC)
- **Cross-environment traces**: Trace context propagation via Istio (B3/W3C headers) ensures end-to-end trace visibility even when a request spans on-prem and AWS

### 13.5 Alerting

| Alert | Severity | Condition | Action |
|-------|----------|-----------|--------|
| Direct Connect down | P1 | Both DC links down | Page on-call, verify VPN failover |
| Replica lag > 30s | P2 | Sustained for 5 min | Alert team, consider pausing AWS traffic |
| AWS spend > daily budget | P2 | Cost anomaly detected | Alert FinOps + platform team |
| HIPAA compliance drift | P1 | AWS Config rule non-compliant | Auto-remediate if possible, alert security |
| Burst capacity exhaustion | P2 | EKS at Karpenter limits | Increase limits or reduce traffic to AWS |
| Certificate expiry < 7 days | P2 | Any cert in either env | Auto-renew or alert |
| PHI access anomaly | P1 | Unusual PHI query patterns | Alert security team, trigger investigation |
| Error rate > 5% | P1 | Either environment | Page on-call, potential rollback |

---

## 14. Disaster Recovery and Business Continuity

### 14.1 Failure Scenarios

| Scenario | Impact | RTO | RPO | Recovery Procedure |
|----------|--------|-----|-----|--------------------|
| Single Nutanix node failure | None (HA within cluster) | 0 (automatic) | 0 | Nutanix self-healing |
| On-prem cluster degraded (multiple nodes) | Reduced capacity | 15 min | 0 | Shift traffic to AWS (if burst is active) |
| On-prem complete outage | Total on-prem loss | 30 min | 5 min | Promote AWS to primary, promote DB replicas |
| AWS region failure | Loss of burst capacity | 15 min | 0 | Shift all traffic to on-prem (capacity permitting) |
| Direct Connect failure | No cross-env communication | 5 min | 0 | Automatic failover to VPN |
| Both DC and VPN failure | Environments isolated | Variable | Variable | Operate independently; AWS reads from local replicas |
| Database corruption | Data integrity issue | 1-4 hours | Varies | Point-in-time recovery from backups |

### 14.2 Backup Strategy

| Data Type | Backup Method | Frequency | Retention | Storage |
|-----------|---------------|-----------|-----------|---------|
| Databases (on-prem) | Nutanix snapshots + pg_dump/mysqldump | Hourly snapshots, daily full | 30 days snapshots, 1 year daily, 7 years monthly | Nutanix + S3 (encrypted) |
| Databases (AWS replicas) | RDS automated backups | Continuous (point-in-time) | 35 days | RDS (same region) + S3 cross-region |
| Kubernetes state | Velero backups | Every 6 hours | 30 days | S3 (on-prem Nutanix Objects + AWS S3) |
| Configuration | Git (Ansible + Terraform repos) | On every change | Indefinite | Git server + GitHub/GitLab |
| Vault | Vault snapshot + Raft backup | Daily | 90 days | Encrypted on Nutanix + S3 |
| Audit logs | Already on S3 with Object Lock | Continuous | 7 years | S3 Glacier (after 90 days) |

### 14.3 DR Testing Schedule

| Test | Frequency | Scope |
|------|-----------|-------|
| Backup restore verification | Monthly | Restore random database backup, verify integrity |
| Direct Connect failover to VPN | Quarterly | Simulate DC failure, verify VPN takes over |
| On-prem to AWS traffic shift | Before each burst season | Full traffic shift drill |
| AWS replica promotion | Annually | Promote RDS replica to primary, run smoke tests |
| Full DR exercise | Annually | Simulate complete on-prem outage during burst |

---

## 15. Security Architecture

### 15.1 Defense in Depth

```
Layer 1: Perimeter
  - AWS WAF on ALB (OWASP Top 10, rate limiting, geo-blocking)
  - On-prem WAF/IPS (existing)
  - DDoS protection (AWS Shield Standard included; consider Advanced)
  - Route 53 DNS firewall

Layer 2: Network
  - VPC segmentation (public/private/data tiers)
  - Security groups (stateful, least privilege)
  - NACLs (stateless, defense in depth)
  - Nutanix Flow microsegmentation
  - Istio authorization policies (L7)

Layer 3: Compute
  - Hardened AMIs/VM images (CIS benchmarks)
  - IMDSv2 enforced (no v1)
  - Read-only root filesystems in containers
  - Non-root containers
  - Seccomp and AppArmor profiles
  - Regular patching (automated via Ansible)

Layer 4: Application
  - mTLS between all services (Istio)
  - Input validation
  - Output encoding
  - CSRF/XSS protections
  - API rate limiting
  - JWT validation at gateway

Layer 5: Data
  - Encryption at rest (AES-256 everywhere)
  - Encryption in transit (TLS 1.2+ everywhere)
  - Field-level encryption for sensitive PHI (via Vault Transit engine)
  - Data masking in non-production environments
  - Key rotation (90-day policy)

Layer 6: Identity
  - MFA required for all human access
  - Short-lived credentials (Vault dynamic secrets)
  - IRSA for AWS service access
  - No long-lived API keys
  - Just-in-time access for administrative operations
```

### 15.2 AWS-Specific Security Controls

| Control | Implementation |
|---------|----------------|
| SCPs | Service Control Policies blocking non-HIPAA-eligible services in production account |
| GuardDuty | Enabled in all accounts; findings sent to Security Hub |
| Security Hub | HIPAA standard enabled; automated findings triage |
| Config Rules | HIPAA conformance pack + custom rules |
| Macie | Enabled on S3 buckets to detect accidental PHI in logs |
| Inspector | Container image scanning in ECR |
| Access Analyzer | IAM policy analysis; external access detection |
| CloudTrail | Organization-level trail; log file validation enabled |
| KMS | CMKs with key policies restricting access; automatic rotation |

#### SCP: Block Non-HIPAA Services

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
        "mechanicalturk:*",
        "sumerian:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1"]
        }
      }
    },
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "sts:*",
        "support:*"
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

### 15.3 Vulnerability Management

| Scope | Tool | Frequency | Action |
|-------|------|-----------|--------|
| Container images | Trivy (in CI/CD) + ECR scanning + Harbor scanning | Every build + daily | Block deployment if critical/high CVEs |
| OS packages | Ansible + yum/apt security updates | Weekly (automated) | Auto-patch non-critical; manual review for critical |
| Kubernetes | kube-bench (CIS) | Weekly | Remediate via Ansible |
| Application dependencies | Dependabot / Snyk | Continuous | PR-based remediation |
| Infrastructure | tfsec + checkov (Terraform) | Every plan | Block non-compliant Terraform changes |
| Penetration testing | Third-party firm | Annually + pre-burst | Remediate findings before burst season |

### 15.4 Incident Response

Unified incident response plan covering both environments:

1. **Detection**: GuardDuty, Security Hub, SIEM alerts, Prometheus alerts
2. **Triage**: On-call engineer assesses severity and PHI impact
3. **Containment**:
   - AWS: Security group lockdown, WAF rules, IAM policy revocation
   - On-prem: Nutanix Flow quarantine, firewall rules
   - Cross-env: Disable Direct Connect/VPN if lateral movement suspected
4. **Eradication**: Identify and remove threat; rotate all potentially compromised credentials via Vault
5. **Recovery**: Restore from known-good state; re-enable services
6. **Post-incident**: Root cause analysis, HIPAA breach assessment (if PHI involved, 60-day notification clock starts)

---

## 16. Cost Analysis

### 16.1 Current State Cost (Estimated Annual)

| Item | Cost | Notes |
|------|------|-------|
| Nutanix hardware (80 nodes, amortized over 4 years) | ~$1,200,000/yr | Mix of NX and Dell XC |
| Nutanix software licenses | ~$400,000/yr | AOS + Prism Pro |
| Data center (power, cooling, space) | ~$300,000/yr | For 80 nodes |
| Network infrastructure | ~$100,000/yr | Switches, firewalls |
| **Total current annual cost** | **~$2,000,000/yr** | |

### 16.2 Target State Cost (Estimated Annual)

| Item | Cost | Notes |
|------|------|-------|
| Nutanix hardware (45 nodes, amortized) | ~$675,000/yr | 44% reduction |
| Nutanix software licenses (45 nodes) | ~$225,000/yr | Reduced node count |
| Data center (45 nodes) | ~$200,000/yr | Less power/cooling |
| Network infrastructure | ~$100,000/yr | Unchanged |
| AWS Direct Connect (2x 10Gbps) | ~$180,000/yr | Port fees + data transfer |
| AWS burst compute (6 weeks peak) | ~$150,000-250,000 | EC2/EKS with Spot mix |
| AWS data services (6 weeks) | ~$50,000-80,000 | RDS, ElastiCache, MSK |
| AWS always-on (management, DR) | ~$36,000/yr | Minimal footprint |
| AWS support (Business tier) | ~$15,000/yr | Required for production |
| Vault / tooling licenses | ~$30,000/yr | If using enterprise Vault |
| **Total target annual cost** | **~$1,511,000-1,641,000/yr** | |

### 16.3 Cost Savings

| Metric | Value |
|--------|-------|
| Annual savings | ~$360,000 - $490,000 (18-25%) |
| 4-year savings (one refresh cycle) | ~$1,440,000 - $1,960,000 |
| Break-even on Direct Connect investment | ~8 months |
| Avoided CapEx at next refresh | ~$600,000 - $900,000 (35 nodes not purchased) |

### 16.4 AWS Cost Optimization Strategies

| Strategy | Estimated Savings | Notes |
|----------|-------------------|-------|
| Spot Instances for Tier 1 | 60-70% vs On-Demand | Stateless workloads only |
| Savings Plans for always-on components | 30-40% vs On-Demand | 1-year commitment for baseline AWS resources |
| Right-sizing (Karpenter) | 15-20% vs static sizing | Automatic bin-packing |
| Aggressive scale-down | Significant | Terminate burst resources within days of traffic drop |
| S3 lifecycle policies | Minor | Move old logs to Glacier |
| Reserved capacity for Direct Connect | Included above | Port fee is fixed |

### 16.5 Cost Governance

- **AWS Budgets**: Daily budget alerts for burst spending
- **Cost Allocation Tags**: Mandatory tags on all AWS resources (`project`, `environment`, `burst-season`, `workload-tier`)
- **FinOps reviews**: Weekly during burst, monthly otherwise
- **Anomaly detection**: AWS Cost Anomaly Detection enabled
- **Auto-termination**: Lambda function that terminates untagged resources after 24 hours

---

## 17. Migration and Implementation Roadmap

### 17.1 Phase 0: Foundation (Months 1-2)

| Task | Owner | Duration | Dependencies |
|------|-------|----------|--------------|
| Sign AWS BAA | Legal + Security | 1 week | None |
| Set up AWS Organizations + accounts | Platform | 2 weeks | BAA |
| Order AWS Direct Connect | Platform + Network | 1 day (order); 4-8 weeks (delivery) | AWS account |
| Implement AWS landing zone (VPC, TGW, security) | Platform | 3 weeks | AWS accounts |
| Set up VPN (temporary connectivity) | Network | 1 week | AWS account |
| Vault multi-environment auth setup | Platform | 2 weeks | VPN |
| Extend Ansible for AWS (collections, dynamic inventory) | Platform | 2 weeks | AWS accounts |
| Terraform module development (VPC, EKS, RDS) | Platform | 3 weeks | AWS accounts |

### 17.2 Phase 1: Containerization Sprint (Months 2-4)

| Task | Owner | Duration | Dependencies |
|------|-------|----------|--------------|
| Containerize Tier 1 services (product catalog, search, static) | App teams | 6 weeks | None |
| Containerize Tier 2 services (patient APIs, order processing) | App teams | 6 weeks | None |
| Set up NKE cluster (on-prem Kubernetes) | Platform | 2 weeks | None |
| Set up Harbor registry | Platform | 1 week | NKE |
| Deploy Tier 1 services to NKE | App teams | 2 weeks | NKE + containers |
| Deploy Tier 2 services to NKE | App teams | 2 weeks | NKE + containers |
| Implement CI/CD pipeline for multi-cluster | Platform | 3 weeks | NKE |
| Set up ArgoCD for GitOps | Platform | 1 week | NKE |

### 17.3 Phase 2: AWS Burst Infrastructure (Months 4-6)

| Task | Owner | Duration | Dependencies |
|------|-------|----------|--------------|
| Direct Connect activation and testing | Network | 2 weeks | DC delivered |
| Provision EKS cluster | Platform | 2 weeks | DC active |
| Configure Istio multi-cluster mesh | Platform | 3 weeks | NKE + EKS |
| Set up database replication (on-prem to RDS) | DBA | 3 weeks | DC active |
| Configure ECR image mirroring from Harbor | Platform | 1 week | EKS |
| Deploy Tier 1 services to EKS | App teams | 2 weeks | EKS + mesh |
| Deploy Tier 2 services to EKS | App teams | 2 weeks | EKS + mesh + HIPAA controls |
| Implement traffic shifting (Route 53 weighted) | Platform | 2 weeks | Services deployed |
| Set up ElastiCache, MSK in AWS | Platform | 2 weeks | VPC |
| AWS security hardening (SCPs, GuardDuty, Config) | Security | 3 weeks | AWS accounts |
| HIPAA compliance validation in AWS | Compliance | 2 weeks | All AWS resources |

### 17.4 Phase 3: Testing and Validation (Months 6-8)

| Task | Owner | Duration | Dependencies |
|------|-------|----------|--------------|
| Functional testing in AWS | QA | 2 weeks | Phase 2 complete |
| Load testing (on-prem only, AWS only, hybrid) | Performance | 3 weeks | Phase 2 complete |
| Chaos engineering (DC failure, node failure, AZ failure) | Platform | 2 weeks | Load testing |
| Security penetration testing (hybrid scope) | Third party | 2 weeks | Phase 2 complete |
| HIPAA risk assessment (updated for hybrid) | Compliance | 3 weeks | All testing |
| DR testing (failover scenarios) | Platform | 2 weeks | Phase 2 complete |
| Burst automation end-to-end testing | Platform | 2 weeks | All above |
| Runbook development and training | Platform + Ops | 2 weeks | All testing |

### 17.5 Phase 4: First Burst Season (Months 8-10)

| Task | Owner | Duration | Dependencies |
|------|-------|----------|--------------|
| Execute burst-prepare playbook | Platform | 1 day | Phase 3 complete |
| Burst readiness review (go/no-go) | All stakeholders | 1 day | Preparation complete |
| Execute burst-warmup (canary traffic to AWS) | Platform | 2 weeks | Go decision |
| Black Friday / holiday burst-active | Platform + Ops | 6 weeks | Warm-up complete |
| Daily monitoring and adjustment | Ops | Ongoing | During burst |
| Execute burst-scaledown | Platform | 2 weeks | Traffic normalizes |
| Execute burst-decommission | Platform | 2 weeks | Scale-down complete |
| Post-season retrospective | All | 1 week | Season ends |

### 17.6 Timeline Summary

```
Month:  1    2    3    4    5    6    7    8    9    10   11   12
        |----|----|----|----|----|----|----|----|----|----|----|----|
Phase 0: [========]
Phase 1:      [================]
Phase 2:                [================]
Phase 3:                          [================]
Phase 4:                                        [====================]
                                                 ^
                                                 |
                                            Black Friday
                                            (target: Year 1)
```

**Critical path**: Direct Connect provisioning (4-8 weeks lead time) is on the critical path. Order immediately.

---

## 18. Risks and Mitigations

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|------------|--------|------------|
| R1 | Direct Connect not ready in time | Medium | High | Order early; use VPN as fallback (reduced bandwidth) |
| R2 | Containerization takes longer than planned | Medium | High | Prioritize Tier 1 services; use AMI pipeline for non-containerized services |
| R3 | Database replication lag during 10x traffic | Medium | Medium | Pre-scale RDS, tune replication, implement circuit breakers |
| R4 | Spot Instance interruptions during peak | Medium | Low | Karpenter handles replacement; use multiple instance types; Tier 2 on On-Demand |
| R5 | HIPAA audit finding blocks deployment | Low | Critical | Engage compliance team from day 1; continuous compliance checks |
| R6 | Cross-environment latency higher than expected | Low | Medium | Benchmark early; optimize data access patterns; cache aggressively |
| R7 | Ansible playbook differences between environments | Medium | Medium | Extensive testing in staging; environment abstraction layer |
| R8 | Team skill gap on AWS/Kubernetes | Medium | Medium | Training budget; pair programming; consider short-term consulting |
| R9 | AWS cost overrun during burst | Medium | Medium | Budgets, alerts, auto-termination; weekly FinOps reviews |
| R10 | Vendor lock-in to AWS | Low | Low | Kubernetes as portability layer; avoid AWS-proprietary APIs in app code |
| R11 | MACsec key management for Direct Connect | Low | Medium | Document procedures; automate rotation; test failover |
| R12 | Secrets sprawl across environments | Medium | High | Vault as single source of truth; no secrets in environment variables or config maps |

---

## 19. Appendices

### Appendix A: HIPAA-Eligible AWS Services Reference

Maintain an up-to-date list from: https://aws.amazon.com/compliance/hipaa-eligible-services-reference/

Review this list quarterly and before any new AWS service adoption.

### Appendix B: AWS Resource Tagging Standard

All AWS resources must have these tags:

| Tag Key | Required | Example Values | Purpose |
|---------|----------|----------------|---------|
| `project` | Yes | `healthcare-ecommerce` | Cost allocation |
| `environment` | Yes | `production`, `staging` | Environment identification |
| `burst-season` | Yes (during burst) | `2026-holiday` | Season tracking |
| `workload-tier` | Yes | `tier-1-stateless`, `tier-2-phi` | Workload classification |
| `hipaa-phi` | Yes | `true`, `false` | PHI flag |
| `owner` | Yes | `platform-team` | Ownership |
| `managed-by` | Yes | `terraform`, `ansible` | IaC tracking |
| `cost-center` | Yes | `engineering-infra` | Finance |

### Appendix C: Runbook Index

| Runbook | Trigger | Owner |
|---------|---------|-------|
| RB-001: Burst Season Preparation | 8 weeks before peak | Platform Lead |
| RB-002: Burst Activation | Traffic threshold or scheduled | On-call Engineer |
| RB-003: Direct Connect Failover | DC link alarm | Network Engineer |
| RB-004: Database Replica Promotion | On-prem DB unavailable during burst | DBA |
| RB-005: Emergency Burst Deactivation | Critical issue in AWS | On-call Engineer |
| RB-006: HIPAA Breach Response | PHI exposure detected | Security Officer |
| RB-007: Cost Overrun Response | Budget alert triggered | Platform Lead + FinOps |
| RB-008: Kubernetes Node Troubleshooting | Node NotReady | On-call Engineer |
| RB-009: Certificate Emergency Rotation | Cert compromise | Security + Platform |
| RB-010: Complete On-Prem Outage | Data center failure | Platform Lead (DR Commander) |

### Appendix D: Decision Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| - | AWS as burst target (not Azure/GCP) | Largest HIPAA service catalog, most mature BAA program, team familiarity | Azure (strong HIPAA but less service breadth), GCP (fewer HIPAA-eligible services) |
| - | Keep primary DB on-prem | Data gravity, compliance simplicity, avoids multi-master complexity | Aurora Global Database (too complex for burst model), CockroachDB (operational overhead) |
| - | Istio for service mesh | Multi-cluster support, mTLS, traffic management | Linkerd (lighter but less multi-cluster support), Consul Connect (adds Consul dependency) |
| - | Karpenter over Cluster Autoscaler | Better bin-packing, faster scaling, node pool flexibility | Cluster Autoscaler (slower, less flexible with instance types) |
| - | Vault over AWS-native secrets | Cross-environment consistency, dynamic secrets, audit trail | AWS Secrets Manager (AWS-only), CyberArk (cost) |
| - | Direct Connect over VPN-only | Latency, bandwidth, reliability for 10x traffic burst | VPN-only (insufficient bandwidth), SD-WAN (additional complexity) |

### Appendix E: Glossary

| Term | Definition |
|------|------------|
| BAA | Business Associate Agreement - legal contract required by HIPAA when sharing PHI |
| ePHI | Electronic Protected Health Information |
| NKE | Nutanix Kubernetes Engine (formerly Karbon) |
| EKS | Amazon Elastic Kubernetes Service |
| IRSA | IAM Roles for Service Accounts (EKS feature) |
| mTLS | Mutual TLS - both client and server authenticate |
| GSLB | Global Server Load Balancing |
| DC | Direct Connect (AWS dedicated network connection) |
| CMK | Customer Managed Key (AWS KMS) |
| WORM | Write Once Read Many (immutable storage) |
| SCP | Service Control Policy (AWS Organizations) |
| PHI | Protected Health Information |

---

*This architecture document should be reviewed and updated quarterly, and specifically before each burst season. All changes must go through the architecture review board and HIPAA compliance review.*
