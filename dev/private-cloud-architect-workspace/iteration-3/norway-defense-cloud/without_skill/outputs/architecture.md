# Architecture Document: HEMMELIG-Classified Private Cloud Platform for Forsvaret

**Document Classification: BEGRENSET (Restricted) -- Architecture Reference Only**
**Version:** 1.0
**Date:** 2026-03-20
**Prepared for:** Norwegian Defence Company -- Forsvaret Contract
**Compliance Frameworks:** Sikkerhetsloven (Security Act), NSM Accreditation, NATO Security Policy

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Compliance Framework](#2-regulatory-and-compliance-framework)
3. [Threat Model and Security Posture](#3-threat-model-and-security-posture)
4. [Platform Architecture Decision: OpenStack vs Kubernetes](#4-platform-architecture-decision-openstack-vs-kubernetes)
5. [Network Architecture and Air-Gap Design](#5-network-architecture-and-air-gap-design)
6. [Compute, Storage, and Infrastructure](#6-compute-storage-and-infrastructure)
7. [Identity, Access Management, and Personnel Security](#7-identity-access-management-and-personnel-security)
8. [NATO Interoperability Layer](#8-nato-interoperability-layer)
9. [Data Sovereignty and Residency](#9-data-sovereignty-and-residency)
10. [Logging, Auditing, and Continuous Monitoring](#10-logging-auditing-and-continuous-monitoring)
11. [Operational Model and Staffing](#11-operational-model-and-staffing)
12. [Accreditation Strategy (NSM)](#12-accreditation-strategy-nsm)
13. [Physical Security](#13-physical-security)
14. [Supply Chain Security](#14-supply-chain-security)
15. [Disaster Recovery and Continuity](#15-disaster-recovery-and-continuity)
16. [Implementation Roadmap](#16-implementation-roadmap)
17. [Risk Register](#17-risk-register)
18. [Appendices](#18-appendices)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform capable of processing data classified at the HEMMELIG (Secret) level under Norwegian law. The platform is contracted by Forsvaret (Norwegian Armed Forces) and must be accredited by Nasjonal sikkerhetsmyndighet (NSM -- Norwegian National Security Authority).

### Key Constraints

| Constraint | Requirement |
|---|---|
| Classification Level | HEMMELIG (NATO SECRET equivalent) |
| Legal Framework | Sikkerhetsloven (Lov om nasjonal sikkerhet, 2018) |
| Accreditation Authority | NSM (Nasjonal sikkerhetsmyndighet) |
| Air-Gap | Full air-gap; no connectivity to public internet or lower-classification networks |
| Data Residency | All data must remain within Norwegian sovereign territory |
| NATO Interoperability | Must support controlled information sharing with NATO partners |
| Team Size | 20 total personnel; 8 hold HEMMELIG klarering (security clearance) |

### Recommended Approach

We recommend a **hybrid architecture** using **OpenStack as the IaaS foundation** with **Kubernetes deployed as a tenant workload layer on top**, providing both traditional VM-based workloads and containerized microservices. This approach maximizes flexibility while maintaining the strict isolation boundaries required at the HEMMELIG level.

---

## 2. Regulatory and Compliance Framework

### 2.1 Sikkerhetsloven (Norwegian Security Act)

The Sikkerhetsloven of 2018 (with supporting regulations in Virksomhetsikkerhetsforskriften, Klareringsforskriften, and Sikkerhetsgraderte anskaffelser) governs all aspects of this platform:

- **Chapter 5 -- Informasjonssikkerhet:** Requires that classified information is protected against unauthorized access, modification, and destruction. Systems processing HEMMELIG information must implement security measures proportional to the damage that unauthorized disclosure would cause.
- **Chapter 6 -- Informasjonssystemsikkerhet:** Mandates that information systems processing classified information must be accredited (godkjent) by NSM before operational use. The system must implement security measures according to the principle of defense-in-depth.
- **Chapter 8 -- Personellsikkerhet:** All personnel with access to HEMMELIG information must hold valid sikkerhetsklarering (security clearance) at the appropriate level, granted through the klarering process managed by the relevant clearance authority.
- **Chapter 9 -- Sikkerhetsgraderte anskaffelser:** The procurement process itself must follow classified procurement procedures, including facility security clearance (bedriftsgodkjenning) and security agreements (sikkerhetsavtale) with NSM.

### 2.2 NSM Guidelines and Requirements

NSM publishes several key documents that directly govern this architecture:

- **Grunnprinsipper for IKT-sikkerhet (Basic Principles for ICT Security):** NSM's foundational security principles organized into four categories: identify, protect, detect, and respond. All are mandatory considerations for accreditation.
- **Krav til sikkerhetsgraderte informasjonssystemer:** Specific technical and procedural requirements for classified information systems, including requirements for TEMPEST protection, access control, cryptographic protection, and audit trails.
- **Veiledning for sikkerhetsgodkjenning:** NSM's accreditation guidance defining the documentation, testing, and approval process required before the system can be authorized to process HEMMELIG data.

### 2.3 NATO Security Policies

Since the platform must support NATO information sharing:

- **C-M(2002)49 -- Security within the North Atlantic Treaty Organisation:** The overarching NATO security policy governing handling of NATO classified information.
- **AC/322-D(2004)0024 -- NATO Information Assurance Policy:** Defines IA requirements for NATO CIS systems.
- **STANAG 4774/4778:** Standards for metadata binding and confidentiality labeling, critical for cross-domain information exchange.
- **NATO SECRET handling requirements:** Equivalent to Norwegian HEMMELIG; interoperability requires mutual recognition and agreed-upon information exchange procedures.

### 2.4 Compliance Matrix

| Requirement Area | Sikkerhetsloven Reference | NSM Grunnprinsipp | Implementation |
|---|---|---|---|
| Access Control | Ch. 5, Ch. 8 | 2.1 - Ivareta satisfactory tilgangskontroll | RBAC + clearance-based access, MFA |
| Encryption | Ch. 6 | 2.3 - Beskytt data i transit og lagring | NSM-approved crypto (national), NATO-approved for interop |
| Audit Logging | Ch. 6 | 3.1 - Oppdage og stopp sikkerhetsbrudd | Immutable centralized logging |
| Physical Security | Ch. 7 | 2.6 - Sikre fysisk infrastruktur | Sperret/beskyttet zone per NSM specs |
| Personnel Security | Ch. 8 | 1.4 - Kontroller personell | HEMMELIG klarering for all privileged access |
| Network Isolation | Ch. 6 | 2.2 - Etabler nettverkssikkerhet | Full air-gap with data diodes |
| Incident Response | Ch. 6 | 4.1 - Vurder og evaluer hendelser | 24/7 SOC capability, NSM notification |
| Configuration Management | Ch. 6 | 2.5 - Kontroller programvare | Hardened baselines, signed images |

---

## 3. Threat Model and Security Posture

### 3.1 Threat Actors

At the HEMMELIG level, the primary threat actors are:

1. **Nation-state adversaries (APT groups):** The predominant threat. State-sponsored actors with significant resources, long time horizons, and sophisticated capabilities including supply chain infiltration, zero-day exploitation, and insider recruitment.
2. **Insider threats:** Personnel with legitimate access who may be compromised, coerced, or disgruntled. The limited pool of 8 cleared personnel makes insider monitoring both more critical and more feasible.
3. **Supply chain compromise:** Hardware or software tampered with during manufacturing, distribution, or maintenance. Particularly relevant for COTS (commercial off-the-shelf) components.

### 3.2 Threat Scenarios

| ID | Scenario | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| T-01 | Supply chain implant in server hardware | Medium | Critical | Trusted procurement, hardware inspection, TEMPEST testing |
| T-02 | Insider exfiltration via removable media | Medium | Critical | USB disabled, media control policy, behavioral monitoring |
| T-03 | Electromagnetic emanation exploitation (TEMPEST) | Low-Medium | Critical | TEMPEST-zoned facility, approved equipment |
| T-04 | Software vulnerability in platform components | High | High | Offline patch management, minimal attack surface |
| T-05 | Cryptographic key compromise | Low | Critical | HSM-backed key management, NSM-approved algorithms |
| T-06 | Unauthorized physical access | Low | Critical | Multi-layer physical security, access logging |
| T-07 | Maintenance/update channel compromise | Medium | High | Offline update process, integrity verification |
| T-08 | Cross-domain data spill to lower classification | Medium | High | Data diodes, strict labeling (STANAG 4774) |

### 3.3 Security Design Principles

1. **Defense in depth (forsvar i dybden):** Multiple independent security layers; no single point of failure in security controls.
2. **Least privilege (minste privilegium):** All access rights are minimized to the operational minimum. Every user and process gets only the permissions required for their function.
3. **Need-to-know (tjenstlig behov):** HEMMELIG clearance alone is insufficient; personnel must also demonstrate a legitimate operational need for access.
4. **Zero trust principles adapted for air-gapped environments:** Even within the air-gapped perimeter, all communications are authenticated, authorized, and encrypted. No implicit trust based on network location.
5. **Fail-secure:** System defaults to a denied/locked state on failure.
6. **Auditability:** Every action that could affect security posture is logged immutably.

---

## 4. Platform Architecture Decision: OpenStack vs Kubernetes

### 4.1 Analysis

| Criterion | OpenStack | Kubernetes | Recommendation |
|---|---|---|---|
| **IaaS Capability** | Native VM management, networking, storage | Requires underlying infra (bare metal or VMs) | OpenStack for base infrastructure |
| **Container Orchestration** | Limited (Zun project, immature) | Industry standard, rich ecosystem | Kubernetes for container workloads |
| **Air-gap Compatibility** | Well-understood offline deployment | Possible but requires careful image registry design | Both viable with planning |
| **Multi-tenancy** | Strong project/domain isolation | Namespace-based; less mature isolation | OpenStack for hard tenant boundaries |
| **Maturity for Defense** | Used in US DoD, UK MOD, and European defense | Increasingly adopted but usually on top of IaaS | OpenStack as proven foundation |
| **Operational Complexity** | High; requires dedicated ops team | High; different skill set | Combined approach requires broad skills |
| **Hardware Management** | Ironic for bare-metal, Nova for VMs | Not designed for hardware management | OpenStack for hardware lifecycle |
| **Network Isolation** | Neutron provides rich network isolation (VLANs, VXLANs, security groups) | Network policies, CNI plugins | Both needed at different layers |
| **Norwegian Defense Experience** | Known usage in FMN contexts | Growing adoption | OpenStack has longer track record |

### 4.2 Recommended Architecture: Hybrid OpenStack + Kubernetes

```
+------------------------------------------------------------------+
|                    HEMMELIG CLOUD PLATFORM                        |
|                                                                   |
|  +------------------------------------------------------------+  |
|  |              OpenStack IaaS Foundation                      |  |
|  |                                                              |  |
|  |  +----------+  +----------+  +-----------+  +----------+   |  |
|  |  |  Nova    |  | Neutron  |  |  Cinder   |  |  Glance  |   |  |
|  |  | (Compute)|  |(Network) |  | (Block    |  | (Image   |   |  |
|  |  |          |  |          |  |  Storage) |  |  Service) |   |  |
|  |  +----------+  +----------+  +-----------+  +----------+   |  |
|  |  +----------+  +----------+  +-----------+  +----------+   |  |
|  |  | Keystone |  | Barbican |  |   Swift   |  |  Ironic  |   |  |
|  |  |  (IdAM)  |  |  (Key    |  | (Object  |  |(Bare     |   |  |
|  |  |          |  |   Mgmt)  |  |  Storage) |  | Metal)   |   |  |
|  |  +----------+  +----------+  +-----------+  +----------+   |  |
|  +------------------------------------------------------------+  |
|                                                                   |
|  +------------------------------------------------------------+  |
|  |         Kubernetes Workload Layer (on OpenStack VMs)        |  |
|  |                                                              |  |
|  |  +------------------+  +------------------+                 |  |
|  |  | Mission Workload |  | Mission Workload |                 |  |
|  |  | Cluster 1        |  | Cluster 2        |                 |  |
|  |  | (Namespace-       |  | (Namespace-       |                 |  |
|  |  |  isolated)        |  |  isolated)        |                 |  |
|  |  +------------------+  +------------------+                 |  |
|  |                                                              |  |
|  |  +------------------+  +------------------+                 |  |
|  |  | Platform Services|  | NATO Interop     |                 |  |
|  |  | (Monitoring, CI, |  | Services         |                 |  |
|  |  |  Registry)        |  | (CDS Gateway)    |                 |  |
|  |  +------------------+  +------------------+                 |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

### 4.3 Component Selection

| Layer | Component | Version/Distribution | Rationale |
|---|---|---|---|
| OS Base | Red Hat Enterprise Linux 9 (or RHEL-derivative) | Latest supported | FIPS 140-2/3 validated crypto modules, long-term support, extensive hardening guides (DISA STIG, CIS), widely used in defense |
| IaaS | OpenStack (Antelope or later) | Enterprise distribution (e.g., Red Hat OpenStack Platform) | Commercially supported, proven in defense contexts, comprehensive IaaS |
| Container Orchestration | Kubernetes (via OpenShift or RKE2) | Latest stable | RKE2 (Rancher Government) is specifically designed for air-gapped government use; OpenShift provides additional enterprise/security features |
| Container Runtime | CRI-O or containerd | Latest stable | CRI-O preferred for OpenShift; containerd for RKE2. Both are OCI-compliant and more secure than Docker daemon |
| CNI Plugin | Calico or Cilium | Latest stable | Calico for mature network policy enforcement; Cilium for eBPF-based observability. Both support network policy for pod isolation |
| Storage | Ceph (via Rook on K8s, native for OpenStack) | Latest LTS | Provides block (RBD), object (RGW), and file (CephFS) storage; encryption at rest; proven at scale in defense |
| Secrets Management | HashiCorp Vault (enterprise) + Barbican | Latest | HSM-backed secret storage; Vault for K8s secrets, Barbican for OpenStack key management |
| Image Registry | Harbor | Latest stable | Air-gap capable, vulnerability scanning (Trivy), content trust (Notary/cosign), RBAC |
| Service Mesh | Istio (with ambient mode) or Linkerd | Latest stable | mTLS between all services, traffic policy enforcement, observability |
| GitOps/CD | Flux CD or ArgoCD | Latest stable | Declarative, air-gap compatible, git-based deployments |
| Monitoring | Prometheus + Grafana + AlertManager | Latest stable | Industry standard; works fully offline |
| Logging | Elasticsearch + Fluentd + Kibana (EFK) or Loki + Promtail | Latest stable | Centralized immutable logging; required for accreditation |
| SIEM Integration | Custom forwarder to Forsvaret SOC | N/A | Security events forwarded via approved channels |

---

## 5. Network Architecture and Air-Gap Design

### 5.1 Air-Gap Architecture

The platform is **fully air-gapped** from all networks of lower classification. There is no IP connectivity, no shared physical media, and no electromagnetic pathway to unclassified or BEGRENSET networks.

```
+================================================================+
|                      HEMMELIG ZONE                               |
|                                                                  |
|   +------------------+    +------------------+                   |
|   | Management       |    | Workload         |                   |
|   | Network          |    | Network          |                   |
|   | (VLAN 100)       |    | (VLAN 200-299)   |                   |
|   |                  |    |                  |                   |
|   | - OpenStack API  |    | - Tenant VMs     |                   |
|   | - K8s API        |    | - K8s Pods       |                   |
|   | - IPMI/BMC       |    | - Inter-service  |                   |
|   +------------------+    +------------------+                   |
|                                                                  |
|   +------------------+    +------------------+                   |
|   | Storage Network  |    | Monitoring/Log   |                   |
|   | (VLAN 300)       |    | Network          |                   |
|   |                  |    | (VLAN 400)       |                   |
|   | - Ceph cluster   |    |                  |                   |
|   | - iSCSI/RBD      |    | - Log collection |                   |
|   | - Replication    |    | - SIEM feeds     |                   |
|   +------------------+    +------------------+                   |
|                                                                  |
|   +------------------+                                           |
|   | NATO Interop     | <=== Data Diode / CDS ===>  NATO Network |
|   | DMZ (VLAN 500)   |     (Cross Domain Solution)              |
|   +------------------+                                           |
|                                                                  |
+============================+=====================================+
                             |
                    DATA DIODE (one-way)
                    for software updates
                             |
                    +--------v--------+
                    | Update Staging  |
                    | (BEGRENSET Zone)|
                    +-----------------+
```

### 5.2 Network Segmentation Details

**Physical separation is mandatory** between the HEMMELIG zone and all lower-classification environments. This means:

- Separate physical switches, cabling, and patch panels
- No shared physical infrastructure with lower-classification systems
- All networking equipment within the HEMMELIG zone must be located within the physically secured area
- Fiber optic cabling preferred within the HEMMELIG zone (easier TEMPEST compliance, no electromagnetic emanation from cables)

**Internal segmentation** within the HEMMELIG zone uses VLANs enforced on managed switches with 802.1Q trunking:

| VLAN | Purpose | Access Control |
|---|---|---|
| 100 | Management (OpenStack/K8s control plane, BMC/IPMI) | Restricted to platform operators (klarert personnel only) |
| 200-299 | Workload networks (tenant-specific) | Neutron security groups + K8s network policies |
| 300 | Storage network (Ceph) | Only compute and storage nodes |
| 400 | Monitoring and logging | Collector agents and SIEM |
| 500 | NATO interoperability DMZ | CDS gateway hosts only |

### 5.3 Data Diode Architecture

For software updates and limited data ingress, a **hardware data diode** is employed:

- **Direction:** One-way from lower classification (BEGRENSET) into HEMMELIG zone
- **Product:** Must be NSM-approved data diode (e.g., Advenica SecyCross, Nexor, or equivalent NSM-evaluated product)
- **Process:**
  1. Software packages, patches, and container images are prepared in the BEGRENSET staging environment
  2. All packages are integrity-verified (GPG signatures, SHA-256 checksums) and malware-scanned
  3. Packages are transferred through the data diode into the HEMMELIG zone intake server
  4. A second integrity verification is performed on the HEMMELIG side
  5. Packages are admitted into the internal repository (Harbor for containers, local RPM repo for OS packages)
  6. All transfers are logged on both sides with personnel identification

### 5.4 Cross Domain Solution (CDS) for NATO Interoperability

The NATO information exchange path uses an **NSM-accredited Cross Domain Solution (CDS)**:

- Bidirectional but heavily filtered and inspected
- Content inspection and data loss prevention (DLP) on all outbound data
- Classification label verification (STANAG 4774/4778) on all transfers
- Protocol break (no direct IP path between HEMMELIG and NATO networks)
- Operated exclusively by HEMMELIG-cleared personnel
- Separate accreditation required for the CDS itself

### 5.5 DNS, NTP, and Core Services

All core network services run internally within the air-gapped environment:

- **DNS:** Internal authoritative DNS servers (BIND or CoreDNS) with no external resolution capability
- **NTP:** Stratum 1 NTP server with GPS receiver (within TEMPEST enclosure) or rubidium clock source, providing time synchronization critical for audit log integrity
- **PKI:** Internal Certificate Authority (CA) for all TLS certificates within the environment, managed via Vault PKI secrets engine
- **LDAP/IdAM:** Internal identity store (FreeIPA or Red Hat IdM) integrated with Keystone and Kubernetes OIDC

---

## 6. Compute, Storage, and Infrastructure

### 6.1 Hardware Specifications

All hardware must be procured through NSM-approved supply chain procedures (sikkerhetsgradert anskaffelse).

#### Compute Nodes (OpenStack Nova / Kubernetes Workers)

| Component | Specification | Quantity | Notes |
|---|---|---|---|
| Server | Enterprise rack server (e.g., Dell PowerEdge R760, HPE ProLiant DL380 Gen11) | 12-16 | Must be sourced through approved supply chain |
| CPU | Intel Xeon Scalable 4th Gen or AMD EPYC 4th Gen | 2 per node | Ensure no unauthorized microcode; verify against known supply chain risks |
| RAM | 512 GB DDR5 ECC per node | -- | ECC mandatory for data integrity |
| Local Storage | 2x NVMe SSD (boot/ephemeral) | Per node | Self-encrypting drives (SED) with FIPS-validated encryption |
| Network | 2x 25GbE (workload), 2x 25GbE (storage), 1GbE (IPMI) | Per node | Intel or Mellanox NICs |
| TPM | TPM 2.0 | Per node | Measured boot, integrity attestation |

#### Control Plane Nodes (OpenStack Controllers / K8s Masters)

| Component | Specification | Quantity | Notes |
|---|---|---|---|
| Server | Enterprise rack server | 3 (HA quorum) | Dedicated control plane for reliability |
| CPU | Intel Xeon Scalable or AMD EPYC | 2 per node | |
| RAM | 256 GB DDR5 ECC | Per node | |
| Local Storage | 2x NVMe SSD (RAID-1) | Per node | Redundant boot/root |
| Network | Same as compute | Per node | |

#### Storage Nodes (Ceph OSDs)

| Component | Specification | Quantity | Notes |
|---|---|---|---|
| Server | Storage-optimized server | 5-8 | Minimum 3 for Ceph quorum |
| CPU | Modern Xeon/EPYC | 1-2 per node | |
| RAM | 256 GB DDR5 ECC | Per node | ~4 GB per OSD recommended |
| Storage | 12x NVMe or SAS SSD per node | Per node | All drives SED with FIPS crypto |
| Network | 2x 25GbE (storage), 1x 25GbE (public) | Per node | Dedicated storage network critical for performance |

### 6.2 Storage Architecture

```
+----------------------------------------------------------+
|                    Ceph Cluster                           |
|                                                          |
|  +--------+   +--------+   +--------+   +--------+      |
|  | OSD    |   | OSD    |   | OSD    |   | OSD    | ...   |
|  | Node 1 |   | Node 2 |   | Node 3 |   | Node 4 |      |
|  +--------+   +--------+   +--------+   +--------+      |
|                                                          |
|  Replication Factor: 3 (minimum for HEMMELIG)            |
|  Encryption: At-rest (dm-crypt / LUKS2 on all OSDs)      |
|  Erasure Coding: Not for primary data at this class.     |
|                   level; replication preferred for        |
|                   reliability and simpler recovery        |
+----------------------------------------------------------+
         |               |              |
    +----+----+    +-----+----+   +-----+-----+
    | RBD     |    | CephFS   |   | RGW       |
    | (Block) |    | (File)   |   | (Object)  |
    +---------+    +----------+   +-----------+
    OpenStack       Shared         Backup/
    Cinder          Filesystems    Archive
    K8s PVs
```

**Encryption at rest** is mandatory:

- All Ceph OSDs use dm-crypt/LUKS2 encryption
- Encryption keys stored in Barbican/Vault backed by HSM
- Self-encrypting drives (SED) provide an additional layer
- Key rotation policy: annually or upon personnel change

**Data classification labeling:**

- All storage volumes tagged with classification metadata
- Ceph pool-level separation for different data sensitivity levels within HEMMELIG
- No mixing of classification levels within the same Ceph cluster

### 6.3 Hardware Security Modules (HSM)

An NSM-approved HSM is **mandatory** for cryptographic key management at the HEMMELIG level:

- **Primary:** Network-attached HSM (e.g., Thales Luna Network HSM 7, Utimaco SecurityServer) -- must be on NSM's list of approved cryptographic products or evaluated under Common Criteria at appropriate assurance level
- **Deployment:** Minimum 2 HSMs in HA configuration
- **Integration:** PKCS#11 interface to Vault and Barbican
- **Key types stored:** TLS CA private keys, Ceph encryption master keys, disk encryption keys, authentication tokens
- **Physical security:** HSMs located within the highest-security zone of the facility, tamper-evident seals inspected regularly

---

## 7. Identity, Access Management, and Personnel Security

### 7.1 Personnel Classification and Access Model

Given the team structure (20 total, 8 with HEMMELIG klarering), a strict role separation is required:

| Role | Required Clearance | Count | Access Scope |
|---|---|---|---|
| Platform Administrator | HEMMELIG | 2-3 | Full infrastructure access (OpenStack admin, K8s cluster-admin, hypervisor, BMC) |
| Security Operator | HEMMELIG | 2 | Security monitoring, audit log review, incident response, CDS operation |
| Application/Workload Operator | HEMMELIG | 2-3 | K8s namespace admin for specific workloads, CI/CD pipeline management |
| CDS/NATO Interop Operator | HEMMELIG | 1-2 | Cross domain solution operation, NATO data exchange approval |
| Software Developer (cleared) | HEMMELIG | Subset of above | Deploy to classified environment (may overlap with workload operator) |
| Software Developer (uncleared) | BEGRENSET or none | Up to 12 | Development in unclassified/BEGRENSET environment only; no access to HEMMELIG platform |
| Project Management | BEGRENSET | As needed | No platform access; receives sanitized status reports |

**Critical constraint:** With only 8 cleared personnel, some role overlap is unavoidable. However, the following **separation of duties** must be maintained:

- No single person can both deploy code AND approve its promotion to production
- Platform administrators cannot suppress or modify audit logs
- CDS operators require a second cleared person to approve NATO data transfers (four-eyes principle / to-persons-regel)
- Key custodian duties split across minimum 2 persons (split knowledge)

### 7.2 Identity Architecture

```
+------------------------------------------------------------+
|                                                            |
|   +-------------+    OIDC/SAML    +-------------------+   |
|   |  FreeIPA /  | <=============> |   OpenStack       |   |
|   |  Red Hat    |                 |   Keystone        |   |
|   |  IdM        |                 +-------------------+   |
|   |             |                                         |
|   | - Users     |    OIDC         +-------------------+   |
|   | - Groups    | <=============> |   Kubernetes      |   |
|   | - Policies  |                 |   API Server      |   |
|   | - 2FA/MFA   |                 +-------------------+   |
|   |             |                                         |
|   | - Sudo      |    LDAP         +-------------------+   |
|   |   rules     | <=============> |   Linux PAM       |   |
|   | - HBAC      |                 |   (all nodes)     |   |
|   |             |                 +-------------------+   |
|   +-------------+                                         |
|         |                                                 |
|         | PKCS#11                                         |
|         v                                                 |
|   +-------------+                                         |
|   |    HSM      |  (Stores auth tokens, cert keys)       |
|   +-------------+                                         |
+------------------------------------------------------------+
```

### 7.3 Authentication Requirements

- **Multi-factor authentication (MFA)** mandatory for all human access:
  - Factor 1: Personal certificate on smartcard (Norwegian defense PKI / Buypass or Commfides equivalent for defense) or FIDO2 hardware token
  - Factor 2: PIN or biometric
- **Service accounts** use short-lived certificates or tokens (no long-lived passwords)
- **Session management:** Maximum session duration of 8 hours; automatic lockout after 15 minutes of inactivity
- **Failed authentication:** Account lockout after 5 failed attempts; security alert generated after 3

### 7.4 Authorization Model

**Role-Based Access Control (RBAC)** combined with **Attribute-Based Access Control (ABAC)**:

- OpenStack: Keystone roles mapped to projects, with policy.json/policy.yaml restricting API actions
- Kubernetes: RBAC with ClusterRoles and RoleBindings; OPA Gatekeeper or Kyverno for policy enforcement
- All authorization decisions logged
- Regular access reviews (quarterly minimum, required by Sikkerhetsloven)

### 7.5 Privileged Access Management (PAM)

- All administrative access routed through a jump host / bastion with session recording
- No direct SSH to production nodes; all access via FreeIPA-managed sudo with HBAC (Host-Based Access Control)
- All privileged sessions recorded (terminal recording via tlog or similar)
- Break-glass procedure documented and tested for emergency access when normal auth chain is unavailable

---

## 8. NATO Interoperability Layer

### 8.1 Federated Mission Networking (FMN)

The platform must be compatible with NATO's Federated Mission Networking framework for interoperability:

- **FMN Spiral 5** compliance target for service interoperability
- Support for NATO Core Services (directory, messaging, web services)
- Implementation of NATO-standard service interfaces

### 8.2 Information Exchange Architecture

```
+--------------------+          +-------------------+         +----------------+
|                    |          |                   |         |                |
|  HEMMELIG          |   CDS   |   NATO SECRET     |  NATO   |  NATO Partner  |
|  Workloads         | <=====> |   Gateway /       | Network |  Systems       |
|                    |  (NSM-  |   DMZ             | <=====> |                |
|  - Data preparation|  approved|                   |         |                |
|  - Label assignment|  bidirec-|  - Protocol       |         |                |
|  - Release auth    |  tional) |    translation    |         |                |
|                    |          |  - Format         |         |                |
+--------------------+          |    conversion     |         +----------------+
                               |  - STANAG 4774    |
                               |    label validation|
                               +-------------------+
```

### 8.3 STANAG 4774/4778 Implementation

All data destined for NATO partners must be labeled according to STANAG 4774 (Confidentiality Metadata Label Syntax) and STANAG 4778 (Metadata Binding Mechanism):

- **Automated labeling:** All data objects within the platform carry classification labels in their metadata
- **Label validation:** The CDS validates that outbound data carries correct NATO classification markings
- **Release authorization:** Outbound data requires explicit release marking (e.g., "NATO SECRET REL TO NATO") applied by authorized personnel
- **Audit trail:** Every label assignment and release decision is logged with operator identity and timestamp

### 8.4 Data Exchange Formats

| Data Type | Standard | Notes |
|---|---|---|
| Operational Data | NATO STANAG-defined formats (e.g., OTH-Gold, ADatP-3) | Mission-specific |
| Documents | PDF/A with embedded metadata labels | STANAG 4774 labels |
| Geospatial | DGIWG standards, GML, GeoPackage | NATO GEOINT interop |
| Messaging | XMPP (STANAG 4406 successor path) or SMTP with SMIME | Encrypted end-to-end |
| Web Services | SOAP/REST per FMN service specifications | NATO SOA standards |

### 8.5 Cryptographic Interoperability

- **National (HEMMELIG) encryption:** NSM-approved national cryptographic products (e.g., for data at rest and internal transport)
- **NATO interoperability encryption:** NATO-approved cryptographic products for data in transit to NATO networks (e.g., Type 1 crypto for NATO SECRET)
- **Key management:** Separate key management for national and NATO keys; no mixing of national and NATO cryptographic material
- **Crypto boundary:** Clear demarcation between national and NATO crypto domains at the CDS

---

## 9. Data Sovereignty and Residency

### 9.1 Legal Requirements

Under Sikkerhetsloven and associated regulations:

- **All HEMMELIG data must remain on Norwegian sovereign territory** at all times
- No data replication, backup, or failover to locations outside Norway
- No foreign personnel access to HEMMELIG data without explicit bilateral agreements and NSM approval
- The cloud platform hardware must be physically located in Norway in a facility with Norwegian security clearance (bedriftsgodkjenning)

### 9.2 Data Location Enforcement

Technical measures to enforce data residency:

1. **Physical:** All infrastructure is located in a single Norwegian facility (or potentially two Norwegian facilities for DR, see section 15)
2. **Network:** Air-gap prevents any data egress except through the accredited CDS
3. **Logical:** OpenStack regions and availability zones are all configured within Norwegian boundaries
4. **Storage:** Ceph CRUSH rules configured to ensure all replicas reside on Norwegian hardware
5. **Backup:** All backups stored on-premises within the HEMMELIG zone; no cloud backup services

### 9.3 Data Lifecycle Management

| Phase | Requirement | Implementation |
|---|---|---|
| Creation | Classified at creation, labeled immediately | Mandatory metadata tagging in workload applications |
| Storage | Encrypted at rest, access-controlled | Ceph encryption + RBAC |
| Processing | Processed only by cleared personnel/approved systems | OpenStack/K8s access control |
| Sharing | NATO interop via CDS only | STANAG-labeled, CDS-mediated transfer |
| Archival | Per Forsvaret retention policies | Encrypted archival storage, minimum retention per regulation |
| Destruction | Cryptographic erasure + physical destruction when required | NSM-approved media sanitization (overwrite standards or physical destruction) |

---

## 10. Logging, Auditing, and Continuous Monitoring

### 10.1 Logging Architecture

NSM accreditation requires comprehensive, immutable audit logging.

```
+----------+   +----------+   +----------+   +----------+
| OS       |   | OpenStack|   | K8s      |   | App      |
| Audit    |   | Logs     |   | Audit    |   | Logs     |
| (auditd) |   | (Oslo)   |   | Logs     |   |          |
+----+-----+   +----+-----+   +----+-----+   +----+-----+
     |              |              |              |
     v              v              v              v
+----------------------------------------------------------+
|           Log Collection (Fluentd / Promtail)             |
+----+-----------------------------------------------------+
     |
     v
+----------------------------------------------------------+
|     Immutable Log Storage (Elasticsearch / Loki)          |
|     - Write-once storage (WORM) or cryptographically      |
|       signed log chains                                   |
|     - Minimum 2 year retention                            |
|     - Tamper-evident (hash chains)                        |
+----------------------------------------------------------+
     |
     v
+----------------------------------------------------------+
|     Security Analytics & SIEM                             |
|     - Automated correlation rules                         |
|     - Anomaly detection                                   |
|     - Alerting to Security Operators                      |
+----------------------------------------------------------+
     |
     v (via approved channel)
+----------------------------------------------------------+
|     Forsvaret SOC / NSM NorCERT Integration               |
|     (incident reporting as required by Sikkerhetsloven)   |
+----------------------------------------------------------+
```

### 10.2 Mandatory Audit Events

The following events MUST be logged for NSM accreditation:

| Category | Events |
|---|---|
| Authentication | All login attempts (success/failure), MFA events, session creation/termination |
| Authorization | All access control decisions, privilege escalation, role changes |
| Data Access | All read/write/delete operations on classified data |
| Administrative | Configuration changes, user management, policy changes, system updates |
| Network | Firewall rule changes, security group modifications, CDS transfers |
| Crypto | Key generation, rotation, destruction, HSM access |
| Physical | Facility access events (integrated from physical access control) |
| System | Boot events (measured boot attestation), service start/stop, resource allocation |

### 10.3 Log Integrity

- All log entries include a cryptographic hash chain (each entry includes hash of previous entry)
- Log storage uses write-once media or WORM-configured storage
- Separate log replication to isolated log archive (accessible only by Security Operators)
- Platform Administrators cannot delete or modify audit logs
- Daily automated integrity verification of log chains

### 10.4 Monitoring and Alerting

- **Infrastructure monitoring:** Prometheus with node_exporter, kube-state-metrics, OpenStack exporters
- **Security monitoring:** OSSEC/Wazuh agents on all nodes for host-based intrusion detection
- **Network monitoring:** Network flow analysis within the HEMMELIG zone (Zeek/Suricata where applicable on internal segments)
- **Alert routing:** Critical security alerts to on-duty Security Operator within 5 minutes
- **Dashboards:** Grafana dashboards for operational and security views (separate dashboards for ops vs security roles)

---

## 11. Operational Model and Staffing

### 11.1 Organizational Structure

Given 8 cleared personnel, the operational model must be carefully designed:

```
+------------------------------------------+
|        HEMMELIG Platform Operations       |
|                                          |
|  Security Lead (1)                       |
|  +-- Security Operator (1)              |
|  |   (Monitoring, audit, incident resp.) |
|  |                                       |
|  Platform Lead (1)                       |
|  +-- Platform Engineer (1-2)            |
|  |   (OpenStack, K8s, Ceph ops)         |
|  |                                       |
|  Workload/DevOps Lead (1)               |
|  +-- Workload Engineer (1)              |
|  |   (App deployment, CI/CD)            |
|  |                                       |
|  NATO Interop Operator (1)              |
|  (CDS operation, NATO data exchange)     |
+------------------------------------------+
          |
          | Sanitized reporting only
          v
+------------------------------------------+
|     BEGRENSET / Unclassified Zone        |
|                                          |
|  Development Team (up to 12)             |
|  - Application development              |
|  - Testing in unclassified environment   |
|  - Code review and preparation           |
|  - Documentation                         |
|                                          |
|  Project Management                      |
|  - Schedule and resource management      |
|  - Stakeholder communication             |
+------------------------------------------+
```

### 11.2 Operational Processes

**Software Development and Deployment Pipeline:**

1. Developers (cleared or uncleared) write code in the BEGRENSET/unclassified development environment
2. Code undergoes peer review, static analysis (SAST), and testing in the unclassified environment
3. Approved code is packaged (container images, RPMs) and signed
4. Packages transferred through data diode into HEMMELIG zone
5. HEMMELIG-cleared Workload Engineer verifies packages, runs security scans in HEMMELIG staging
6. Deployment to HEMMELIG production via GitOps (Flux/Argo) -- approved by a second cleared engineer

**Patch Management:**

1. Patches sourced from vendor in unclassified environment
2. Tested in unclassified staging environment
3. Transferred through data diode
4. Applied in HEMMELIG staging environment
5. Validated by platform team
6. Rolled out to production with change management approval
7. Emergency patches follow expedited process but still require data diode transfer

**On-call and Coverage:**

- With 8 cleared personnel, 24/7 on-site coverage is not feasible
- **Recommended model:** Business hours on-site operations with on-call rotation for after-hours
- Critical monitoring alerts forwarded to on-call cleared personnel via approved secure communications
- Physical response within 2 hours for critical incidents (personnel must be based near the facility)

### 11.3 Knowledge Management and Bus Factor

With only 8 cleared personnel, the **bus factor** is a significant risk:

- Cross-training mandatory: every critical function must be performable by at least 2 people
- All procedures documented in classified operational runbooks stored within the HEMMELIG zone
- Regular tabletop exercises and disaster recovery drills
- Plan for security clearance pipeline: identify additional team members for klarering processing (note: HEMMELIG clearance processing typically takes 6-12 months via Sivil Klareringsmyndighet or relevant military authority)

---

## 12. Accreditation Strategy (NSM)

### 12.1 Accreditation Process Overview

NSM accreditation (sikkerhetsgodkjenning) for a HEMMELIG system follows a structured process:

```
Phase 1: Planning          Phase 2: Implementation    Phase 3: Evaluation       Phase 4: Authorization
+-------------------+     +--------------------+     +-------------------+     +------------------+
| - System Security |     | - Build platform   |     | - Security testing|     | - NSM reviews    |
|   Target (SST)    |     | - Implement        |     | - Penetration     |     |   all evidence   |
| - Security Concept|     |   controls         |     |   testing         |     | - Residual risk  |
|   (Sikkerhets-    |     | - Document as-built|     | - TEMPEST testing |     |   accepted?      |
|   konsept)        |     |   configuration    |     | - Vulnerability   |     | - Accreditation  |
| - Risk Assessment |     | - Establish SOPs   |     |   assessment      |     |   decision       |
| - NSM engagement  |     |                    |     | - NSM audit       |     |   (ATO/DATO/IATO)|
+-------------------+     +--------------------+     +-------------------+     +------------------+
```

### 12.2 Required Documentation

| Document | Norwegian Term | Description |
|---|---|---|
| System Security Target | Sikkerhetsmål | Defines what the system must protect and the security objectives |
| Security Concept | Sikkerhetskonsept | How the security objectives are met; the security architecture |
| Risk Assessment | Risikovurdering | Threats, vulnerabilities, likelihood, impact, residual risk |
| Security Operating Procedures | Sikkerhetsinstruks | Day-to-day security procedures for operators |
| Configuration Documentation | Konfigurasjonsdokumentasjon | As-built documentation of all components, versions, settings |
| Interconnection Security Agreement | Sambandsikkerhetsinstruks | For the NATO interconnection via CDS |
| TEMPEST Assessment | TEMPEST-vurdering | Emanation security assessment of the facility and equipment |
| Incident Response Plan | Hendelseshåndteringsplan | Procedures for security incident detection, response, reporting |
| Continuity Plan | Kontinuitetsplan | Business continuity and disaster recovery procedures |
| Test Reports | Testrapporter | Results of security testing, penetration testing, vulnerability assessments |

### 12.3 Accreditation Timeline

Realistic timeline for HEMMELIG system accreditation:

| Phase | Duration | Activities |
|---|---|---|
| Planning and Documentation | 3-4 months | SST, security concept, risk assessment, NSM initial engagement |
| Infrastructure Build | 4-6 months | Hardware procurement (sikkerhetsgradert anskaffelse), facility preparation, platform deployment |
| Security Implementation | 2-3 months (overlapping) | Hardening, access control, crypto, monitoring implementation |
| Testing and Evaluation | 2-3 months | Security testing, pen testing, TEMPEST testing, NSM audit visits |
| Remediation | 1-2 months | Address findings from testing and NSM review |
| Authorization Decision | 1-2 months | NSM review and decision |
| **Total** | **12-18 months** | From project start to operational accreditation |

### 12.4 Interim Authority to Operate (IATO)

An IATO (midlertidig godkjenning) may be pursued to allow limited operational use before full accreditation:

- Requires demonstration that critical controls are in place
- Typically granted for 6-12 months
- May include restrictions on data types or operations
- Requires a plan and commitment to achieve full accreditation

---

## 13. Physical Security

### 13.1 Facility Requirements

The facility housing the HEMMELIG cloud platform must meet NSM's requirements for a **sperret område (restricted area)** at minimum, with the server room itself classified as a **beskyttet område (protected area)**:

| Zone | NSM Classification | Controls |
|---|---|---|
| Perimeter | Kontrollert område | Fence, CCTV, access control, guard service |
| Building Entrance | Administrativt område | Badge access, visitor logging, reception |
| Operations Area | Sperret område | Two-factor access (badge + PIN/biometric), escort requirement for uncleared visitors, CCTV |
| Server Room | Beskyttet område (HEMMELIG) | Two-person integrity (to-persons-regel for entry), combination lock + badge, intrusion detection, CCTV, no windows, TEMPEST shielding |
| HSM/Crypto Room | Beskyttet område (HEMMELIG) | Same as server room plus additional access restrictions, limited to key custodians |

### 13.2 TEMPEST Requirements

At the HEMMELIG level, TEMPEST (emanation security) is a mandatory consideration:

- **TEMPEST zoning:** The facility must be assessed by NSM's TEMPEST authority to determine the required TEMPEST zone level
- **Equipment:** Depending on zoning results, may require TEMPEST-approved equipment (Zone 1 or Zone 2), or alternatively, sufficient physical separation from the facility boundary (controlled space / inspectable space)
- **Cabling:** Fiber optic cabling strongly preferred for data paths (no electromagnetic emanation)
- **Power:** Filtered power supply to prevent data leakage through power lines
- **Assessment:** NSM will conduct or commission TEMPEST measurements as part of accreditation

### 13.3 Environmental Controls

- UPS and diesel generator for power continuity
- Redundant cooling (N+1 minimum)
- Fire suppression (gas-based, not water, to protect equipment)
- Environmental monitoring (temperature, humidity, water detection)
- Physical intrusion detection on all access points and walls/ceiling/floor of beschyttet område

---

## 14. Supply Chain Security

### 14.1 Sikkerhetsgradert Anskaffelse (Classified Procurement)

All hardware and critical software must be procured following NSM's classified procurement procedures:

1. **Leverandørklarering (Supplier clearance):** Hardware vendors must hold appropriate facility security clearance or operate under a security agreement with NSM
2. **Transport security:** Hardware transported via approved secure logistics; tamper-evident packaging
3. **Receipt inspection:** Upon delivery, all hardware inspected for tampering; serial numbers verified against purchase orders
4. **Chain of custody:** Documented chain of custody from manufacturer to installation
5. **Pre-deployment inspection:** Before rack installation, hardware inspected by qualified personnel (potentially including x-ray or visual inspection for implants, depending on threat assessment)

### 14.2 Software Supply Chain

- All open-source software built from source in a controlled build environment where feasible
- Container base images built internally from verified OS packages
- Software Bill of Materials (SBOM) generated and maintained for all components
- Vulnerability tracking against SBOM (offline CVE database updated via data diode)
- All software packages signed; signature verification at every transfer point
- No direct internet access for package downloads; all software enters through data diode after review

### 14.3 Firmware and Microcode

- Server BIOS/UEFI firmware verified against vendor-published hashes
- BMC/IPMI firmware controlled and updated only through approved process
- CPU microcode updates reviewed and applied through standard patch process
- Drive firmware managed as part of hardware lifecycle

---

## 15. Disaster Recovery and Continuity

### 15.1 Recovery Objectives

| Metric | Target | Rationale |
|---|---|---|
| RPO (Recovery Point Objective) | 4 hours | Daily full backup + 4-hour incremental |
| RTO (Recovery Time Objective) | 24 hours | Restore from backup to spare hardware |
| MTPD (Maximum Tolerable Period of Disruption) | 72 hours | Aligned with Forsvaret operational requirements |

### 15.2 Backup Architecture

- **Backup tool:** Velero for Kubernetes workloads; OpenStack backup APIs + Ceph snapshots for IaaS
- **Backup storage:** Dedicated Ceph pool on physically separate storage (still within HEMMELIG zone)
- **Encryption:** All backups encrypted with keys stored in HSM
- **Testing:** Quarterly backup restoration tests (full environment restore annually)
- **Off-site consideration:** If a secondary Norwegian facility is available with HEMMELIG accreditation, encrypted backup replication via dedicated secure link (not via public network). Otherwise, all backups remain on-site in a physically separated area (different fire zone)

### 15.3 High Availability Design

- OpenStack control plane: 3-node HA with HAProxy + Pacemaker/Corosync
- Kubernetes control plane: 3 master nodes with etcd HA
- Ceph: 3x replication, 3+ monitors, 3+ managers
- Network: Redundant switches, bonded NICs (LACP), redundant power feeds
- No single point of failure in critical path

### 15.4 Failure Scenarios

| Scenario | Impact | Response |
|---|---|---|
| Single node failure | Minimal; workloads migrate | Automatic failover, replace hardware |
| Storage node failure | Minimal; Ceph rebalances | Monitor rebalancing, replace hardware |
| Network switch failure | Minimal if redundant | Failover to secondary switch |
| Full rack failure | Significant; capacity reduced | Workload redistribution, capacity planning |
| Facility-level disaster | Total loss | Restore from off-site backup (if available) or complete rebuild |
| CDS failure | NATO interop down | Failover CDS or manual data exchange procedures |

---

## 16. Implementation Roadmap

### Phase 0: Project Initiation (Months 1-2)

- [ ] Execute sikkerhetsavtale (security agreement) with NSM
- [ ] Verify bedriftsgodkjenning (facility security clearance) status
- [ ] Begin klarering process for additional personnel if needed
- [ ] Engage NSM for initial consultation on system security target
- [ ] Finalize facility selection and begin physical security upgrades
- [ ] Begin sikkerhetsgradert anskaffelse process for hardware procurement

### Phase 1: Planning and Design (Months 2-5)

- [ ] Complete System Security Target (Sikkerhetsmål)
- [ ] Complete Security Concept (Sikkerhetskonsept)
- [ ] Complete Risk Assessment (Risikovurdering)
- [ ] Design detailed network architecture
- [ ] Design detailed storage architecture
- [ ] Specify hardware bill of materials
- [ ] Design CDS architecture for NATO interop
- [ ] Develop security operating procedures (draft)
- [ ] Submit documentation to NSM for review

### Phase 2: Infrastructure Build (Months 5-10)

- [ ] Receive and inspect hardware (supply chain security procedures)
- [ ] Complete facility physical security upgrades
- [ ] Install and cable racks, switches, and power distribution
- [ ] TEMPEST assessment and remediation
- [ ] Deploy base operating system (hardened RHEL)
- [ ] Deploy and configure OpenStack
- [ ] Deploy and configure Ceph storage cluster
- [ ] Deploy and configure Kubernetes (on OpenStack)
- [ ] Deploy supporting services (Harbor, Vault, FreeIPA, monitoring, logging)
- [ ] Deploy and configure data diode
- [ ] Deploy and configure CDS for NATO interop
- [ ] Implement and test backup/recovery procedures

### Phase 3: Security Hardening and Testing (Months 9-13)

- [ ] Apply CIS/DISA STIG benchmarks to all systems
- [ ] Configure and validate all access controls
- [ ] Configure and validate all encryption (at rest, in transit)
- [ ] Configure and validate audit logging and monitoring
- [ ] Conduct internal vulnerability assessment
- [ ] Conduct internal penetration testing
- [ ] Commission external penetration testing (NSM-approved tester)
- [ ] TEMPEST testing and certification
- [ ] CDS security testing
- [ ] Remediate all critical and high findings

### Phase 4: Accreditation (Months 13-17)

- [ ] Compile complete accreditation evidence package
- [ ] Submit to NSM for formal evaluation
- [ ] Support NSM audit visits and technical inspections
- [ ] Address any NSM findings or conditions
- [ ] Receive accreditation decision (ATO)
- [ ] (Alternative: seek IATO at month 12 if operational need is urgent)

### Phase 5: Operational Transition (Months 17-18)

- [ ] Onboard initial workloads
- [ ] Conduct operational readiness review
- [ ] Transition to steady-state operations
- [ ] Establish ongoing security monitoring rhythm
- [ ] Schedule first re-accreditation review (typically 2-3 years)

---

## 17. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R-01 | Insufficient cleared personnel (only 8) leading to operational fragility | High | High | Cross-training; initiate klarering for additional staff immediately; document all procedures | Project Lead |
| R-02 | NSM accreditation takes longer than planned | Medium | High | Early engagement with NSM; pursue IATO; maintain regular communication with NSM case officer | Security Lead |
| R-03 | Hardware procurement delays due to sikkerhetsgradert anskaffelse process | Medium | High | Begin procurement in Phase 0; identify alternative approved suppliers; buffer in timeline | Project Lead |
| R-04 | TEMPEST requirements more stringent than anticipated | Medium | Medium | Early TEMPEST pre-assessment; budget for TEMPEST-rated equipment as contingency | Facility Manager |
| R-05 | NATO CDS accreditation delays (separate accreditation from main platform) | Medium | Medium | Engage NSM and NATO security authority early; CDS can be accredited in parallel | NATO Interop Lead |
| R-06 | Key person departure (cleared personnel) | Medium | Critical | Retention strategy; ensure minimum 2-person coverage per role; succession planning | Management |
| R-07 | Software vulnerability in air-gapped environment difficult to patch quickly | High | Medium | Proactive monitoring of CVE databases; pre-staged critical patches; compensating controls | Platform Lead |
| R-08 | Ceph cluster data loss due to multiple concurrent failures | Low | Critical | 3x replication; regular scrubbing; backup to separate storage; monitor OSD health | Platform Lead |
| R-09 | Supply chain compromise of hardware or software | Low | Critical | Follow NSM procurement guidelines; hardware inspection; build software from source; SBOM tracking | Security Lead |
| R-10 | Scope creep from Forsvaret requiring higher-classification processing | Low | High | Clear contractual scope; any change to STRENGT HEMMELIG requires complete re-architecture and re-accreditation | Project Lead |

---

## 18. Appendices

### Appendix A: Glossary of Norwegian Terms

| Norwegian Term | English Translation | Context |
|---|---|---|
| Sikkerhetsloven | Security Act | Norwegian law governing national security |
| NSM (Nasjonal sikkerhetsmyndighet) | National Security Authority | Accreditation and oversight authority |
| Forsvaret | Norwegian Armed Forces | Customer |
| Klarering / Sikkerhetsklarering | Security Clearance | Personnel vetting for access to classified information |
| HEMMELIG | Secret | Classification level (NATO SECRET equivalent) |
| STRENGT HEMMELIG | Top Secret | Higher classification (not in scope) |
| KONFIDENSIELT | Confidential | Lower classification (NATO CONFIDENTIAL equivalent) |
| BEGRENSET | Restricted | Lowest national classification (NATO RESTRICTED equivalent) |
| Sikkerhetsgodkjenning | Security Accreditation | Formal approval to process classified data |
| Sikkerhetsgradert anskaffelse | Classified Procurement | Procurement process for classified contracts |
| Bedriftsgodkjenning | Facility Security Clearance | Clearance for a company/facility to handle classified info |
| Sikkerhetsavtale | Security Agreement | Agreement between company and NSM |
| Sperret område | Restricted Area | Physical security zone |
| Beskyttet område | Protected Area | Higher physical security zone |
| To-persons-regel | Two-person rule | Dual-control requirement |
| Sikkerhetskonsept | Security Concept | Architecture document for accreditation |
| Sikkerhetsmål | Security Target | Objectives document for accreditation |
| Risikovurdering | Risk Assessment | Formal risk analysis |
| Hendelseshåndteringsplan | Incident Response Plan | IR procedures |

### Appendix B: Key Assumptions

1. The contracting company holds valid bedriftsgodkjenning at HEMMELIG level
2. A suitable facility in Norway is available or can be prepared within the project timeline
3. NSM will engage constructively in the accreditation process within standard timelines
4. NATO interoperability requirements are limited to NATO SECRET level (not COSMIC TOP SECRET or other special handling)
5. The 8 cleared personnel are available full-time for this project during build phase
6. Forsvaret will provide the NATO-side connectivity and crypto equipment for the CDS interconnection
7. Budget is sufficient for TEMPEST-rated equipment if required by NSM assessment

### Appendix C: Reference Documents

1. Lov om nasjonal sikkerhet (Sikkerhetsloven), LOV-2018-06-01-24
2. Forskrift om virksomheters arbeid med forebyggende sikkerhet (Virksomhetsikkerhetsforskriften)
3. Forskrift om sikkerhetsklarering og annen personkontroll (Klareringsforskriften)
4. Forskrift om sikkerhetsgraderte anskaffelser
5. NSM Grunnprinsipper for IKT-sikkerhet (v2.0)
6. NSM Veiledning for sikkerhetsgodkjenning av informasjonssystemer
7. C-M(2002)49 -- Security within the North Atlantic Treaty Organisation
8. AC/322-D(2004)0024 -- NATO Information Assurance Policy
9. STANAG 4774 -- Confidentiality Metadata Label Syntax
10. STANAG 4778 -- Metadata Binding Mechanism
11. NATO Federated Mission Networking Implementation Plan

### Appendix D: Technology Version Matrix

| Component | Target Version | End of Support | Update Path |
|---|---|---|---|
| RHEL | 9.x (latest minor) | 2032 | Minor updates via data diode |
| OpenStack | 2024.2 (Dalmatian) or 2025.1 | Per upstream + vendor | Major upgrades during maintenance windows |
| Kubernetes | 1.29+ (latest stable) | ~14 months per minor | Rolling upgrades via GitOps |
| Ceph | Reef (18.x) or Squid (19.x) | Per upstream + vendor | Rolling upgrades |
| Harbor | 2.x latest | Continuous | Container image update |
| Vault | 1.15+ enterprise | Per vendor | Rolling upgrade |
| Prometheus/Grafana | Latest stable | Continuous | Container image update |
| Calico/Cilium | Latest stable | Continuous | CNI upgrade procedure |

---

*This document is a living architecture reference and must be updated as the design evolves through the accreditation process. All changes must be tracked and approved by the Security Lead and submitted to NSM as part of configuration management.*

**Document Control:**
- Author: Cloud Architecture Team
- Review: Security Lead, Platform Lead, Project Lead
- Approval: Required before submission to NSM
- Classification of this document: To be determined based on content sensitivity; likely BEGRENSET or HEMMELIG depending on level of detail about specific security controls.
