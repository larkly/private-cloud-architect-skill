# Architecture Document: HEMLIG-Classified Private Cloud Platform

## Swedish Defense Contractor -- Air-Gapped Kubernetes Platform

**Classification:** HEMLIG (Secret) according to Sakerhetsskyddslagen (2018:585)
**Version:** 1.0
**Date:** 2026-03-20
**Status:** Draft for FMV/MUST Accreditation Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Legal Framework](#2-regulatory-and-legal-framework)
3. [Threat Model and Security Objectives](#3-threat-model-and-security-objectives)
4. [Physical Architecture](#4-physical-architecture)
5. [Network Architecture](#5-network-architecture)
6. [Kubernetes Platform Architecture](#6-kubernetes-platform-architecture)
7. [Storage Architecture](#7-storage-architecture)
8. [Identity, Access, and Clearance Management](#8-identity-access-and-clearance-management)
9. [Air-Gap Strategy and Supply Chain](#9-air-gap-strategy-and-supply-chain)
10. [Encryption and Key Management](#10-encryption-and-key-management)
11. [Logging, Auditing, and SIEM](#11-logging-auditing-and-siem)
12. [Disaster Recovery and Business Continuity](#12-disaster-recovery-and-business-continuity)
13. [Operational Model and Staffing](#13-operational-model-and-staffing)
14. [Accreditation Roadmap](#14-accreditation-roadmap)
15. [Risk Register](#15-risk-register)
16. [Appendices](#16-appendices)

---

## 1. Executive Summary

This document describes the architecture for a private, air-gapped cloud platform designed to process data classified as HEMLIG (Secret) under Swedish law. The platform is built on Kubernetes and is designed from the ground up for accreditation by Forsvarets materielverk (FMV) and Militara underrattelse- och sakerhetsinstitutet (MUST).

### Key Design Principles

- **Air-gapped by default:** No network path exists between the classified environment and any unclassified network or the public internet. All data transfer is controlled through a one-way data diode or manual media transfer procedures.
- **Data sovereignty:** All data, processing, metadata, backups, and key material remain within Swedish territory at all times, in facilities controlled by Swedish-cleared personnel.
- **Least privilege and need-to-know:** Access is restricted to the 12 engineers holding Sapo security clearances at the appropriate level. Role-based access control (RBAC) enforces need-to-know compartmentalization.
- **Defense in depth:** Multiple independent security layers ensure that no single failure compromises the system.
- **Auditability:** Every action is logged, tamper-evident, and available for review by FMV/MUST inspectors.

### Platform Summary

| Attribute | Value |
|---|---|
| Classification Level | HEMLIG (Secret) |
| Deployment Model | On-premises, air-gapped |
| Orchestration | Kubernetes (RKE2 / Rancher Government) |
| Compute Nodes | 12-16 bare-metal servers |
| Storage | Ceph (self-hosted, encrypted) |
| Location | Sweden, Sakerhetsklassad anlaggning |
| Cleared Personnel | 12 of 40 engineers |
| Target Accreditation | FMV/MUST IT-sakerhetsgranskning |

---

## 2. Regulatory and Legal Framework

### 2.1 Sakerhetsskyddslagen (2018:585)

The Security Protection Act (Sakerhetsskyddslagen) establishes the framework for protecting national security interests. For HEMLIG-classified information systems, the following obligations apply:

- **Sakerhetsskyddsanalys:** A formal security protection analysis must be completed before system design, identifying the information assets, threat actors, and protective measures required.
- **Informationssakerhet:** Technical and organizational measures to protect classified information in IT systems, covering confidentiality, integrity, and availability.
- **Fysisk sakerhet:** Physical protection of premises, equipment, and media.
- **Personsakerhet:** Personnel security through Sapo-administered security clearance (sakerhetsklarering) and background checks (registerkontroll).
- **Sakerhetsskyddsavtal:** Security protection agreements with any suppliers or subcontractors who may access the system or its components.

### 2.2 Sakerhetsskyddsforordningen (2021:955)

The implementing regulation specifies detailed requirements for classification levels, handling procedures, and reporting obligations.

### 2.3 Forsvarsmaktens Foreskrifter (FFS)

FMV and MUST issue specific technical directives for IT systems processing classified defense information. Key requirements include:

- System must undergo formal IT security evaluation (IT-sakerhetsgranskning)
- TEMPEST/EMSEC requirements for emission security
- Cryptographic requirements (Swedish-approved crypto only for HEMLIG)
- Physical installation requirements for server rooms

### 2.4 MUST CIS Security Directives

MUST publishes classified directives for Communication and Information Systems (CIS). These cover:

- System architecture requirements
- Network segmentation mandates
- Approved product lists (evaluated and approved equipment)
- Incident response and reporting procedures

### 2.5 EU and NATO Considerations

If the platform will process information shared under EU or NATO security agreements:

- EU Council Security Rules and associated CIS policies may apply
- NATO INFOSEC requirements (AC/322 series) if NATO-classified info is co-processed
- Cross-domain solutions must be approved for any information sharing

### 2.6 Additional Swedish Regulations

- **Lagen om elektronisk kommunikation (2022:482):** Applies to network infrastructure
- **Offentlighets- och sekretesslagen (2009:400):** Governs handling of classified government information
- **GDPR/Dataskyddsforordningen:** Applies to any personal data processed on the platform (e.g., user access logs containing personnel identities)

---

## 3. Threat Model and Security Objectives

### 3.1 Threat Actors

For a HEMLIG-classified system in the Swedish defense sector, the threat model must account for:

| Threat Actor | Capability | Motivation |
|---|---|---|
| State-sponsored APT (e.g., GRU, MSS) | Very High | Intelligence collection, sabotage |
| Insider Threat (cleared personnel) | High | Espionage, coercion, disgruntlement |
| Supply Chain Compromise | High | Pre-positioned access, backdoors |
| Organized Crime (proxy for state) | Medium | Financial, contracted espionage |
| Hacktivists | Low-Medium | Disruption, embarrassment |

### 3.2 Attack Vectors Addressed

1. **Network intrusion:** Eliminated by air-gap; residual risk from maintenance media
2. **Supply chain compromise:** Mitigated by verified supply chain, signed artifacts, approved product list
3. **Insider threat:** Mitigated by clearance, RBAC, multi-person integrity (MPI), behavioral monitoring
4. **TEMPEST/emanation:** Mitigated by EMSEC-rated facility and equipment
5. **Physical intrusion:** Mitigated by facility security (skalskydd, larmsystem, bevakning)
6. **Media exfiltration:** Mitigated by media control procedures, no USB ports, locked BIOS

### 3.3 Security Objectives (CIA+)

| Objective | Requirement |
|---|---|
| **Confidentiality** | HEMLIG-classified data must never be exposed to unauthorized persons, systems, or territories |
| **Integrity** | All data and system configurations must be protected against unauthorized modification; changes must be traceable |
| **Availability** | System availability target of 99.5% during operational hours; RTO of 4 hours, RPO of 1 hour |
| **Accountability** | All actions by all actors (human and machine) must be attributable and logged |
| **Non-repudiation** | Critical actions require multi-person authorization and cryptographic evidence |

---

## 4. Physical Architecture

### 4.1 Facility Requirements

The platform must be hosted in a facility classified for HEMLIG information processing. This requires:

**Skalskydd (Perimeter Protection):**
- Outer perimeter: Fence, access control, CCTV, motion detection
- Building shell: Reinforced walls, anti-tamper doors, alarmed windows
- Server room: Reinforced room within building, independent access control, man-trap entry
- Rack level: Lockable racks with individual access logging

**TEMPEST/EMSEC Zone:**
- The server room must meet SDIP-27 Level A or B (NATO SDIP-27) or equivalent Swedish MUST EMSEC requirements
- This may require RF-shielded room (Faraday cage), filtered power lines, and fiber-only external connections
- EMSEC zone boundary must be formally defined and inspected by MUST

**Environmental Controls:**
- Redundant cooling (N+1)
- Redundant power with UPS and diesel generator (minimum 72-hour fuel capacity)
- Fire suppression (inert gas, e.g., Novec 1230 or FM-200)
- Water leak detection
- Environmental monitoring (temperature, humidity, smoke, intrusion) feeding into the security monitoring system

### 4.2 Recommended Facility Layout

```
+------------------------------------------------------------------+
|                     OUTER PERIMETER (Fence/Wall)                  |
|  +------------------------------------------------------------+  |
|  |                 BUILDING (Reinforced shell)                 |  |
|  |  +------------------+  +--------------------------------+  |  |
|  |  | VISITOR/          |  | UNCLASSIFIED WORK AREA         |  |  |
|  |  | RECEPTION         |  | (28 non-cleared engineers)     |  |  |
|  |  | (No classified    |  |                                |  |  |
|  |  |  material)        |  | - Standard IT                  |  |  |
|  |  +------------------+  | - Internet access               |  |  |
|  |                        | - Development (unclassified)    |  |  |
|  |  +------------------+  +--------------------------------+  |  |
|  |  | SECURITY OFFICE   |                                      |  |
|  |  | (Sakerhetschef)   |  +--------------------------------+  |  |
|  |  | - Clearance mgmt  |  | CLASSIFIED WORK AREA            |  |  |
|  |  | - Incident resp.  |  | (12 cleared engineers only)    |  |  |
|  |  | - Audit logs      |  |                                |  |  |
|  |  +------------------+  | +----------------------------+  |  |  |
|  |                        | | SERVER ROOM (EMSEC Zone)   |  |  |  |
|  |                        | | - Man-trap entry            |  |  |  |
|  |                        | | - Dual-person access        |  |  |  |
|  |                        | | - Faraday shielding         |  |  |  |
|  |                        | | - 8 server racks            |  |  |  |
|  |                        | | - 2 network racks           |  |  |  |
|  |                        | | - 1 storage rack            |  |  |  |
|  |                        | | - 1 management rack         |  |  |  |
|  |                        | +----------------------------+  |  |  |
|  |                        |                                |  |  |
|  |                        | - Classified workstations      |  |  |
|  |                        | - Secure print/shred           |  |  |
|  |                        | - Media safe                   |  |  |
|  |                        +--------------------------------+  |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

### 4.3 Hardware Bill of Materials (Indicative)

| Component | Quantity | Specification | Purpose |
|---|---|---|---|
| Compute Nodes | 12 | 2x AMD EPYC 9004, 512GB ECC RAM, 2x 1TB NVMe (OS), dual 25GbE | Kubernetes worker nodes |
| Control Plane Nodes | 3 | 2x AMD EPYC 9004, 256GB ECC RAM, 2x 1TB NVMe, dual 25GbE | Kubernetes control plane |
| Storage Nodes | 4 | 2x AMD EPYC, 256GB RAM, 12x 8TB NVMe SSD, 2x 100GbE | Ceph distributed storage |
| Management Node | 1 | Standard server, 128GB RAM | Bastion, deployment, monitoring |
| Spine Switches | 2 | 100GbE spine, Swedish/EU-manufactured preferred | Core network fabric |
| Leaf Switches | 4 | 25GbE ToR switches | Rack-level networking |
| Management Switches | 2 | 1GbE, out-of-band management | IPMI/BMC network (isolated) |
| Data Diode | 1 | Hardware-enforced unidirectional device (e.g., Advenica SecuriCDS) | One-way data export |
| Hardware Security Module | 2 | FIPS 140-3 Level 3 or CC EAL4+ (e.g., Thales Luna, Utimaco) | Key management, crypto |
| KVM Console | 2 | Local crash carts | Emergency console access |
| Backup Appliance | 1 | LTO-9 tape library or dedicated NVMe backup array | Offline backup |

**Note on Advenica:** Advenica is a Swedish company specializing in data diodes and cross-domain solutions, with products evaluated for Swedish and EU classified systems. Their products (e.g., SecuriCDS Data Diode) are strong candidates for the unidirectional gateway.

---

## 5. Network Architecture

### 5.1 Air-Gap Enforcement

The air-gap is the single most critical architectural control. The classified network has **zero** IP-routable connections to any unclassified network.

**Air-gap boundaries:**
- No wireless interfaces (Wi-Fi, Bluetooth, NFC) exist on any equipment within the EMSEC zone
- No USB ports are physically accessible (epoxy-filled or BIOS-disabled)
- BMC/IPMI interfaces are on a physically separate, non-routed management network with no external connectivity
- The only sanctioned data path crossing the boundary is a hardware data diode (unidirectional, outbound-only for specific approved use cases) and controlled media transfer

### 5.2 Network Topology

```
                        AIR GAP BOUNDARY
                    ========================

 UNCLASSIFIED SIDE  |  DATA DIODE  |  CLASSIFIED SIDE
                    |  (one-way    |
                    |   outbound)  |
 +---------------+  |              |  +----------------------------------+
 | Log collector |<-|==============|<-| Audit log export (filtered)     |
 | (unclass)     |  |              |  +----------------------------------+
 +---------------+  |              |
                    ========================

 CLASSIFIED NETWORK (entirely within EMSEC zone):

 +-------------------------------------------------------------------+
 |                        SPINE LAYER (100GbE)                       |
 |    +------------------+              +------------------+          |
 |    | Spine Switch A   |              | Spine Switch B   |          |
 |    +--------+---------+              +--------+---------+          |
 |             |                                 |                    |
 |    +--------+---------+              +--------+---------+          |
 |    |     Leaf SW 1    |              |     Leaf SW 2    |          |
 |    | (Control Plane   |              | (Workers 1-6)    |          |
 |    |  + Mgmt)         |              |                  |          |
 |    +--+--+--+---------+              +--+--+--+---------+          |
 |       |  |  |                           |  |  |                    |
 |      CP1 CP2 CP3                      W1  W2 W3...                |
 |       |                                                            |
 |    +--+---------------+              +--+--+--+---------+          |
 |    |     Leaf SW 3    |              |     Leaf SW 4    |          |
 |    | (Workers 7-12)   |              | (Storage Nodes)  |          |
 |    +--+--+--+---------+              +--+--+--+---------+          |
 |       |  |  |                           |  |  |                    |
 |      W7 W8 W9...                      S1  S2 S3  S4               |
 |                                                                    |
 |  +-------------------------------------------------------------+  |
 |  | OUT-OF-BAND MANAGEMENT NETWORK (1GbE, physically separate)  |  |
 |  | - IPMI/BMC for all nodes                                     |  |
 |  | - Management switch A + B (redundant)                        |  |
 |  | - Accessible only from Management Node (dual-homed)          |  |
 |  +-------------------------------------------------------------+  |
 +-------------------------------------------------------------------+
```

### 5.3 Network Segmentation (VLANs / Subnets)

| VLAN | Subnet | Purpose | Access |
|---|---|---|---|
| 10 | 10.10.10.0/24 | Kubernetes Control Plane | CP nodes, admin workstations |
| 20 | 10.10.20.0/24 | Kubernetes Pod Network (Calico) | Pod-to-pod (policy-controlled) |
| 30 | 10.10.30.0/24 | Kubernetes Service Network | Service discovery |
| 40 | 10.10.40.0/24 | Ceph Storage (public) | Storage clients |
| 41 | 10.10.41.0/24 | Ceph Cluster (replication) | Ceph OSDs only |
| 50 | 10.10.50.0/24 | Management/IPMI | BMC, management node only |
| 60 | 10.10.60.0/24 | User Workstations | Cleared engineer workstations |
| 99 | 10.10.99.0/24 | Monitoring/Logging | Prometheus, ELK, SIEM |

### 5.4 Network Policies

- Default-deny ingress and egress on all Kubernetes namespaces
- Explicit NetworkPolicy objects for every permitted flow
- Calico or Cilium as CNI with full network policy enforcement and flow logging
- East-west traffic encryption (WireGuard via Cilium or mutual TLS via service mesh)
- All inter-node traffic on encrypted overlay

### 5.5 DNS and Service Discovery

- CoreDNS running within the cluster for internal service resolution
- No external DNS resolvers (air-gapped)
- All internal zones managed via CoreDNS ConfigMaps
- Split horizon not needed (no external network exists)

---

## 6. Kubernetes Platform Architecture

### 6.1 Distribution Selection: RKE2 (Rancher Government)

**Rationale:**
- RKE2 (also known as "RKE Government") is purpose-built for air-gapped, security-sensitive deployments
- FIPS 140-2/140-3 compliant crypto modules
- CIS Kubernetes Benchmark hardened by default
- DISA STIG compliance out of the box
- Designed for disconnected/air-gapped installation
- Embeds etcd (no external dependency)
- Supported by Rancher Government Solutions, which has experience with defense sector deployments in allied nations
- **Alternative considered:** Vanilla Kubernetes with kubeadm -- rejected due to higher operational overhead and lack of built-in hardening. OpenShift was considered but rejected due to Red Hat subscription dependency and complexity.

### 6.2 Cluster Topology

```
+-------------------------------------------------------------------+
|                    KUBERNETES CLUSTER                              |
|                                                                    |
|  CONTROL PLANE (HA - 3 nodes)                                    |
|  +-------------+  +-------------+  +-------------+                |
|  | cp-node-01  |  | cp-node-02  |  | cp-node-03  |                |
|  | - kube-api  |  | - kube-api  |  | - kube-api  |                |
|  | - etcd      |  | - etcd      |  | - etcd      |                |
|  | - scheduler |  | - scheduler |  | - scheduler |                |
|  | - ctrl-mgr  |  | - ctrl-mgr  |  | - ctrl-mgr  |                |
|  +-------------+  +-------------+  +-------------+                |
|         |                |                |                        |
|         +--------+-------+--------+-------+                        |
|                  |                |                                 |
|  WORKER POOL A: General Workloads (6 nodes)                       |
|  +----------+ +----------+ +----------+                            |
|  | work-01  | | work-02  | | work-03  |                            |
|  +----------+ +----------+ +----------+                            |
|  +----------+ +----------+ +----------+                            |
|  | work-04  | | work-05  | | work-06  |                            |
|  +----------+ +----------+ +----------+                            |
|                                                                    |
|  WORKER POOL B: Sensitive/Compartmented Workloads (4 nodes)       |
|  +----------+ +----------+ +----------+ +----------+               |
|  | work-07  | | work-08  | | work-09  | | work-10  |               |
|  +----------+ +----------+ +----------+ +----------+               |
|                                                                    |
|  WORKER POOL C: Infrastructure Services (2 nodes)                 |
|  +----------+ +----------+                                         |
|  | infra-01 | | infra-02 |                                         |
|  | - Logging| | - Monitor|                                         |
|  | - Registry| | - GitOps|                                         |
|  +----------+ +----------+                                         |
+-------------------------------------------------------------------+
```

### 6.3 Control Plane Hardening

- **etcd encryption at rest:** AES-256-GCM encryption of all Secrets in etcd using keys from the HSM
- **API server:**
  - TLS 1.3 only
  - Client certificate authentication required (no token-based auth)
  - Audit logging enabled at RequestResponse level
  - Admission controllers: PodSecurity (Restricted profile), OPA/Gatekeeper, ImagePolicyWebhook
  - RBAC with no default cluster-admin binding
  - API server audit log shipped to SIEM
- **kubelet:**
  - TLS bootstrap with auto-rotation
  - Read-only port disabled
  - Anonymous auth disabled
  - Protect kernel defaults enabled
- **Scheduler and Controller Manager:**
  - Bound to localhost
  - Profiling disabled

### 6.4 Worker Node Hardening

- **Operating System:** SUSE Linux Enterprise Micro (SLE Micro) or Rocky Linux 9 FIPS, hardened per CIS benchmark
- **Immutable infrastructure:** OS is read-only where possible; changes applied via re-provisioning only
- **Kernel hardening:**
  - SELinux enforcing mode
  - Seccomp profiles for all pods (RuntimeDefault minimum)
  - AppArmor profiles for infrastructure pods
  - Kernel parameters tuned per CIS benchmark (sysctl hardening)
- **Container runtime:** containerd with configured seccomp and AppArmor
- **No SSH:** Worker nodes do not run SSH daemons; access only via kubectl exec (audited) or physical console
- **Automatic patching:** Managed via GitOps-driven re-provisioning from air-gapped repository

### 6.5 Namespace Strategy

| Namespace | Purpose | Security Tier |
|---|---|---|
| `kube-system` | Core Kubernetes components | Platform |
| `rke2-system` | RKE2 system components | Platform |
| `cert-manager` | Certificate management | Platform |
| `monitoring` | Prometheus, Grafana, Alertmanager | Platform |
| `logging` | Elasticsearch, Fluentd/Fluent Bit, Kibana | Platform |
| `security` | OPA/Gatekeeper, Falco, Trivy | Platform |
| `registry` | Harbor container registry | Platform |
| `gitops` | Flux CD / ArgoCD | Platform |
| `vault` | HashiCorp Vault (HSM-backed) | Platform |
| `app-<project>` | Application workloads per project | Application |
| `data-<project>` | Data processing pipelines | Application |

### 6.6 Admission Control and Policy Enforcement

**OPA/Gatekeeper Policies (mandatory):**
- All images must come from the internal Harbor registry
- No privileged containers
- No host network/PID/IPC namespace access
- Resource limits required on all pods
- All pods must have security context with `runAsNonRoot: true`
- No `latest` tags; all images must use SHA256 digests
- Labels required: `classification`, `project`, `owner`, `data-handling`
- Persistent volumes must be encrypted

**Kyverno or Gatekeeper (choose one) for:**
- Automatic mutation of security contexts
- Image signature verification (Cosign/Sigstore)
- Network policy existence checks per namespace

### 6.7 Service Mesh (Optional but Recommended)

**Istio (Ambient Mesh) or Linkerd** for:
- Mutual TLS between all services (zero-trust networking within the cluster)
- Fine-grained traffic authorization policies
- Observability (distributed tracing, traffic metrics)
- Traffic encryption in transit without application changes

Recommendation: **Linkerd** for lower complexity and resource footprint, unless advanced traffic management is needed.

---

## 7. Storage Architecture

### 7.1 Ceph Distributed Storage

**Why Ceph:**
- Open-source, no vendor lock-in, no foreign cloud dependency
- Provides block (RBD), file (CephFS), and object (RGW) storage from one platform
- Self-healing and self-managing with configurable replication
- Well-supported Kubernetes integration via Rook operator
- Can enforce data placement rules to ensure all replicas stay within controlled nodes

### 7.2 Ceph Cluster Design

```
+-------------------------------------------------------------------+
|                     CEPH CLUSTER (via Rook)                       |
|                                                                    |
|  +---------------+  +---------------+  +---------------+           |
|  | Storage-01    |  | Storage-02    |  | Storage-03    |           |
|  | 12x 8TB NVMe |  | 12x 8TB NVMe |  | 12x 8TB NVMe |           |
|  | 12 OSDs       |  | 12 OSDs       |  | 12 OSDs       |           |
|  | MON + MGR     |  | MON + MGR     |  | MON + MGR     |           |
|  +---------------+  +---------------+  +---------------+           |
|                                                                    |
|  +---------------+                                                 |
|  | Storage-04    |  Total Raw: ~384 TB                            |
|  | 12x 8TB NVMe |  Usable (3x replication): ~128 TB              |
|  | 12 OSDs       |  Usable (EC 4+2): ~256 TB                     |
|  | MON           |                                                 |
|  +---------------+                                                 |
+-------------------------------------------------------------------+
```

### 7.3 Storage Classes

| Storage Class | Backend | Replication | Use Case |
|---|---|---|---|
| `ceph-block-replicated` | Ceph RBD, 3x replication | 3 copies | Databases, stateful apps, etcd backup |
| `ceph-block-ec` | Ceph RBD, EC 4+2 | Erasure coded | Large datasets, archival |
| `ceph-filesystem` | CephFS, 3x replication | 3 copies | Shared file access (ReadWriteMany) |
| `ceph-object` | Ceph RGW | 3 copies | S3-compatible object storage |

### 7.4 Encryption at Rest

- **Ceph OSD encryption:** All OSDs use LUKS2 (dm-crypt) with AES-256-XTS
- **Keys managed by:** HashiCorp Vault backed by HSM
- **Key rotation:** Automated quarterly rotation with Vault
- **Deleted data:** Crypto-erase via key destruction for decommissioned volumes

### 7.5 Backup Strategy

- **Velero** for Kubernetes resource and persistent volume backup
- **Backup destination:** Dedicated backup storage (LTO-9 tape library or isolated NVMe array)
- **Backup frequency:** Daily incremental, weekly full
- **Retention:** Per project policy, minimum 90 days, maximum per regulatory requirement
- **Backup encryption:** AES-256-GCM with keys from HSM, separate from primary storage keys
- **Backup media handling:** Classified media; stored in rated safe when not in tape library; destruction via degaussing and physical shredding when decommissioned

---

## 8. Identity, Access, and Clearance Management

### 8.1 Personnel Security Model

```
Total Engineering Staff: 40
|
+-- Cleared for HEMLIG: 12 engineers
|   |
|   +-- Platform Operators (SRE): 4 engineers
|   |   - Full cluster admin access
|   |   - Physical server room access
|   |   - On-call rotation
|   |
|   +-- Security Engineers: 2 engineers
|   |   - Audit log review
|   |   - Policy management
|   |   - Incident response
|   |   - No application data access
|   |
|   +-- Application Developers: 4 engineers
|   |   - Namespace-scoped access only
|   |   - Deploy via GitOps only (no direct kubectl)
|   |   - Read access to own project logs
|   |
|   +-- Data Engineers: 2 engineers
|       - Access to data processing namespaces
|       - No access to platform infrastructure
|
+-- Not cleared for HEMLIG: 28 engineers
    - Work on unclassified systems ONLY
    - NO access to classified work area, network, or data
    - May develop code that is later reviewed and imported
      into classified environment via controlled process
```

### 8.2 Authentication Stack

```
+-------------------------------------------------------------------+
|                    AUTHENTICATION FLOW                             |
|                                                                    |
|  [Smart Card / CAC]                                               |
|       |                                                            |
|       v                                                            |
|  [Workstation PAM] --> [FreeIPA / Red Hat IdM]                    |
|                              |                                     |
|                              v                                     |
|                        [LDAP / Kerberos]                          |
|                              |                                     |
|                    +---------+---------+                           |
|                    |                   |                            |
|                    v                   v                            |
|              [Kubernetes OIDC]   [Vault Auth]                     |
|              (via Dex/Keycloak)  (LDAP backend)                   |
|                    |                   |                            |
|                    v                   v                            |
|              [RBAC Policies]    [Secret Access]                   |
+-------------------------------------------------------------------+
```

**Components:**
- **FreeIPA (or Red Hat Identity Management):** On-premises directory and authentication server; manages users, groups, and certificate-based authentication
- **Keycloak or Dex:** OIDC provider for Kubernetes API server authentication
- **Smart card / PKI:** Physical token required for authentication (two-factor: something you have + something you know)
- **No passwords over the wire:** Certificate-based or Kerberos authentication only

### 8.3 Authorization (RBAC)

Kubernetes RBAC is mapped to FreeIPA groups:

| FreeIPA Group | K8s ClusterRole | Scope | Permissions |
|---|---|---|---|
| `platform-admins` | `cluster-admin` (custom, reduced) | Cluster-wide | Full platform management, no app data read |
| `security-ops` | `security-auditor` | Cluster-wide | Read all audit logs, policies; write policy changes |
| `app-dev-<project>` | `namespace-developer` | Single namespace | Deploy, read logs, debug pods (own namespace only) |
| `data-eng-<project>` | `namespace-data` | Data namespaces | Read/write data pipelines, no infra access |

**Multi-Person Integrity (MPI):**
- Destructive operations (node drain, namespace deletion, secret modification) require approval from two cleared personnel
- Implemented via GitOps pull request review requirements (two approvals from different persons)
- Break-glass procedure documented and requires post-incident review

### 8.4 Workstation Security

- **Thin clients or hardened workstations** in the classified work area
- **Full-disk encryption** (LUKS2) with pre-boot authentication
- **SELinux enforcing**
- **No removable media** (USB ports disabled in BIOS + epoxy)
- **Screen lock** after 5 minutes of inactivity
- **No personal devices** in classified work area (phones, smartwatches, etc. stored in lockers outside)

---

## 9. Air-Gap Strategy and Supply Chain

### 9.1 Software Supply Chain

This is one of the most operationally challenging aspects of the platform. All software must traverse the air-gap via a controlled, auditable process.

**Supply Chain Pipeline:**

```
UNCLASSIFIED SIDE                    |  AIR GAP  |  CLASSIFIED SIDE
                                     |           |
+-----------------------------+      |           |  +---------------------------+
| External Sources            |      |           |  | Internal Harbor Registry  |
| - RKE2 releases             |      |           |  | - All container images    |
| - Container images          |      |           |  | - Helm charts             |
| - OS packages (RPM/DEB)     |      |           |  | - OS repo mirror          |
| - Helm charts               |      |           |  +---------------------------+
| - Security advisories       |      |           |            ^
+-------------+---------------+      |           |            |
              |                      |           |            |
              v                      |           |            |
+-----------------------------+      |           |  +---------------------------+
| Staging Environment         |      |           |  | Import Station            |
| (Unclassified, isolated)    |      |           |  | (Classified side)         |
| - Download & verify GPG sigs|      |           |  | - Verify signatures       |
| - Vulnerability scan (Trivy)|      |  MEDIA    |  | - Secondary malware scan  |
| - SBOM generation           |------|  TRANSFER |->| - Load into Harbor        |
| - Compliance check          |      |  (USB/DVD |  | - Sign with internal key  |
| - Build from source where   |      |  write-   |  | - Update GitOps repo      |
|   possible                  |      |  once     |  +---------------------------+
| - Cosign signature          |      |  media)   |
+-----------------------------+      |           |
                                     |           |
```

### 9.2 Media Transfer Procedure

1. **Preparation (unclassified side):**
   - Software artifacts downloaded to isolated staging machine
   - GPG/Cosign signatures verified against known keys
   - Vulnerability scan with current CVE database
   - SBOM generated and reviewed
   - Artifacts written to write-once media (DVD-R or hardware-locked USB)
   - Transfer manifest created and signed

2. **Transfer (physical):**
   - Media placed in tamper-evident bag
   - Two-person escort from unclassified staging area to classified import station
   - Transfer logged in physical transfer register (bound, numbered pages)
   - Both persons sign the register entry

3. **Import (classified side):**
   - Media inserted into dedicated import station (hardened, isolated machine)
   - Automated verification: signature check, hash comparison against manifest
   - Secondary malware scan with internal scanner
   - If verification passes: artifacts pushed to internal Harbor registry
   - Media physically destroyed after successful import (shredder for optical, crypto-erase for USB)

4. **Approval:**
   - Import must be approved by both a platform operator and a security engineer
   - All imported artifacts logged with full provenance chain

### 9.3 Internal Container Registry (Harbor)

- **Harbor** deployed within the cluster on infrastructure nodes
- All images signed with Cosign using internal signing key (stored in HSM)
- Vulnerability scanning with Trivy (database updated via media transfer)
- Image retention policies enforced
- Replication disabled (air-gapped, single instance)
- Robot accounts for CI/CD, no human pull credentials

### 9.4 GitOps Repository (Internal)

- **Gitea** deployed within the cluster as the internal Git server
- All Kubernetes manifests, Helm values, and configuration stored as code
- **Flux CD** or **ArgoCD** watches the Gitea repository and reconciles cluster state
- All changes require merge request with two-person review
- Git history provides full audit trail of all configuration changes
- Repository itself backed up to encrypted offline media

### 9.5 OS and Package Management

- Internal RPM/DEB repository mirror on classified side
- Updated via media transfer (monthly, or as-needed for critical CVEs)
- Package signing keys verified against known fingerprints
- Kickstart/AutoYaST files version-controlled in Gitea

---

## 10. Encryption and Key Management

### 10.1 Cryptographic Requirements

For HEMLIG-classified data, cryptographic implementations must meet Swedish national requirements. MUST specifies approved algorithms and, in some cases, approved products.

**Note:** For HEMLIG, commercially available FIPS 140-3 or CC-evaluated crypto may be acceptable, but this must be confirmed with MUST during the accreditation process. For KVALIFICERAT HEMLIG, Swedish national crypto (e.g., from Tutus/Advenica) would likely be required.

### 10.2 Encryption Overview

| Data State | Method | Algorithm | Key Management |
|---|---|---|---|
| Data at rest (storage) | LUKS2 (dm-crypt) | AES-256-XTS | Vault + HSM |
| Data at rest (etcd) | Kubernetes encryption provider | AES-256-GCM | Vault + HSM |
| Data at rest (backups) | GPG or age | AES-256 | HSM-stored keys |
| Data in transit (cluster) | mTLS (service mesh) | TLS 1.3, ECDHE, AES-256-GCM | cert-manager + Vault PKI |
| Data in transit (node-to-node) | WireGuard (Cilium) or mTLS | ChaCha20-Poly1305 or AES-256-GCM | Automatic key rotation |
| Data in transit (Ceph) | Ceph messenger v2 encryption | AES-128-GCM or AES-256-GCM | Ceph internal + Vault |

### 10.3 Key Management Architecture

```
+-------------------------------------------------------------------+
|                     KEY MANAGEMENT                                 |
|                                                                    |
|  +---------------------+                                          |
|  | Hardware Security    |  - Root of trust                        |
|  | Module (HSM)         |  - Master key never leaves HSM          |
|  | (FIPS 140-3 L3)     |  - Dual-custody initialization          |
|  +----------+----------+                                          |
|             |                                                      |
|             v                                                      |
|  +---------------------+                                          |
|  | HashiCorp Vault      |  - Auto-unseal via HSM                  |
|  | (HA, 3 instances)    |  - Transit engine for app encryption    |
|  |                      |  - PKI engine for certificate issuance  |
|  |                      |  - KV engine for application secrets    |
|  +----------+----------+                                          |
|             |                                                      |
|     +-------+-------+-------+                                     |
|     |       |       |       |                                      |
|     v       v       v       v                                      |
|  [LUKS]  [etcd]  [TLS]  [Apps]                                   |
|  keys    keys    certs   secrets                                  |
+-------------------------------------------------------------------+
```

**HSM Initialization:**
- HSM initialized with dual-custody (two different cleared personnel each hold part of the admin credentials)
- Master key ceremony witnessed and documented
- HSM firmware verified against manufacturer signatures
- HSM physically secured in server room rack with separate lock

**Vault Configuration:**
- Vault auto-unseals using HSM (no Shamir key shares on disk)
- Vault audit log captures every secret access
- Vault policies restrict access per Kubernetes service account and namespace
- Secrets injected into pods via Vault Agent Injector or CSI driver (secrets never stored in Kubernetes Secrets objects in plaintext)

### 10.4 Certificate Management

- **cert-manager** deployed in cluster for automated certificate lifecycle
- Internal CA chain: HSM Root CA -> Vault Intermediate CA -> cert-manager issuer
- Certificate rotation: 90-day maximum lifetime for workload certificates, automatic renewal
- mTLS enforced between all services via service mesh

---

## 11. Logging, Auditing, and SIEM

### 11.1 Logging Architecture

Comprehensive logging is essential for accreditation and incident response.

```
+-------------------------------------------------------------------+
|                     LOGGING PIPELINE                               |
|                                                                    |
|  Sources:                                                          |
|  +------------------+  +------------------+  +------------------+  |
|  | Kubernetes Audit |  | Application Logs |  | System Logs      |  |
|  | (API server)     |  | (stdout/stderr)  |  | (journald)       |  |
|  +--------+---------+  +--------+---------+  +--------+---------+  |
|           |                     |                     |            |
|           v                     v                     v            |
|  +-----------------------------------------------------------+    |
|  | Fluent Bit (DaemonSet on every node)                       |    |
|  | - Collects, parses, enriches                               |    |
|  | - Adds node, pod, namespace, user metadata                 |    |
|  +----------------------------+------------------------------+    |
|                               |                                    |
|                               v                                    |
|  +-----------------------------------------------------------+    |
|  | Elasticsearch / OpenSearch (3-node cluster)                |    |
|  | - Encrypted indices (encryption at rest via Ceph)          |    |
|  | - Index lifecycle management (ILM)                         |    |
|  | - Retention: 2 years online, 7 years archived              |    |
|  +----------------------------+------------------------------+    |
|                               |                                    |
|              +----------------+----------------+                   |
|              |                                 |                    |
|              v                                 v                    |
|  +---------------------+          +------------------------+      |
|  | Kibana / OpenSearch  |          | SIEM / Correlation     |      |
|  | Dashboards           |          | (Wazuh or Elastic      |      |
|  | (Visualization)      |          |  Security)             |      |
|  +---------------------+          +------------------------+      |
|                                            |                       |
|                                            v                       |
|                                   +------------------+             |
|                                   | Alert Manager    |             |
|                                   | (Prometheus AM)  |             |
|                                   | -> PagerDuty     |             |
|                                   |    equivalent    |             |
|                                   |    (internal)    |             |
|                                   +------------------+             |
+-------------------------------------------------------------------+
```

### 11.2 What Must Be Logged

| Event Category | Examples | Retention |
|---|---|---|
| Authentication | Login success/failure, certificate authentication, smart card events | 7 years |
| Authorization | RBAC decisions, denied access attempts, privilege escalation | 7 years |
| Kubernetes API | All API calls (RequestResponse level) | 2 years online, 7 years archive |
| Pod Lifecycle | Create, delete, exec into pod, port-forward | 2 years |
| Network | Firewall/NetworkPolicy drops, DNS queries, connection metadata | 1 year |
| Storage | Volume create/delete/attach, snapshot operations | 2 years |
| Vault | Secret read/write/delete, policy changes, token creation | 7 years |
| System | Boot, shutdown, kernel messages, SELinux denials | 2 years |
| Physical | Server room access, rack open/close, media transfer | 7 years |
| Application | Application-defined audit events | Per project policy |

### 11.3 Tamper-Evidence

- Audit logs are append-only (immutable index in Elasticsearch)
- Critical audit logs (authentication, authorization, Vault access) are additionally written to WORM storage or hashed and the hash chain exported via data diode to unclassified log receiver
- Log integrity verified daily via automated hash chain validation
- Any detected tampering triggers immediate security incident

### 11.4 Security Monitoring and Alerting

**Falco** deployed as DaemonSet for runtime threat detection:
- Detects unexpected process execution in containers
- Detects unexpected network connections
- Detects file access in sensitive paths
- Detects privilege escalation attempts
- Custom rules for defense-specific threat patterns

**Alert Escalation:**
1. **P1 (Immediate):** Potential compromise indicators -> Security engineer paged immediately, incident response initiated
2. **P2 (Urgent):** Policy violations, unusual access patterns -> Security team notified within 15 minutes
3. **P3 (Normal):** Configuration drift, non-critical anomalies -> Reviewed in daily security review
4. **P4 (Informational):** Audit trail entries -> Available for review

### 11.5 Export via Data Diode

Selected, filtered, and sanitized audit data may be exported through the hardware data diode for:
- Integration with enterprise SIEM on unclassified side
- Compliance reporting
- Aggregated operational metrics (no classified content)

**The data diode ensures:**
- Data flows only outbound from classified to unclassified
- No return channel exists (hardware-enforced)
- Content filtering applied before diode (strip any classified data)

---

## 12. Disaster Recovery and Business Continuity

### 12.1 Recovery Objectives

| Metric | Target | Notes |
|---|---|---|
| RTO (Recovery Time Objective) | 4 hours | Time to restore service after total site failure |
| RPO (Recovery Point Objective) | 1 hour | Maximum data loss in time |
| MTTR (Mean Time To Repair) | 2 hours | Average repair time for component failure |
| Availability Target | 99.5% | ~44 hours downtime per year (planned + unplanned) |

### 12.2 Failure Scenarios and Mitigations

| Scenario | Mitigation | Recovery Procedure |
|---|---|---|
| Single node failure | Kubernetes reschedules pods; Ceph rebalances data | Automatic; replace hardware during maintenance window |
| Control plane node failure | 3-node HA; survives loss of 1 node | Automatic failover; rebuild failed node from GitOps |
| Storage node failure | Ceph 3x replication survives 1 node loss | Automatic rebalancing; replace hardware |
| Dual storage node failure | Ceph 3x replication on 4 nodes can survive with reduced redundancy | Emergency: restrict writes, replace hardware urgently |
| Network switch failure | Redundant spine-leaf; dual-homed nodes | Automatic failover via LAG/ECMP |
| Power failure | UPS + diesel generator (72h) | Automatic switchover; fuel resupply procedure |
| Cooling failure | Redundant cooling; thermal shutdown procedure | Graceful workload migration; emergency maintenance |
| Total site loss | Encrypted offline backups stored at secondary cleared site | Rebuild cluster at secondary site; restore from backup |
| Ransomware/malware | Air-gap limits attack surface; immutable backups; read-only OS | Restore from known-good backup; incident investigation |

### 12.3 Backup and Restore

- **Cluster state:** GitOps repository contains complete desired state; cluster can be rebuilt from scratch
- **Persistent data:** Velero snapshots to offline storage; tested quarterly
- **Secrets/keys:** HSM backup to secondary HSM at cleared secondary site; Vault snapshots encrypted and stored offline
- **Restore testing:** Full restore test performed quarterly; results documented for accreditation

### 12.4 Secondary Site (Optional)

If operational requirements demand higher availability:
- Secondary cleared facility in different Swedish location
- Warm standby cluster with replicated data (async replication via encrypted media transport or dedicated fiber if approved by MUST)
- Failover procedure documented and tested semi-annually
- Both sites must meet same physical and EMSEC requirements

---

## 13. Operational Model and Staffing

### 13.1 Team Structure

```
+-------------------------------------------------------------------+
|                     ORGANIZATIONAL STRUCTURE                       |
|                                                                    |
|  Sakerhetschef (Security Officer)                                 |
|  - Reports to CEO/Board                                           |
|  - Responsible for Sakerhetsskyddsanalysen                        |
|  - Interface with FMV/MUST/Sapo                                   |
|  - Must be Swedish citizen with appropriate clearance              |
|                                                                    |
|  +-- Platform Operations Team (4 cleared engineers)               |
|  |   - Cluster lifecycle management                                |
|  |   - Infrastructure provisioning                                 |
|  |   - On-call rotation (2-person, 12h shifts during incidents)   |
|  |   - Media transfer procedures                                   |
|  |   - Capacity planning                                           |
|  |                                                                  |
|  +-- Security Operations Team (2 cleared engineers)               |
|  |   - Continuous monitoring and log review                        |
|  |   - Policy enforcement and audit                                |
|  |   - Vulnerability management                                    |
|  |   - Incident response                                           |
|  |   - Accreditation maintenance                                   |
|  |                                                                  |
|  +-- Application Development Team (4 cleared engineers)           |
|  |   - Develop and deploy classified applications                  |
|  |   - Operate within assigned namespaces                          |
|  |   - Follow secure development lifecycle                         |
|  |                                                                  |
|  +-- Data Engineering Team (2 cleared engineers)                  |
|      - Data pipeline development                                   |
|      - Data quality and governance                                 |
|      - Operate within data namespaces                              |
|                                                                    |
|  UNCLASSIFIED SUPPORT (28 non-cleared engineers)                  |
|  - Develop code on unclassified side                               |
|  - Code review and testing in unclassified staging                 |
|  - Documentation and training material                             |
|  - Upstream open-source contributions                              |
|  - Hardware procurement coordination                               |
+-------------------------------------------------------------------+
```

### 13.2 Operational Procedures

**Daily:**
- Security engineer reviews overnight alerts and audit logs
- Platform operator checks cluster health dashboard
- Automated integrity checks run and results reviewed

**Weekly:**
- Security operations review meeting (cleared personnel only)
- Vulnerability scan review and patch planning
- Capacity utilization review

**Monthly:**
- Media transfer for software updates (or as-needed for critical patches)
- Access review: verify all accounts match current clearance status
- Backup verification (automated restore test of random sample)

**Quarterly:**
- Full disaster recovery test
- Penetration test / red team exercise (by cleared security team or FMV-approved contractor)
- Security protection analysis review (sakerhetsskyddsanalys update)
- Hardware inventory reconciliation

**Annually:**
- Full accreditation review with FMV/MUST
- Personnel clearance re-verification
- Comprehensive risk assessment update
- Training and exercise for all cleared personnel

### 13.3 Change Management

All changes to the classified environment follow a strict process:

1. **Request:** Change requested via internal ticketing system (on classified side)
2. **Review:** Change reviewed by at least one platform operator and one security engineer
3. **Approve:** Two-person approval required for all changes
4. **Implement:** Change applied via GitOps merge (never manual kubectl)
5. **Verify:** Automated tests confirm expected state; monitoring confirms no anomalies
6. **Document:** Change recorded in change log with full details

**Emergency changes:** Break-glass procedure allows single-person execution with mandatory post-incident review within 24 hours.

### 13.4 Cleared Personnel Sustainability

With only 12 cleared engineers, staff burnout and single-points-of-knowledge are real risks:

- **Cross-training:** Every function must be performable by at least 2 people
- **Documentation:** All procedures documented in classified runbooks (stored in Gitea)
- **On-call:** Reasonable rotation schedule (no more than 1 week in 4)
- **Clearance pipeline:** Actively sponsor additional engineers through the Sapo clearance process (plan for 4-6 additional clearances over 12-18 months)
- **Knowledge transfer:** Structured sessions between cleared and unclassified teams for non-sensitive technical knowledge

---

## 14. Accreditation Roadmap

### 14.1 FMV/MUST Accreditation Process

The accreditation (IT-sakerhetsgranskning) is the formal approval to process HEMLIG information on the platform.

**Phase 1: Preparation (Months 1-3)**
- [ ] Complete sakerhetsskyddsanalys for the IT system
- [ ] Engage FMV/MUST early for pre-consultation (samrad)
- [ ] Define system boundary (what is in scope for accreditation)
- [ ] Identify applicable requirements from MUST CIS directives
- [ ] Select and engage MUST-approved TEMPEST/EMSEC assessor
- [ ] Begin facility preparation (EMSEC zone construction if needed)

**Phase 2: Design and Documentation (Months 3-6)**
- [ ] Complete this architecture document and get preliminary FMV/MUST feedback
- [ ] Develop Security Target document (Sakerhetsmalsattning)
- [ ] Develop Operating Procedures document (Driftsakerhetsinstruktion)
- [ ] Develop Incident Response Plan
- [ ] Complete TEMPEST/EMSEC zone design review
- [ ] Hardware procurement (allow for long lead times on EMSEC-rated equipment)

**Phase 3: Build and Harden (Months 6-10)**
- [ ] Install and configure physical infrastructure
- [ ] EMSEC zone construction and initial assessment
- [ ] Deploy Kubernetes platform (air-gapped installation)
- [ ] Implement all security controls documented in architecture
- [ ] Configure logging, monitoring, and SIEM
- [ ] Populate internal registries and repositories
- [ ] Begin staff training on classified procedures

**Phase 4: Testing and Verification (Months 10-13)**
- [ ] Internal security testing (configuration audit, vulnerability scan)
- [ ] Penetration testing by FMV-approved assessor
- [ ] TEMPEST/EMSEC measurement and certification
- [ ] Disaster recovery test
- [ ] Full audit trail verification
- [ ] Document all test results and remediate findings

**Phase 5: Accreditation Review (Months 13-15)**
- [ ] Submit complete documentation package to FMV/MUST
- [ ] FMV/MUST on-site inspection
- [ ] Address any findings or conditions
- [ ] Receive accreditation decision (godkannande)
- [ ] Implement any conditions attached to accreditation

**Phase 6: Operational Accreditation Maintenance (Ongoing)**
- [ ] Continuous compliance monitoring
- [ ] Annual re-assessment with FMV/MUST
- [ ] Change management with accreditation impact assessment
- [ ] Incident reporting to FMV/MUST as required

### 14.2 Key Accreditation Documents

| Document | Swedish Term | Purpose |
|---|---|---|
| Security Protection Analysis | Sakerhetsskyddsanalys | Risk and threat analysis |
| Security Target | Sakerhetsmalsattning | Security objectives and requirements |
| System Security Plan | Systemsakerhetsplan | Technical security architecture (this document) |
| Operating Security Instructions | Driftsakerhetsinstruktion | Day-to-day operational procedures |
| Incident Response Plan | Incidenthanteringsplan | Procedures for security incidents |
| TEMPEST/EMSEC Report | TEMPEST/EMSEC-rapport | Emission security assessment results |
| Penetration Test Report | Penetrationstestrapport | Results of security testing |
| Risk Acceptance Document | Riskacceptans | Formal acceptance of residual risks by system owner |

---

## 15. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| R01 | Supply chain compromise in container images or OS packages | Medium | Critical | Multi-stage verification, SBOM analysis, source builds where feasible | Low-Medium |
| R02 | Insider threat (cleared employee) | Low | Critical | MPI, behavioral monitoring, RBAC, audit logging, least privilege | Low |
| R03 | Key personnel departure (12-person team) | Medium | High | Cross-training, documentation, active clearance pipeline | Medium |
| R04 | Zero-day vulnerability in Kubernetes or OS | Medium | High | Defense in depth, network policies, runtime protection (Falco), rapid patching via media transfer | Medium |
| R05 | EMSEC emanation leak | Low | Critical | EMSEC-rated facility, annual TEMPEST assessment | Low |
| R06 | HSM failure | Low | High | Redundant HSM pair, offline backup of HSM state | Low |
| R07 | Extended power outage (>72h) | Low | High | 72h diesel, fuel resupply agreement, graceful shutdown procedure | Low-Medium |
| R08 | Accreditation delays due to FMV/MUST findings | Medium | Medium | Early engagement, experienced consultant, iterative reviews | Medium |
| R09 | Hardware failure causing data loss | Low | High | 3x replication, regular backups, tested restore | Low |
| R10 | Unauthorized physical access to server room | Very Low | Critical | Multi-layer physical security, dual-person access, 24/7 monitoring | Very Low |
| R11 | Media transfer introduces malware | Low | High | Multi-stage scanning, signature verification, isolated import station | Low |
| R12 | Ceph cluster split-brain or data corruption | Very Low | High | 4-node cluster, proper failure domain configuration, regular scrubbing | Very Low |

---

## 16. Appendices

### Appendix A: Technology Stack Summary

| Layer | Technology | Version (indicative) | License |
|---|---|---|---|
| Bare Metal OS | SUSE SLE Micro / Rocky Linux 9 | Latest stable | Commercial / Open Source |
| Container Runtime | containerd | 1.7+ | Apache 2.0 |
| Kubernetes | RKE2 | 1.29+ | Apache 2.0 |
| CNI | Cilium or Calico | Latest stable | Apache 2.0 |
| Service Mesh | Linkerd | Latest stable | Apache 2.0 |
| Storage | Ceph (via Rook) | Reef+ | LGPL |
| Container Registry | Harbor | 2.x | Apache 2.0 |
| GitOps | Flux CD or ArgoCD | Latest stable | Apache 2.0 |
| Git Server | Gitea | Latest stable | MIT |
| Secret Management | HashiCorp Vault | Enterprise (HSM support) | BSL / Commercial |
| Certificate Management | cert-manager | Latest stable | Apache 2.0 |
| Policy Engine | OPA/Gatekeeper or Kyverno | Latest stable | Apache 2.0 |
| Monitoring | Prometheus + Grafana | Latest stable | Apache 2.0 / AGPL |
| Logging | Fluent Bit + OpenSearch | Latest stable | Apache 2.0 |
| Runtime Security | Falco | Latest stable | Apache 2.0 |
| Vulnerability Scanning | Trivy | Latest stable | Apache 2.0 |
| Identity | FreeIPA + Keycloak | Latest stable | GPL / Apache 2.0 |
| Backup | Velero | Latest stable | Apache 2.0 |
| Data Diode | Advenica SecuriCDS | Current | Commercial |
| HSM | Thales Luna Network HSM 7 | Current | Commercial |

### Appendix B: Network Port Matrix

| Source | Destination | Port | Protocol | Purpose |
|---|---|---|---|---|
| API clients | kube-apiserver | 6443 | TCP/TLS | Kubernetes API |
| kubelet | kube-apiserver | 6443 | TCP/TLS | Node registration, status |
| kube-apiserver | etcd | 2379-2380 | TCP/TLS | etcd client + peer |
| kube-apiserver | kubelet | 10250 | TCP/TLS | Exec, logs, port-forward |
| Pods | CoreDNS | 53 | UDP/TCP | DNS resolution |
| All nodes | Cilium | 4240 | TCP | Cilium health |
| All nodes | Cilium VXLAN | 8472 | UDP | Overlay network |
| All nodes | Ceph MON | 6789,3300 | TCP | Ceph monitor |
| Ceph OSD | Ceph OSD | 6800-7300 | TCP | Ceph OSD cluster |
| All nodes | Vault | 8200 | TCP/TLS | Secret access |
| All nodes | Harbor | 443 | TCP/TLS | Image pull |
| Fluent Bit | OpenSearch | 9200 | TCP/TLS | Log ingestion |
| Prometheus | All nodes | 9100 | TCP | Node exporter metrics |
| Prometheus | All pods | various | TCP | Pod metrics scrape |

### Appendix C: Compliance Mapping

| Requirement Category | Sakerhetsskyddslagen Reference | Implementation |
|---|---|---|
| Information Classification | 2 kap. 1-5 SS | Classification labels on all K8s resources; data handling labels enforced by policy |
| Access Control | 2 kap. 6 SS, Personsakerhet | Sapo clearance + RBAC + MPI + smart card authentication |
| Physical Security | 3 kap. SS | EMSEC zone, multi-layer perimeter, 24/7 monitoring |
| Cryptographic Protection | MUST directives | HSM-backed key management, AES-256 at rest and in transit |
| Audit and Logging | 2 kap. 6 SS | Comprehensive logging, tamper-evident, 7-year retention |
| Incident Management | 6 kap. SS | Incident response plan, FMV/MUST reporting procedure |
| Supply Chain Security | 4 kap. SS (Sakerhetsskyddsavtal) | Verified supply chain, SBOM, multi-stage scanning |
| Data Sovereignty | 2 kap. SS | All data in Sweden, air-gapped, no foreign cloud |
| Personnel Security | 3 kap. SS | Sapo clearance for all personnel with access |
| Continuity | General requirement | DR plan, backup strategy, tested quarterly |

### Appendix D: Glossary

| Term | Definition |
|---|---|
| HEMLIG | Swedish classification level equivalent to NATO SECRET |
| Sakerhetsskyddslagen | The Swedish Security Protection Act (2018:585) |
| FMV | Forsvarets materielverk (Swedish Defence Materiel Administration) |
| MUST | Militara underrattelse- och sakerhetstjansten (Swedish Military Intelligence and Security Service) |
| Sapo | Sakerhetspolisen (Swedish Security Service) |
| EMSEC | Emission Security (protection against compromising emanations) |
| TEMPEST | NATO codename for EMSEC standards and testing |
| MPI | Multi-Person Integrity (requiring multiple persons for sensitive operations) |
| CIS | Communication and Information Systems |
| SDIP-27 | NATO standard for EMSEC zone classification |
| Skalskydd | Perimeter/shell protection (physical security) |
| Sakerhetsskyddsavtal | Security protection agreement (with suppliers) |
| Sakerhetsklarering | Security clearance |
| Registerkontroll | Background check via security service registers |
| HSM | Hardware Security Module |
| SBOM | Software Bill of Materials |
| CNI | Container Network Interface (Kubernetes networking plugin) |
| RKE2 | Rancher Kubernetes Engine 2 (also "RKE Government") |

---

## Document Control

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-03-20 | Architecture Team | Initial draft |

**Classification of this document:** This architecture document itself should be classified according to the organization's information classification policy. The detailed network topology, port matrix, and security control descriptions likely warrant classification at HEMLIG level once populated with site-specific details.

---

*End of document.*
