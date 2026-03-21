# Architecture: Air-Gapped Kubernetes Platform for HEMLIG Classified Data

## Swedish Defense Contractor -- Private Cloud Reference Architecture

**Classification:** HEMLIG (Secret) per Sakerhetsskyddsforordningen (2021:955)
**Regulatory Framework:** Sakerhetsskyddslagen (2018:585), Forsvarsmakten (FM) and FMV accreditation requirements
**Accreditation Authority:** FMV / MUST (Militara underrattelse- och sakerhetsinstitutet)
**Date:** 2026-03-21
**Version:** 1.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Compliance Context](#2-regulatory-and-compliance-context)
3. [Architecture Principles](#3-architecture-principles)
4. [Physical Infrastructure and Facility](#4-physical-infrastructure-and-facility)
5. [Network Architecture](#5-network-architecture)
6. [Kubernetes Platform Architecture](#6-kubernetes-platform-architecture)
7. [Storage Architecture](#7-storage-architecture)
8. [Identity, Access, and Clearance Management](#8-identity-access-and-clearance-management)
9. [Cryptographic Architecture](#9-cryptographic-architecture)
10. [Air-Gap Operations Model](#10-air-gap-operations-model)
11. [Software Supply Chain](#11-software-supply-chain)
12. [Monitoring, Logging, and Audit](#12-monitoring-logging-and-audit)
13. [Backup, Recovery, and Data Destruction](#13-backup-recovery-and-data-destruction)
14. [Staffing and Clearance Model](#14-staffing-and-clearance-model)
15. [Accreditation Roadmap](#15-accreditation-roadmap)
16. [Risk Register](#16-risk-register)
17. [Technology Selection Summary](#17-technology-selection-summary)

---

## 1. Executive Summary

This document defines the architecture for an air-gapped, Kubernetes-based private cloud platform capable of processing data classified at the Swedish HEMLIG (Secret) level. The platform is designed to meet the requirements of Sakerhetsskyddslagen (2018:585) and its associated ordinances, with the goal of achieving accreditation from FMV and MUST.

**Key design decisions:**

- Kubernetes-based container orchestration (recommended: RKE2 or upstream Kubernetes with hardened configuration, not managed cloud Kubernetes)
- Fully air-gapped -- no connectivity to the internet or any unclassified network
- All infrastructure physically located in Sweden, in facilities meeting SS 3492 / FortV requirements
- Hardware sourced from approved vendors with supply chain attestation
- All personnel with logical or physical access hold Swedish security clearance at HEMLIG level via Sapo
- Cryptographic components approved by MUST (Swedish-approved COMSEC)

---

## 2. Regulatory and Compliance Context

### 2.1 Governing Legislation and Standards

| Regulation / Standard | Relevance |
|---|---|
| Sakerhetsskyddslagen (2018:585) | Primary law governing protection of security-sensitive activities |
| Sakerhetsskyddsforordningen (2021:955) | Detailed requirements for information security classification |
| MUST IT-sakerhetshandbok (ITSM) | Technical security requirements for classified IT systems |
| Forsvarsmaktens Kravdokument (FM KravIT) | Specific technical accreditation criteria |
| PMESII / NATO equivalence | HEMLIG maps roughly to NATO SECRET |
| SS 3492 (physical security) | Physical protection requirements for facilities |
| FMV Sakerhetsavtal | Security agreement between FMV and contractor |

### 2.2 Accreditation Requirements

The platform must undergo a formal accreditation process (systemsakerhetsgodkannande) managed by FMV with technical evaluation support from MUST. Key deliverables include:

- **Sakerhetsanalys (Security Analysis):** Threat assessment and risk evaluation
- **Systemsakerhetsplan (System Security Plan):** Detailed description of all security controls
- **Driftsakerinstruktion (Operational Security Instructions):** Procedures for day-to-day operations
- **Incidenthanteringsplan (Incident Response Plan):** Procedures for security incidents
- **Kryptoplan (Cryptographic Plan):** All cryptographic components and key management
- **Tempestdokumentation (TEMPEST Documentation):** Emanation security compliance

### 2.3 Data Sovereignty

All data, at rest and in transit, must remain within Swedish national borders at all times. No data, metadata, telemetry, logs, or diagnostic information may leave the air-gapped boundary. No foreign-controlled cloud services or SaaS components are permitted.

---

## 3. Architecture Principles

| # | Principle | Rationale |
|---|---|---|
| 1 | **Air-gap is absolute** | No network path may exist between the classified environment and any unclassified or internet-connected network |
| 2 | **Defense in depth** | Multiple independent security layers; no single point of failure in security controls |
| 3 | **Least privilege everywhere** | All access (human and system) scoped to minimum required; RBAC enforced at every layer |
| 4 | **Swedish data sovereignty** | All compute, storage, and network components physically in Sweden; no foreign dependencies at runtime |
| 5 | **Clearance-gated access** | No logical or physical access without verified Sapo clearance at appropriate level |
| 6 | **Auditable by design** | Every action logged, tamper-evident, and available for MUST inspection |
| 7 | **Approved cryptography only** | All cryptographic operations use MUST-approved algorithms and implementations |
| 8 | **Reproducible and declarative** | Infrastructure as code; GitOps within the air-gap; deterministic builds |
| 9 | **Minimal attack surface** | Only required services run; all others disabled; hardened OS baselines |
| 10 | **Operational sustainability** | Architecture must be operable by the 12 cleared engineers available |

---

## 4. Physical Infrastructure and Facility

### 4.1 Facility Requirements

The data center hosting the platform must meet requirements for processing HEMLIG information:

- **Physical location:** Within Sweden, not in shared commercial data center
- **Security zone classification:** Sakerhetsskyddat omrade (Security Protected Area), inner zone classified for HEMLIG
- **Access control:** Multi-factor physical access (badge + biometric + PIN), mantrap entry, 24/7 guarding or equivalent alarm/monitoring
- **TEMPEST:** Equipment and facility must meet SDIP-27 Level B (or as specified by MUST zoning assessment); shielded room (skarmad rum) likely required
- **Power:** Redundant power feeds, UPS, diesel generator with minimum 72h fuel
- **Fire suppression:** Gas-based suppression (e.g., Novec/FM-200), early smoke detection (VESDA)
- **Environmental:** Redundant cooling, temperature/humidity monitoring

### 4.2 Hardware Specification

```
Recommended Compute Nodes:
---------------------------------------------------------------------------
Role              | Count | Spec (minimum)
---------------------------------------------------------------------------
Control plane     | 3     | 2x Intel Xeon (or AMD EPYC), 256 GB ECC RAM,
                  |       | 2x 960 GB NVMe (OS/etcd), dual 25 GbE
Worker nodes      | 6-9   | 2x Intel Xeon (or AMD EPYC), 512 GB ECC RAM,
                  |       | 2x 1.92 TB NVMe, dual 25 GbE
Storage nodes     | 3-5   | 2x CPU, 256 GB RAM, 12x 7.68 TB NVMe,
                  |       | dual 25 GbE + dedicated storage network
Infrastructure    | 3     | For registry, GitOps, logging, monitoring
                  |       | 128 GB RAM, 2x 1.92 TB NVMe
Admin/jump nodes  | 2     | Hardened workstations for cluster administration
---------------------------------------------------------------------------
```

**Hardware sourcing considerations:**
- Procure through FMV-approved channels or with supply chain attestation
- Verify no unauthorized firmware modifications (hardware attestation / TPM-based boot verification)
- Maintain hardware inventory with serial numbers, tracked in the accreditation documentation
- Consider Swedish or European hardware vendors where possible to reduce supply chain risk
- All hardware must be physically inspected upon delivery in a controlled environment

### 4.3 Network Hardware

- **Switches:** Enterprise-grade, supporting 802.1Q VLANs, no management plane internet connectivity, firmware verified
- **Firewalls:** Two layers -- perimeter (surrounding the classified zone) and internal microsegmentation
- **No wireless:** Wi-Fi and Bluetooth must be physically disabled or absent in the classified zone
- **Diodes (optional, for future):** Hardware data diodes if one-way data import is required from lower classification levels

---

## 5. Network Architecture

### 5.1 Network Zones

```
+------------------------------------------------------------------+
|                    UNCLASSIFIED ZONE                              |
|  (Corporate network, internet -- NO connectivity to below)       |
+==================================================================+
                         AIR GAP (physical separation)
+==================================================================+
|                 CLASSIFIED ZONE (HEMLIG)                         |
|                                                                  |
|  +--------------------+    +--------------------+                |
|  |  MANAGEMENT VLAN   |    |   DATA IMPORT      |                |
|  |  (admin access,    |    |   STAGING AREA      |                |
|  |   jump hosts)      |    |   (data diode /     |                |
|  |  10.100.0.0/24     |    |    sneakernet)      |                |
|  +--------+-----------+    +--------+-----------+                |
|           |                         |                            |
|  +--------+-------------------------+-----------+                |
|  |              CORE NETWORK                     |                |
|  |              10.200.0.0/16                    |                |
|  |                                               |                |
|  |  +-------------+  +-------------+  +---------+|               |
|  |  | K8s Control |  | K8s Worker  |  | Storage ||               |
|  |  | Plane VLAN  |  | Node VLAN   |  | VLAN    ||               |
|  |  | 10.200.1/24 |  | 10.200.2/22 |  | 10.200 ||               |
|  |  |             |  |             |  | .10/24  ||               |
|  |  +-------------+  +-------------+  +---------+|               |
|  |                                               |                |
|  |  +-------------+  +-------------+             |                |
|  |  | Registry/   |  | Monitoring  |             |                |
|  |  | GitOps VLAN |  | VLAN        |             |                |
|  |  | 10.200.5/24 |  | 10.200.6/24 |             |                |
|  |  +-------------+  +-------------+             |                |
|  +-----------------------------------------------+                |
+------------------------------------------------------------------+
```

### 5.2 Network Security Controls

- **No default routes to any external network.** The classified zone has no gateway to the outside.
- **Internal firewalling:** All inter-VLAN traffic passes through firewall rules. Default deny.
- **Kubernetes network policies:** Enforced via Calico or Cilium with default-deny namespace policies.
- **DNS:** Internal-only DNS servers. No external resolution possible.
- **NTP:** Internal Stratum 1 NTP server (GPS-based clock within the facility or rubidium oscillator). No external NTP.
- **All management interfaces** (IPMI/iLO/iDRAC) on a dedicated, isolated VLAN with strict access controls.

### 5.3 Traffic Encryption

All intra-cluster traffic encrypted using mutual TLS (mTLS) via the service mesh or Kubernetes-native mechanisms. Certificates issued by an internal CA operated within the air-gapped environment. Encryption algorithms must be approved by MUST.

---

## 6. Kubernetes Platform Architecture

### 6.1 Distribution Selection

**Recommended: RKE2 (Rancher Government / DISA STIG-hardened)**

Rationale:
- RKE2 is specifically designed for air-gapped, security-sensitive deployments
- Ships with DISA STIG compliance profiles (closely aligned with MUST hardening requirements)
- Includes embedded etcd, container runtime (containerd), and hardened defaults
- Has a strong track record in defense/government deployments internationally
- Does not depend on any cloud provider APIs

**Alternative:** Vanilla upstream Kubernetes with manual CIS Benchmark hardening, but this requires significantly more engineering effort.

**Not recommended:** Managed Kubernetes (EKS, AKS, GKE) -- these require cloud connectivity and foreign-controlled infrastructure, which is incompatible with HEMLIG requirements.

### 6.2 Cluster Topology

```
+------------------------------------------+
|           CONTROL PLANE (HA)             |
|                                          |
|  +----------+ +----------+ +----------+ |
|  | cp-node1 | | cp-node2 | | cp-node3 | |
|  | etcd     | | etcd     | | etcd     | |
|  | api-srv  | | api-srv  | | api-srv  | |
|  | sched    | | sched    | | sched    | |
|  | ctrl-mgr | | ctrl-mgr | | ctrl-mgr | |
|  +----------+ +----------+ +----------+ |
+------------------------------------------+
          |
          | (internal LB: keepalived/HAProxy)
          |
+------------------------------------------+
|            WORKER NODE POOL              |
|                                          |
|  +--------+ +--------+ +--------+       |
|  |worker01| |worker02| |worker03|  ...  |
|  +--------+ +--------+ +--------+       |
|                                          |
|  Namespaces:                             |
|  - system (platform services)            |
|  - workload-<project> (per-project)      |
|  - monitoring                            |
|  - logging                               |
|  - registry                              |
|  - gitops                                |
+------------------------------------------+
```

### 6.3 Kubernetes Hardening

The following hardening measures align with CIS Kubernetes Benchmark and MUST IT-security requirements:

- **API Server:**
  - Authentication via client certificates and OIDC (internal IdP)
  - Authorization via RBAC (no ABAC, no AlwaysAllow)
  - Admission controllers: PodSecurity (Restricted profile), OPA/Gatekeeper or Kyverno for policy enforcement
  - Audit logging enabled for all API requests (write to persistent, tamper-evident storage)
  - `--anonymous-auth=false`
  - `--insecure-port` disabled
  - TLS 1.2+ only, with MUST-approved cipher suites

- **etcd:**
  - Encrypted at rest (AES-256-GCM via EncryptionConfiguration)
  - mTLS between etcd peers and from API server
  - Access restricted to control plane nodes only
  - Regular backups to encrypted offline storage

- **Kubelet:**
  - Authentication required (no anonymous access)
  - Authorization mode set to Webhook
  - Read-only port disabled
  - Protect kernel defaults enabled

- **Pod Security:**
  - PodSecurity admission enforced at Restricted level cluster-wide
  - No privileged containers except explicitly approved system components
  - No host networking, host PID, or host IPC by default
  - Seccomp profiles enforced (RuntimeDefault minimum)
  - AppArmor or SELinux mandatory on all worker nodes

- **Network Policies:**
  - Default deny all ingress and egress per namespace
  - Explicit allow rules for each required communication path
  - Enforced by CNI plugin (Calico or Cilium)

- **Secrets Management:**
  - Kubernetes Secrets encrypted at rest via etcd encryption
  - For sensitive workloads: HashiCorp Vault (air-gapped deployment) as external secrets store
  - Secrets injected via CSI Secrets Store Driver, not environment variables

### 6.4 Container Runtime Hardening

- **containerd** as the runtime (no Docker daemon)
- **Read-only root filesystem** for all application containers
- **No root containers** -- all workloads run as non-root UID
- **Resource limits** enforced on all pods (CPU, memory, ephemeral storage)
- **Image pull policy:** Always pull from internal registry; no external registries reachable

### 6.5 Service Mesh (Optional but Recommended)

Deploy Istio or Linkerd within the air-gap for:
- Automatic mTLS between all services
- Fine-grained traffic policies
- Observability (distributed tracing within the classified environment)
- Certificate rotation

---

## 7. Storage Architecture

### 7.1 Persistent Storage

**Recommended: Rook-Ceph (air-gapped deployment)**

- Provides block (RBD), filesystem (CephFS), and object (RGW) storage
- Runs as Kubernetes-native operator
- Supports encryption at rest (dm-crypt/LUKS on OSDs)
- Data replication factor of 3 across failure domains
- No external dependencies

**Alternative:** Longhorn (simpler, suitable for smaller deployments) or local-path-provisioner with manual replication.

### 7.2 Encryption at Rest

All persistent data must be encrypted at rest:

- **OS disks:** LUKS2 encryption with TPM-sealed keys
- **etcd:** Kubernetes EncryptionConfiguration with AES-256-GCM
- **Ceph OSDs:** dm-crypt with keys managed by Vault or Ceph's internal KMS
- **Encryption algorithms:** Must be approved by MUST; AES-256 is generally acceptable

### 7.3 Data Classification Labels

Storage classes should be labeled to ensure data placement awareness:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hemlig-replicated
  labels:
    classification: hemlig
    data-sovereignty: sweden
parameters:
  replication: "3"
  encrypted: "true"
reclaimPolicy: Delete   # Ensures PV cleanup; see data destruction section
```

---

## 8. Identity, Access, and Clearance Management

### 8.1 Identity Architecture

```
+-------------------------------------------+
|         INTERNAL IDENTITY PROVIDER        |
|         (Keycloak, air-gapped)            |
|                                           |
|  - OIDC provider for Kubernetes           |
|  - SAML/OIDC for web-based tools          |
|  - MFA enforcement (hardware tokens)      |
|  - Clearance-level attribute mapping      |
+-------------------------------------------+
         |              |              |
    Kubernetes     GitOps/Gitea    Monitoring
    API Server     (Harbor)        (Grafana)
```

### 8.2 Authentication Requirements

- **Multi-factor authentication (MFA):** Required for all access. Hardware security tokens (YubiKey FIPS or equivalent) for the second factor. No SMS or phone-based MFA (no phone connectivity in classified zone).
- **Certificate-based authentication:** For service-to-service and API access.
- **Session management:** Short-lived tokens (maximum 8 hours), forced re-authentication after idle timeout (15 minutes).

### 8.3 Authorization Model (RBAC)

```
Role Hierarchy (Kubernetes RBAC):
---------------------------------------------------------------------------
Role                    | Scope        | Clearance | Personnel
---------------------------------------------------------------------------
cluster-admin           | Cluster      | HEMLIG    | 2-3 senior engineers
platform-operator       | Cluster      | HEMLIG    | 4-5 platform engineers
namespace-admin         | Namespace    | HEMLIG    | Project leads
developer               | Namespace    | HEMLIG    | Cleared developers
read-only-auditor       | Cluster      | HEMLIG    | Security/compliance
---------------------------------------------------------------------------
```

All 12 cleared engineers are mapped to appropriate roles. No shared accounts. Named individual accounts only.

### 8.4 Clearance Verification Process

- Clearance status verified against Sapo records before account provisioning
- Periodic re-verification (minimum annually, or as required by Sakerhetsskyddsavtal)
- Immediate account suspension upon clearance revocation or expiration
- Clearance level stored as identity attribute but NOT exposed to application workloads

---

## 9. Cryptographic Architecture

### 9.1 MUST Approval Requirements

All cryptographic implementations must be approved by MUST for use at HEMLIG level. This includes:

- TLS cipher suites used for intra-cluster communication
- Encryption algorithms for data at rest
- Key exchange mechanisms
- Hash functions used for integrity verification
- Random number generation

**Engage MUST crypto evaluation early** -- this is often the longest lead-time item in accreditation.

### 9.2 Key Management

- **Internal CA:** Operated within the air-gap using a tool such as cfssl, step-ca, or EJBCA
- **Root CA:** Offline, stored in HSM (Hardware Security Module) -- MUST-approved HSM required
- **Intermediate CAs:** Online for automated certificate issuance, short-lived certificates (24-72 hours)
- **HSM:** SafeNet Luna or equivalent MUST-approved HSM for root key storage and critical signing operations
- **Key rotation:** Automated rotation schedules; documented in Kryptoplan

### 9.3 Certificate Lifecycle

```
Root CA (offline, HSM-backed, 10-year validity)
  |
  +-- Intermediate CA: Kubernetes (2-year, auto-renewed)
  |     +-- API server certs
  |     +-- kubelet certs
  |     +-- etcd peer/client certs
  |
  +-- Intermediate CA: Service Mesh (1-year, auto-renewed)
  |     +-- Workload identity certs (24h SVID)
  |
  +-- Intermediate CA: User Authentication (2-year)
        +-- Admin client certs
        +-- Service account tokens
```

---

## 10. Air-Gap Operations Model

### 10.1 Software Import Process (Sneakernet)

Since the environment is fully air-gapped, all software must be physically transported into the classified zone. This is the most operationally critical process.

```
UNCLASSIFIED SIDE                    CLASSIFIED SIDE

1. Download/build software    -->  4. Connect media to import station
2. Verify signatures/hashes         (write-once, isolated host)
3. Write to approved media    -->  5. Automated scan + integrity check
   (encrypted USB / optical)       6. Push to internal registry/repo
                                   7. Destroy or securely wipe media

   PHYSICAL TRANSFER
   (logged, two-person rule)
```

**Approved transfer media:**
- Write-once optical media (BD-R) preferred for high-assurance imports
- Encrypted USB drives (hardware-encrypted, FIPS 140-2/3) for larger transfers
- All media logged in transfer register with classification markings
- Two-person integrity rule: two cleared individuals verify the transfer

### 10.2 Import Station

A dedicated, hardened workstation in the classified zone used exclusively for importing software:

- Not connected to the production Kubernetes cluster network
- Runs antivirus/malware scanning (ClamAV or equivalent, with signatures imported separately)
- Performs cryptographic verification of all imported artifacts (GPG signatures, SHA-256 checksums verified against out-of-band published hashes)
- After verification, pushes to the internal container registry and artifact repository
- Audit log of all imports maintained

### 10.3 Internal Container Registry

**Recommended: Harbor (air-gapped deployment)**

- Image vulnerability scanning (Trivy, with vulnerability database imported periodically)
- Image signing and verification (cosign/Notary)
- RBAC integrated with internal IdP
- Replication not needed (single site), but garbage collection configured
- All base images curated and approved before import

### 10.4 Internal Git Repository

**Recommended: Gitea (lightweight, air-gapped)**

- Houses all infrastructure-as-code, Kubernetes manifests, Helm charts
- GitOps controller (Flux CD or Argo CD) reconciles desired state from Git
- Branch protection, code review requirements enforced
- All commits signed with individual developer keys

### 10.5 GitOps Workflow (Within Air-Gap)

```
Developer (cleared) --> Gitea (internal) --> Flux CD / Argo CD --> Kubernetes Cluster
     |                       |                      |
     |-- commits code        |-- PR review          |-- auto-reconcile
     |-- signs commit        |-- merge              |-- drift detection
                             |-- audit trail        |-- rollback capability
```

### 10.6 Patch and Update Cycle

- **OS patches:** Imported monthly (or as critical CVEs warrant), tested in staging namespace first
- **Kubernetes upgrades:** Quarterly, following upstream release + 30-day stabilization period
- **Container base images:** Rebuilt and imported monthly with latest patches
- **Vulnerability database:** Updated weekly via sneakernet import for Trivy/scanning tools

---

## 11. Software Supply Chain

### 11.1 Bill of Materials (SBOM)

Every container image and software component must have a documented SBOM:

- Generated using Syft or equivalent tooling
- Stored in Harbor alongside the image
- Reviewed during import approval process
- Available for MUST inspection during accreditation

### 11.2 Image Build Pipeline

Images are built on the unclassified side (or a dedicated build environment) and transferred:

```
Source Code --> Build (reproducible) --> SBOM Generation --> Sign -->
  Scan (unclass side) --> Transfer Media --> Import Station -->
    Rescan (classified side) --> Harbor Registry --> Approved for Deployment
```

### 11.3 Approved Base Images

Maintain a curated catalog of approved base images:

- Hardened Linux base (e.g., Chainguard images, or custom-built minimal images)
- Each base image has a documented security profile
- Unapproved base images blocked by admission controller policy

---

## 12. Monitoring, Logging, and Audit

### 12.1 Logging Architecture

```
+-------------------------------------------------------+
|                   LOG COLLECTION                       |
|                                                       |
|  Kubernetes audit logs  ----+                         |
|  System logs (journald) ----+--> Fluentd/Fluent Bit   |
|  Application logs       ----+      |                  |
|  Network flow logs      ----+      v                  |
|                            Elasticsearch / Loki       |
|                               (air-gapped)            |
|                                    |                  |
|                                    v                  |
|                               Grafana                 |
|                          (dashboards + alerts)        |
+-------------------------------------------------------+
```

### 12.2 Audit Requirements

- **Kubernetes API audit:** All requests logged at Metadata level minimum; RequestResponse for sensitive resources (Secrets, RBAC)
- **Node-level audit:** auditd on all nodes capturing file access, privilege escalation, authentication events
- **Tamper evidence:** Logs written to append-only storage; integrity verified via cryptographic chaining (log signing)
- **Retention:** Minimum 5 years (or as specified by Sakerhetsskyddsforordningen), stored within the classified environment
- **No log export:** Logs never leave the classified zone. Auditors (MUST, FMV) access logs on-site.

### 12.3 Monitoring Stack

- **Prometheus:** Metrics collection from all cluster components and workloads
- **Grafana:** Dashboards for operational and security monitoring
- **Alertmanager:** Alerts routed to on-call within the classified environment (no external notification channels)
- **Node Exporter:** Hardware and OS metrics
- **Falco:** Runtime security monitoring -- detects anomalous container behavior, syscall violations

### 12.4 Security Monitoring

- **Falco rules** tuned for defense-specific threats (data exfiltration attempts, unauthorized access patterns)
- **Network flow analysis** via Cilium Hubble or Calico flow logs
- **Failed authentication tracking** with automatic lockout after threshold
- **Drift detection:** GitOps controller alerts on any manual changes to cluster state

---

## 13. Backup, Recovery, and Data Destruction

### 13.1 Backup Strategy

- **etcd:** Automated snapshots every 6 hours, retained for 30 days, encrypted, stored on separate storage
- **Persistent volumes:** Daily snapshots via Ceph RBD snapshots or Velero
- **GitOps state:** Git repository is the source of truth; backed up to separate encrypted storage
- **Backup media:** Stored in a safe rated for HEMLIG within the facility

### 13.2 Disaster Recovery

- **RTO (Recovery Time Objective):** 4 hours for platform services; 8 hours for workloads
- **RPO (Recovery Point Objective):** 6 hours maximum data loss
- **Recovery procedure:** Documented, tested quarterly, executable by the platform team
- **No off-site backup** (unless a second accredited facility exists)

### 13.3 Data Destruction

When data must be destroyed (decommissioning, media failure, clearance changes):

- **Storage media:** Physical destruction per MUST/FM guidelines (degaussing for HDD; physical shredding for SSD/NVMe)
- **Kubernetes resources:** Secure deletion of PVCs triggers encrypted volume destruction; crypto-erase where supported
- **Media tracking:** All destruction events logged with witness signatures
- **Applies to all media** that has ever contained HEMLIG data, including backup media, transfer media, and failed drives

---

## 14. Staffing and Clearance Model

### 14.1 Team Structure (12 Cleared Engineers)

```
+-------------------------------------------------------+
|  PLATFORM TEAM (12 engineers, all HEMLIG cleared)     |
|                                                       |
|  Security Lead (1)                                    |
|    - Accreditation liaison with FMV/MUST              |
|    - Security policy and compliance                   |
|    - Incident response lead                           |
|                                                       |
|  Platform Engineering (5)                             |
|    - Kubernetes cluster operations                    |
|    - Infrastructure as code                           |
|    - Storage and networking                           |
|    - 2 of these serve as cluster-admin                |
|                                                       |
|  Application/DevOps Engineering (4)                   |
|    - CI/CD pipeline management (within air-gap)       |
|    - Developer support                                |
|    - Image curation and import                        |
|    - Workload onboarding                              |
|                                                       |
|  On-call rotation: 2-person coverage, 24/7            |
|    (drawn from Platform + App engineers)              |
|                                                       |
|  Security Operations (2)                              |
|    - Log review and security monitoring               |
|    - Vulnerability management                         |
|    - Audit support                                    |
+-------------------------------------------------------+
```

### 14.2 Roles for Non-Cleared Staff (28 engineers)

The remaining 28 engineers who are NOT cleared for HEMLIG can contribute to:

- Development and testing of application code on unclassified systems
- Building and testing container images in unclassified build environment
- Documentation (unclassified portions)
- Tooling and automation development (tested in unclassified lab, imported to classified)
- Training and knowledge development

**They must never have access** (physical or logical) to the classified environment.

### 14.3 Training Requirements

- All cleared personnel: Annual Sakerhetsskydd training
- Platform team: Kubernetes security training (CKS certification recommended)
- Security operations: Incident response drills (quarterly)
- All cleared personnel: Procedures for handling HEMLIG information and media

---

## 15. Accreditation Roadmap

### 15.1 Phase Overview

```
Phase 1: Preparation (Months 1-3)
  - Establish Sakerhetsskyddsavtal with FMV
  - Engage MUST for crypto evaluation (early!)
  - Facility assessment and TEMPEST survey
  - Sakerhetsanalys (security analysis) drafting
  - Hardware procurement initiated

Phase 2: Build (Months 4-8)
  - Facility buildout / modification
  - Hardware delivery and inspection
  - Platform installation and hardening
  - Import toolchain established (sneakernet process)
  - Systemsakerhetsplan drafted

Phase 3: Hardening and Testing (Months 9-11)
  - Penetration testing (by approved assessors)
  - TEMPEST testing
  - Vulnerability assessment
  - Operational procedures finalized
  - Driftsakerinstruktion completed

Phase 4: Accreditation (Months 12-15)
  - Documentation package submitted to FMV
  - MUST technical review
  - On-site inspection
  - Findings remediation
  - Accreditation decision (systemsakerhetsgodkannande)

Phase 5: Operations (Ongoing)
  - Continuous compliance monitoring
  - Annual re-assessment
  - Change management (all changes may require re-accreditation approval)
```

### 15.2 Key Dependencies and Long Lead-Time Items

| Item | Lead Time | Notes |
|---|---|---|
| Sakerhetsskyddsavtal | 2-4 months | Must be in place before classified work begins |
| MUST crypto approval | 3-6 months | Start immediately; this often gates the project |
| TEMPEST survey + remediation | 2-4 months | Facility may need shielding modifications |
| Hardware procurement | 2-3 months | Longer if supply chain attestation required |
| Personnel clearance upgrades | 3-12 months | If additional staff need HEMLIG clearance |
| FMV accreditation review | 3-6 months | After documentation submission |

---

## 16. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | MUST crypto approval delays | High | High | Engage MUST in month 1; prepare fallback cipher suites |
| R2 | Accreditation findings require major rework | Medium | High | Engage FMV early for informal reviews; use MUST ITSM as checklist |
| R3 | Air-gap operations slow development velocity | High | Medium | Invest heavily in automation; robust GitOps pipeline |
| R4 | Key person dependency (12 cleared staff) | High | High | Cross-train aggressively; document everything; initiate additional clearances |
| R5 | Hardware supply chain compromise | Low | Critical | Procure through trusted channels; inspect on delivery; use TPM attestation |
| R6 | TEMPEST compliance failure | Medium | High | Conduct early survey; budget for shielded room if needed |
| R7 | Software vulnerability in air-gapped environment | Medium | Medium | Regular patch imports; vulnerability scanning; defense in depth |
| R8 | Insider threat | Low | Critical | Two-person rule for admin; comprehensive audit logging; behavioral monitoring |
| R9 | Loss of accreditation on re-assessment | Low | High | Continuous compliance monitoring; maintain living documentation |
| R10 | Scalability -- 12 engineers insufficient for 24/7 ops | Medium | High | Automate aggressively; consider initiating clearances for 4-6 additional staff |

---

## 17. Technology Selection Summary

| Component | Selected Technology | Rationale |
|---|---|---|
| **Kubernetes Distribution** | RKE2 | Air-gap native, STIG-hardened, no cloud dependency |
| **Operating System** | Ubuntu 22.04 LTS (FIPS) or RHEL 9 (STIG) | Long-term support, security profiles available |
| **Container Runtime** | containerd (bundled with RKE2) | Industry standard, CIS-benchmarked |
| **CNI / Network Policy** | Cilium | eBPF-based, advanced network policy, Hubble observability |
| **Storage** | Rook-Ceph | Kubernetes-native, block/file/object, encryption support |
| **Container Registry** | Harbor | Air-gap support, vulnerability scanning, RBAC |
| **Git Repository** | Gitea | Lightweight, self-hosted, air-gap friendly |
| **GitOps Controller** | Flux CD | Lightweight, secure, pull-based reconciliation |
| **Secrets Management** | HashiCorp Vault (OSS) | Air-gap deployment, dynamic secrets, encryption as a service |
| **Identity Provider** | Keycloak | OIDC/SAML, MFA support, air-gap deployable |
| **Monitoring** | Prometheus + Grafana | Industry standard, no external dependencies |
| **Logging** | Fluent Bit + Loki (or Elasticsearch) | Efficient, Kubernetes-native |
| **Security Monitoring** | Falco | Runtime threat detection, syscall monitoring |
| **Policy Engine** | Kyverno or OPA/Gatekeeper | Admission control, policy as code |
| **Backup** | Velero + Ceph snapshots | Kubernetes-aware backup/restore |
| **Certificate Management** | cert-manager + step-ca | Automated cert lifecycle within air-gap |
| **HSM** | SafeNet Luna (or MUST-approved equivalent) | Root CA key protection |
| **Service Mesh** | Istio (optional) | mTLS, traffic policy, observability |

---

## Appendix A: Glossary of Swedish Terms

| Swedish Term | English Equivalent |
|---|---|
| Sakerhetsskyddslagen | Protective Security Act |
| Sakerhetsskyddsforordningen | Protective Security Ordinance |
| Sakerhetsskyddsavtal | Security Protection Agreement |
| Sakerhetsskyddat omrade | Security Protected Area |
| Systemsakerhetsgodkannande | System Security Accreditation |
| Sakerhetsanalys | Security Analysis |
| Systemsakerhetsplan | System Security Plan |
| Driftsakerinstruktion | Operational Security Instructions |
| Kryptoplan | Cryptographic Plan |
| Forsvarsmakten (FM) | Swedish Armed Forces |
| FMV | Defence Materiel Administration |
| MUST | Military Intelligence and Security Service |
| Sapo | Swedish Security Service |
| HEMLIG | Secret (classification level) |
| Forsvarsindustri | Defense Industry |

## Appendix B: Compliance Checklist (Abbreviated)

- [ ] Sakerhetsskyddsavtal signed with FMV
- [ ] Facility meets physical security requirements for HEMLIG
- [ ] TEMPEST survey completed and compliant
- [ ] All personnel with access hold valid HEMLIG clearance via Sapo
- [ ] MUST crypto approval obtained for all cryptographic components
- [ ] Sakerhetsanalys completed and approved
- [ ] Systemsakerhetsplan completed and submitted
- [ ] Driftsakerinstruktion completed and approved
- [ ] Kryptoplan completed and submitted to MUST
- [ ] Incidenthanteringsplan completed
- [ ] Air-gap validated (no network paths to unclassified systems)
- [ ] Kubernetes CIS Benchmark compliance verified
- [ ] All data encrypted at rest and in transit
- [ ] Audit logging operational with tamper-evident storage
- [ ] Penetration testing completed by approved assessors
- [ ] Backup and recovery procedures tested
- [ ] Data destruction procedures documented and tested
- [ ] On-call rotation established (24/7 coverage)
- [ ] Systemsakerhetsgodkannande (accreditation) obtained from FMV

---

*This document is intended as a technical architecture reference for internal planning and accreditation preparation. It should be reviewed by your organization's security officer (sakerhetsskyddschef) and updated based on specific guidance from FMV and MUST during the accreditation process.*
