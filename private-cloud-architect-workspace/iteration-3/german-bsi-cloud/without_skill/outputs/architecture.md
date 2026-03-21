# Private Cloud Architecture for VS-NfD Data Processing
## Bundesbehoerde — BSI IT-Grundschutz & C5 Compliant OpenStack Cloud

**Document Classification:** VS-NfD (Draft — to be classified upon completion)
**Version:** 1.0
**Date:** 2026-03-20
**Author:** Cloud Architecture Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Legal Framework](#2-regulatory-and-legal-framework)
3. [Threat Model and Risk Assessment](#3-threat-model-and-risk-assessment)
4. [Architecture Overview](#4-architecture-overview)
5. [Physical Infrastructure Layer](#5-physical-infrastructure-layer)
6. [Network Architecture](#6-network-architecture)
7. [OpenStack Platform Architecture](#7-openstack-platform-architecture)
8. [Encryption and Key Management](#8-encryption-and-key-management)
9. [Identity, Access, and Authorization](#9-identity-access-and-authorization)
10. [Logging, Monitoring, and SIEM](#10-logging-monitoring-and-siem)
11. [FLOSS vs. BSI-Approved Products — Layer-by-Layer Analysis](#11-floss-vs-bsi-approved-products--layer-by-layer-analysis)
12. [Data Sovereignty and Residency](#12-data-sovereignty-and-residency)
13. [Backup, Disaster Recovery, and Business Continuity](#13-backup-disaster-recovery-and-business-continuity)
14. [Operational Security Processes](#14-operational-security-processes)
15. [BSI Accreditation Roadmap](#15-bsi-accreditation-roadmap)
16. [C5 Attestation Process](#16-c5-attestation-process)
17. [IT-Grundschutz Implementation](#17-it-grundschutz-implementation)
18. [Personnel and Organizational Requirements](#18-personnel-and-organizational-requirements)
19. [Supply Chain and Procurement Considerations](#19-supply-chain-and-procurement-considerations)
20. [Migration and Deployment Strategy](#20-migration-and-deployment-strategy)
21. [Cost Estimation Framework](#21-cost-estimation-framework)
22. [Open Issues and Recommendations](#22-open-issues-and-recommendations)
23. [Appendices](#23-appendices)

---

## 1. Executive Summary

This document defines the reference architecture for a private cloud platform operated by a German federal agency (Bundesbehoerde) for the processing of data classified as **VS-NfD** (Verschlusssache — Nur fuer den Dienstgebrauch) under the Verschlusssachenanweisung (VSA). The platform is based on **OpenStack** and aims to achieve:

- **BSI IT-Grundschutz** compliance at the level of **Hoch** (high protection) per BSI-Standard 200-series
- **BSI C5** (Cloud Computing Compliance Criteria Catalogue) attestation (Type 2)
- Full **data residency within Germany** at all times, including backups and metadata
- Maximum use of **FLOSS (Free/Libre Open Source Software)** where BSI requirements permit

**Key architectural decisions:**

- OpenStack serves as the IaaS control plane with hardened, audited configuration
- BSI-approved (or BSI-evaluated) cryptographic modules are **mandatory** for VS-NfD encryption at rest and in transit — this is the primary area where pure FLOSS is insufficient without certification
- Network segmentation uses a defense-in-depth model with physically separated management, tenant, and storage networks
- All infrastructure is housed in BSI-approved or agency-controlled data centers within Germany
- A Sovereign Cloud Operating Model is adopted with German-national-only operations staff holding security clearances

---

## 2. Regulatory and Legal Framework

### 2.1 Primary Regulations

| Regulation | Relevance | Authority |
|---|---|---|
| **Verschlusssachenanweisung (VSA)** | Governs handling of classified information including VS-NfD | BMI / BSI |
| **BSI IT-Grundschutz Kompendium** (Edition 2025) | Mandatory baseline security standard for federal IT | BSI |
| **BSI-Standard 200-1** | Information Security Management Systems (ISMS) | BSI |
| **BSI-Standard 200-2** | IT-Grundschutz Methodology | BSI |
| **BSI-Standard 200-3** | Risk Analysis based on IT-Grundschutz | BSI |
| **BSI-Standard 200-4** | Business Continuity Management | BSI |
| **BSI C5:2020** (updated) | Cloud Computing Compliance Criteria Catalogue | BSI |
| **BDSG / DSGVO** | Data protection (personal data aspects) | BfDI |
| **IT-Sicherheitsgesetz 2.0** | Critical infrastructure protection | BSI |
| **Technische Richtlinie TR-02102** | Cryptographic algorithms and key lengths | BSI |
| **Technische Richtlinie TR-03116** | Crypto usage in federal administration | BSI |
| **SaetUebergVerschwordenRichtlinie (VS-IT-Richtlinien)** | IT processing of classified data | BSI |

### 2.2 VS-NfD Specific Requirements

VS-NfD is the lowest classification level in Germany but still mandates:

- **Encryption:** All VS-NfD data must be encrypted at rest and in transit using BSI-approved cryptographic products or at minimum algorithms compliant with TR-02102
- **Access Control:** Need-to-know principle; personnel must hold Ue1 (einfache Sicherheitsueberpruefung) clearance at minimum
- **Physical Security:** Zones must meet VS-NfD physical security requirements (no public access, access logging, locked server rooms)
- **Approved Products:** For certain use cases (especially network encryption and VPN), BSI maintains a list of **zugelassene Produkte** (approved products) — use of these is mandatory for cross-site VS-NfD transport
- **Incident Reporting:** Security incidents must be reported to the BSI and agency's own Informationssicherheitsbeauftragter (ISB)
- **Audit Trail:** Complete, tamper-evident logging of all access to VS-NfD data

### 2.3 Legal Data Residency

- All data, including metadata, logs, backups, and encryption keys, must remain within the Federal Republic of Germany
- No foreign-entity access to infrastructure management (this excludes standard US-headquartered cloud providers unless they offer a fully sovereign German instance with legal separation)
- Operations staff must be German nationals with appropriate security clearance

---

## 3. Threat Model and Risk Assessment

### 3.1 Threat Actors

| Actor | Capability | Relevance to VS-NfD |
|---|---|---|
| Foreign intelligence services | Very high | Primary threat — VS-NfD is specifically designed to protect against unauthorized foreign access |
| Organized cybercrime | High | Ransomware, data exfiltration |
| Insider threats | Medium-High | Disgruntled employees, social engineering |
| Hacktivists | Medium | Politically motivated attacks on government infrastructure |
| Supply chain compromise | Medium-High | Backdoors in hardware/software components |

### 3.2 Key Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Unauthorized access to VS-NfD data | Medium | Very High | Encryption, MFA, RBAC, network segmentation |
| Supply chain backdoor in hardware | Medium | Very High | BSI-evaluated hardware, supply chain verification |
| Cryptographic weakness | Low | Very High | BSI TR-02102 compliant algorithms, approved products |
| Insider data exfiltration | Medium | High | DLP, behavioral monitoring, need-to-know access |
| Hypervisor escape | Low | Very High | Hardened KVM, SELinux/AppArmor, regular patching |
| Network lateral movement | Medium | High | Microsegmentation, zero-trust networking |
| Physical access breach | Low | Very High | BSI zone model, access controls, CCTV |
| Loss of availability | Medium | High | HA architecture, DR site, BCM per BSI 200-4 |

---

## 4. Architecture Overview

### 4.1 High-Level Design

```
+============================================================================+
|                        AGENCY PRIVATE CLOUD (VS-NfD)                       |
|                                                                            |
|  +---------------------------+    +---------------------------+            |
|  |   PRIMARY SITE (RZ-1)    |    |   SECONDARY SITE (RZ-2)  |            |
|  |   (e.g., Bonn region)    |    |   (e.g., Berlin region)  |            |
|  |                           |    |                           |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  | Management Plane    | |    |  | Management Plane    | |            |
|  |  | (OpenStack Control) | |    |  | (Standby Control)   | |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  | Compute Plane       | |    |  | Compute Plane       | |            |
|  |  | (Nova/KVM)          | |    |  | (Nova/KVM)          | |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  | Storage Plane       | |    |  | Storage Plane       | |            |
|  |  | (Ceph)              | |    |  | (Ceph Replica)      | |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  |  | Network Plane       | |    |  | Network Plane       | |            |
|  |  | (OVN + HW FW)       | |    |  | (OVN + HW FW)      | |            |
|  |  +---------------------+ |    |  +---------------------+ |            |
|  +---------------------------+    +---------------------------+            |
|                                                                            |
|  +----------------------------------------------------------------------+ |
|  |              Cross-Site Link: BSI-Approved VPN (IPsec)               | |
|  |              (e.g., SINA Box, genuscreen, or equivalent)             | |
|  +----------------------------------------------------------------------+ |
|                                                                            |
|  +----------------------------------------------------------------------+ |
|  |              Central Services                                         | |
|  |  - SIEM (Wazuh/Elastic or BSI-evaluated product)                    | |
|  |  - PKI (EJBCA or DFN-PKI / agency CA)                               | |
|  |  - IAM (Keycloak + LDAP/AD)                                         | |
|  |  - Secrets Management (HashiCorp Vault OSS)                         | |
|  |  - Configuration Management (Ansible)                                | |
|  |  - Container Registry (Harbor)                                       | |
|  +----------------------------------------------------------------------+ |
+============================================================================+
```

### 4.2 Design Principles

1. **Defense in Depth:** Multiple independent security layers; no single point of compromise
2. **Zero Trust:** No implicit trust based on network location; every request is authenticated and authorized
3. **Least Privilege:** Minimum necessary permissions for all accounts and services
4. **Separation of Duties:** Administrative roles are split; no single administrator has full access
5. **Sovereignty by Design:** Every component evaluated for German/EU sovereignty; no foreign intelligence access vectors
6. **Auditability:** Every action logged, centrally collected, tamper-evident
7. **FLOSS Preference:** Open source used wherever BSI requirements do not mandate certified proprietary products
8. **Automation:** Infrastructure as Code (IaC) to ensure reproducibility and reduce human error

---

## 5. Physical Infrastructure Layer

### 5.1 Data Center Requirements

Both data center sites (RZ-1, RZ-2) must meet:

- **Location:** Within Germany, minimum 100 km geographic separation
- **Physical Security Zones:** Compliant with BSI IT-Grundschutz module INF.2 (Rechenzentrum) and VS-NfD zone model
  - Zone 0: Public perimeter, fencing, CCTV
  - Zone 1: Building access — mantrap, badge + PIN
  - Zone 2: Server room access — biometric + badge + PIN
  - Zone 3: Rack-level — locked cabinets, individual access logging
- **Environmental:** Redundant power (2N), UPS, diesel generator, redundant cooling, fire suppression (gas-based, not water)
- **Certification:** Ideally EN 50600 Class 3 or higher; if agency-owned, must pass BSI audit for VS-NfD processing
- **Ownership/Operation:** Preferred: agency-owned. Alternative: German government data center provider (e.g., ITZBund facilities). If third-party: must be German-owned with no foreign parent company influence

### 5.2 Hardware Selection

| Component | Recommendation | Notes |
|---|---|---|
| **Servers** | HPE ProLiant DL or Dell PowerEdge with TPM 2.0 | Ensure supply chain verification; consider servers assembled in EU |
| **TPM** | Infineon or STMicroelectronics TPM 2.0 | European TPM manufacturer preferred for sovereignty |
| **Network Switches** | Enterprise switches with no known backdoors | Evaluate Huawei/ZTE ban implications; prefer European or US-allied vendors |
| **Firewalls (Perimeter)** | BSI-approved appliances | Required for VS-NfD perimeter; see Section 11 |
| **HSM** | Utimaco SecurityServer or equivalent BSI-evaluated HSM | Mandatory for key storage; German manufacturer |
| **Storage** | Standard enterprise SSDs/NVMe for Ceph OSD nodes | Self-encrypting drives (SEDs) with FIPS or CC certification preferred |
| **Out-of-Band Management** | iLO/iDRAC on isolated management VLAN | Firmware must be current; disable internet-facing access |

### 5.3 Hardware Supply Chain Considerations

For VS-NfD, BSI recommends:

- Procurement through verified channels (no grey market)
- Hardware integrity verification upon delivery (tamper-evident packaging, serial number verification)
- Firmware validation against vendor-published hashes
- TPM-based measured boot to detect firmware tampering
- Consideration of BSI hardware recommendations (BSI publishes guidance on trusted hardware)

---

## 6. Network Architecture

### 6.1 Network Segmentation Model

The network is divided into physically and logically separated planes:

```
+------------------------------------------------------------------+
|                    NETWORK SEGMENTATION                           |
|                                                                   |
|  [EXTERNAL ZONE]                                                  |
|    |                                                              |
|    +-- BSI-Approved Perimeter Firewall (VS-NfD approved)         |
|    |                                                              |
|  [DMZ ZONE]                                                       |
|    |  - Reverse Proxy / Load Balancer                             |
|    |  - API Gateway (for OpenStack public endpoints)              |
|    |                                                              |
|    +-- Internal Firewall (can be FLOSS: OPNsense or HW)          |
|    |                                                              |
|  [MANAGEMENT ZONE] (physically separated network)                 |
|    |  - OpenStack Control Plane (Keystone, Nova-API, etc.)        |
|    |  - Ansible Control Node                                      |
|    |  - SIEM Collectors                                           |
|    |  - Monitoring (Prometheus/Grafana)                            |
|    |  - Vault, PKI                                                |
|    |                                                              |
|  [COMPUTE ZONE] (VLAN-separated)                                  |
|    |  - Nova Compute Nodes                                        |
|    |  - Tenant VM overlay networks (GENEVE via OVN)               |
|    |                                                              |
|  [STORAGE ZONE] (physically separated network)                    |
|    |  - Ceph MON, OSD, MDS nodes                                  |
|    |  - Ceph public + cluster networks                            |
|    |                                                              |
|  [CROSS-SITE ZONE]                                                |
|    |  - BSI-Approved VPN Gateway (SINA or genuscreen)             |
|    |  - Dedicated dark fiber or MPLS with encryption              |
+------------------------------------------------------------------+
```

### 6.2 Network Security Controls

| Control | Implementation | FLOSS/Proprietary |
|---|---|---|
| **Perimeter Firewall** | BSI-approved product (see Section 11) | **Proprietary required** |
| **Cross-site VPN** | SINA Box (secunet) or genuscreen (genua) | **Proprietary required** (BSI-zugelassen) |
| **Internal Segmentation** | OVN (Open Virtual Network) + Linux iptables/nftables | FLOSS |
| **Microsegmentation** | OpenStack Security Groups + OVN ACLs | FLOSS |
| **East-West Firewalling** | OVN distributed firewall | FLOSS |
| **TLS for API endpoints** | HAProxy or Nginx with TLS 1.3 (BSI TR-02102 ciphers) | FLOSS |
| **DNS** | Internal BIND9 or Unbound (DNSSEC enabled) | FLOSS |
| **NTP** | Chrony with authenticated NTP sources | FLOSS |
| **Network Monitoring** | Suricata IDS/IPS | FLOSS |
| **802.1X Port Security** | Switch-level NAC | Depends on switch vendor |

### 6.3 Cryptographic Requirements for Network Layer

Per **BSI TR-02102-2** (TLS) and **TR-02102-3** (IPsec/IKE):

- **TLS:** Minimum TLS 1.2 with PFS cipher suites; TLS 1.3 preferred. Mandatory ciphers: AES-256-GCM with ECDHE key exchange (P-256 or P-384 curves, or X25519)
- **IPsec (cross-site):** IKEv2 with AES-256-GCM, SHA-384/SHA-512, DH Group 15+ or ECDH with brainpoolP256r1 / brainpoolP384r1 (BSI preference for Brainpool curves over NIST curves)
- **Internal API communication:** mTLS between all OpenStack services

**Critical:** For the cross-site VPN carrying VS-NfD traffic, a **BSI-zugelassenes Produkt** (BSI-approved product) is mandatory. This cannot be replaced with a FLOSS IPsec implementation (e.g., strongSwan) because:
- The BSI requires evaluated and approved crypto devices for transporting classified data over untrusted networks
- The approval covers the entire product including hardware, firmware, and tamper resistance
- SINA (secunet) and genuscreen (genua) are the most commonly deployed solutions in German government

---

## 7. OpenStack Platform Architecture

### 7.1 OpenStack Release and Distribution

**Recommended Approach:** Use a community OpenStack release (current stable: 2025.2 "E-series" or newer at time of deployment) deployed via **Kolla-Ansible** or **OpenStack-Ansible** (both FLOSS).

**Why not a commercial OpenStack distribution?**
- Commercial distributions (e.g., former SUSE OpenStack Cloud, Canonical Charmed OpenStack) add vendor dependency
- Kolla-Ansible provides containerized deployment with reproducible builds
- FLOSS deployment tools allow full auditability of the control plane — critical for BSI IT-Grundschutz
- However, the agency should consider a German OpenStack integrator (e.g., B1 Systems, OSISM, Cleura, Noris Network, or T-Systems Open Telekom Cloud team for consulting) for support

**Alternative consideration:** **OSISM** (Open Source Infrastructure & Service Manager) is a German-developed OpenStack lifecycle management framework specifically designed for sovereign cloud deployments. It is used in the Sovereign Cloud Stack (SCS) initiative funded by the German Federal Ministry for Economic Affairs. This is the **recommended deployment tool** for this architecture.

### 7.2 OpenStack Component Selection

| OpenStack Service | Function | Version/Notes |
|---|---|---|
| **Keystone** | Identity & Authentication | Federated with agency LDAP/Keycloak; token encryption |
| **Nova** | Compute (VM lifecycle) | KVM hypervisor; CPU pinning, NUMA awareness for isolation |
| **Neutron + OVN** | Networking | OVN as ML2 mechanism driver (replaces OVS agent) |
| **Cinder** | Block Storage | Ceph RBD backend; encrypted volumes |
| **Glance** | Image Service | Ceph RBD backend; image signature verification |
| **Swift** or **Ceph RGW** | Object Storage | Ceph RADOS Gateway preferred for unified storage |
| **Horizon** | Web Dashboard | Hardened, served behind reverse proxy |
| **Barbican** | Key Management | Integrated with HSM backend (Utimaco PKCS#11) |
| **Octavia** | Load Balancing | For tenant-facing LBaaS |
| **Designate** | DNS as a Service | Internal DNS management for tenants |
| **Heat** | Orchestration | Infrastructure as Code for tenant workloads |
| **Ironic** | Bare Metal (optional) | For workloads requiring physical isolation |
| **Manila** | Shared Filesystems (optional) | CephFS backend if shared storage needed |
| **Magnum** | Kubernetes on OpenStack (optional) | If container workloads are planned |

### 7.3 Hypervisor Hardening (Nova/KVM)

The hypervisor is the most critical security boundary in the cloud:

- **KVM** with **QEMU** (FLOSS) — the Linux kernel's built-in hypervisor
- **SELinux** in enforcing mode with sVirt labels (every VM gets unique SELinux context)
- **Seccomp filters** for QEMU processes to limit syscall surface
- **CPU isolation:** Dedicated physical cores for VMs using `cpu_dedicated_set` in Nova
- **NUMA pinning:** Prevent cross-NUMA scheduling to reduce side-channel attack surface
- **No CPU feature passthrough** unless specifically needed (disable SMT/Hyperthreading consideration for Spectre/Meltdown class attacks)
- **Memory encryption:** AMD SEV (Secure Encrypted Virtualization) or Intel TDX if available on selected hardware — provides per-VM memory encryption at the hardware level
- **Live migration:** Encrypted live migration only (QEMU native TLS)
- **Disable unnecessary devices:** Remove USB, serial, and other unnecessary virtual devices from VM definitions
- **Kernel hardening:** `kernel.kptr_restrict=2`, `kernel.dmesg_restrict=1`, `kernel.perf_event_paranoid=3`
- **Regular patching:** Automated security updates for hypervisor host OS (RHEL, Ubuntu LTS, or openSUSE Leap) with controlled reboot windows

### 7.4 Base Operating System

**Recommended:** One of the following FLOSS options:

1. **RHEL 9 / CentOS Stream 9 / AlmaLinux 9** — SELinux-first, widely used in government, FIPS-validated crypto modules available
2. **Ubuntu 24.04 LTS** — Strong OpenStack support, Canonical provides FIPS-validated packages
3. **openSUSE Leap 15.x** — German company (SUSE), good government track record

**For VS-NfD, the preference is RHEL 9 or derivative** because:
- RHEL's OpenSSL and kernel crypto modules have FIPS 140-2/3 validation (recognized by BSI as equivalent baseline)
- SELinux with sVirt provides the strongest hypervisor isolation available in Linux
- BSI has published specific IT-Grundschutz modules for RHEL

All systems must be configured according to the BSI SiSyPHuS Linux hardening guide and/or the relevant CIS Benchmark.

---

## 8. Encryption and Key Management

### 8.1 Encryption Requirements Summary

| Data State | Requirement | Implementation |
|---|---|---|
| **Data at rest (volumes)** | AES-256 | Cinder volume encryption with dm-crypt (LUKS2) |
| **Data at rest (object store)** | AES-256 | Ceph-native encryption or client-side encryption |
| **Data at rest (images)** | AES-256 | Encrypted Glance images |
| **Data at rest (databases)** | AES-256 | TDE or filesystem-level encryption for MariaDB/PostgreSQL |
| **Data in transit (internal)** | TLS 1.2+ | mTLS between all OpenStack services |
| **Data in transit (cross-site)** | IPsec (BSI-approved) | SINA/genuscreen VPN appliance |
| **Data in transit (tenant)** | TLS 1.3 / WireGuard | Tenant responsibility; platform provides guidance |
| **Key storage** | HSM-backed | Utimaco HSM with PKCS#11; Barbican as API frontend |
| **Key wrapping** | AES-256-KWP | Master keys in HSM; DEKs wrapped and stored in Barbican |

### 8.2 Key Management Architecture

```
+-------------------------------------------+
|              Key Hierarchy                 |
|                                            |
|  [HSM] --- Master KEK (Key Encryption Key) |
|    |                                        |
|    +-- [Barbican] --- Project KEKs          |
|         |                                    |
|         +-- Volume DEKs (per Cinder volume)  |
|         +-- Object DEKs (per object)         |
|         +-- Image DEKs (per Glance image)    |
|         +-- Database DEKs                    |
+-------------------------------------------+
```

- **HSM:** Utimaco SecurityServer (or CryptoServer) — German manufacturer, Common Criteria certified, BSI-recognized
- **Barbican:** OpenStack Key Manager — FLOSS, interfaces with HSM via PKCS#11
- **Key rotation:** Automated rotation every 365 days for KEKs; DEKs rotated on re-encryption (policy-driven)
- **Key backup:** HSM key backup to a second HSM at the DR site using vendor-specific secure backup mechanisms
- **NEVER** store master keys in software-only keystores for VS-NfD data

### 8.3 BSI-Approved vs. FLOSS Cryptographic Products

This is the most important decision point:

**Where BSI-approved products are MANDATORY:**

1. **Cross-site VPN for VS-NfD traffic:** BSI-zugelassene Kryptogeraete (e.g., SINA Box by secunet, genuscreen by genua). No alternative.
2. **HSM for master key storage:** Must be Common Criteria EAL4+ evaluated at minimum. Utimaco (German) is the standard choice.

**Where FLOSS cryptographic implementations are ACCEPTABLE (with conditions):**

1. **dm-crypt/LUKS2 for volume encryption:** Acceptable if using BSI TR-02102 compliant algorithms (AES-256-XTS) AND the underlying crypto library (kernel crypto API, OpenSSL) is from a FIPS-validated or BSI-recognized build
2. **TLS for internal service communication:** OpenSSL or GnuTLS with TR-02102 cipher suites
3. **OpenStack Barbican:** Acceptable as key management API layer since actual key protection is delegated to the HSM
4. **Linux kernel crypto API:** Acceptable for data-at-rest encryption — the kernel's AES-NI accelerated implementation is well-audited

**Key principle:** The BSI requires approved *products* primarily for data crossing trust boundaries (network encryption of classified data). For internal encryption within a controlled zone, FLOSS implementations using approved *algorithms* per TR-02102 are acceptable, provided the overall system achieves IT-Grundschutz high protection level.

---

## 9. Identity, Access, and Authorization

### 9.1 Identity Architecture

```
+----------------------------------------------------------+
|                   IDENTITY STACK                          |
|                                                           |
|  [Agency LDAP / Active Directory]                         |
|    |                                                      |
|    +-- [Keycloak] (FLOSS IdP)                             |
|         |  - SAML 2.0 / OIDC provider                     |
|         |  - MFA enforcement (TOTP, FIDO2/WebAuthn)       |
|         |  - Federation with BundID (optional)             |
|         |                                                  |
|         +-- [OpenStack Keystone]                           |
|              |  - Federated authentication via Keycloak    |
|              |  - Domain/Project/Role model                |
|              |  - Token: Fernet (encrypted, time-limited)  |
|              |                                              |
|              +-- [OpenStack Services]                      |
|                   - Nova, Neutron, Cinder, etc.            |
|                   - Policy.yaml for fine-grained RBAC      |
+----------------------------------------------------------+
```

### 9.2 Authentication Requirements

- **Multi-Factor Authentication (MFA):** Mandatory for all administrative access
  - First factor: Username/Password (stored in LDAP with bcrypt/scrypt hashing)
  - Second factor: FIDO2 hardware token (YubiKey or similar) for administrators; TOTP for standard users
  - Smartcard/eID integration possible via Keycloak for agency PKI cards
- **Service accounts:** Certificate-based authentication (mTLS) for inter-service communication
- **No shared accounts:** Every action must be attributable to a named individual
- **Session management:** Token lifetime maximum 4 hours; re-authentication required for privileged operations

### 9.3 Authorization Model

- **OpenStack RBAC** with custom `policy.yaml` files restricting default permissions
- **Principle of Least Privilege:** Custom roles beyond the standard admin/member/reader:
  - `cloud-admin`: Full platform administration (2-3 persons, 4-eye principle for destructive operations)
  - `domain-admin`: Manage projects within an organizational domain
  - `project-admin`: Manage resources within a specific project
  - `security-auditor`: Read-only access to all logs and configurations
  - `operator`: Day-to-day operations without security configuration changes
- **Separation of Duties:** Security configuration changes require approval from ISB (Informationssicherheitsbeauftragter)
- **4-Eye Principle:** Critical operations (e.g., deleting encryption keys, modifying security groups on VS-NfD projects) require approval from a second authorized person

### 9.4 Privileged Access Management (PAM)

- **Bastion Host / Jump Server:** All administrative SSH access goes through a hardened bastion host with session recording
- **Session Recording:** All administrative sessions recorded (e.g., using Teleport OSS, or `script`/`ttyrec` with central log shipping)
- **Just-in-Time Access:** Privileged access granted on request with time-limited elevation
- **No direct root access:** All admin actions via `sudo` with full logging

---

## 10. Logging, Monitoring, and SIEM

### 10.1 Logging Architecture

```
+--------------------------------------------------------------+
|                    LOGGING PIPELINE                            |
|                                                                |
|  [Sources]                                                     |
|    - OpenStack services (oslo.log -> syslog/journal)           |
|    - Linux audit subsystem (auditd)                            |
|    - Network devices (syslog)                                  |
|    - Firewall logs                                             |
|    - Hypervisor/libvirt logs                                   |
|    - Ceph cluster logs                                         |
|    - Application logs                                          |
|                                                                |
|  [Collection]                                                  |
|    - Fluentd or Fluent Bit (FLOSS) on every node               |
|    - Forwarding via TLS-encrypted syslog (TCP/TLS)             |
|                                                                |
|  [Processing & Storage]                                        |
|    - Wazuh Manager (FLOSS SIEM) or OpenSearch/Elasticsearch    |
|    - Log integrity: hash chaining or write-once storage        |
|    - Retention: minimum 12 months online, 5 years archive      |
|                                                                |
|  [Analysis & Alerting]                                         |
|    - Wazuh rules for BSI IT-Grundschutz detection              |
|    - Grafana dashboards for operational monitoring              |
|    - Alert routing to operations team (email, ticketing)       |
+--------------------------------------------------------------+
```

### 10.2 Mandatory Audit Events

Per BSI IT-Grundschutz and VS-NfD requirements, the following events must be logged:

- All authentication attempts (success and failure)
- All authorization decisions (especially denials)
- All administrative actions (API calls with admin scope)
- All changes to security configurations (security groups, firewall rules, encryption settings)
- All access to VS-NfD classified resources (VM start/stop, volume attach/detach, image download)
- All changes to user accounts and role assignments
- All cryptographic key operations (creation, rotation, deletion)
- All network connections crossing zone boundaries
- All physical access events (from building management system integration)

### 10.3 Log Integrity

- **Tamper-evidence:** Logs signed or hash-chained at source before forwarding
- **Write-once storage:** Archive logs to WORM (Write Once Read Many) storage
- **Separation:** Log storage accessible only to security-auditor role; operators cannot modify logs
- **Time synchronization:** All nodes synchronized to authenticated NTP (Chrony) to ensure log correlation

### 10.4 Monitoring Stack

| Component | Tool | FLOSS |
|---|---|---|
| Metrics collection | Prometheus | Yes |
| Metrics storage | Prometheus TSDB / Thanos | Yes |
| Dashboarding | Grafana | Yes |
| Alerting | Alertmanager | Yes |
| Uptime monitoring | Prometheus Blackbox Exporter | Yes |
| OpenStack-specific | prometheus-openstack-exporter | Yes |
| Ceph monitoring | Ceph Dashboard + Prometheus | Yes |
| Network monitoring | Suricata + Zeek | Yes |
| Vulnerability scanning | OpenVAS/Greenbone | Yes |

---

## 11. FLOSS vs. BSI-Approved Products — Layer-by-Layer Analysis

This section provides the definitive analysis of where FLOSS is sufficient and where BSI-approved or certified products are required.

### 11.1 Summary Matrix

| Layer | Component | FLOSS Possible? | BSI-Approved Required? | Recommendation |
|---|---|---|---|---|
| **Perimeter Firewall** | Firewall for VS-NfD zone boundary | Partially | **YES** for VS-NfD perimeter | genuscreen (genua) or SINA (secunet) |
| **Cross-site VPN** | VPN for classified data transport | No | **YES** (mandatory) | SINA Box L3 or genuscreen |
| **Internal Firewalling** | Microsegmentation within cloud | Yes | No | OVN ACLs + nftables (FLOSS) |
| **HSM** | Key storage for VS-NfD keys | No | **YES** (CC EAL4+) | Utimaco CryptoServer / SecurityServer |
| **Disk Encryption** | Volume/block encryption | Yes (with caveats) | Recommended | dm-crypt/LUKS2 with FIPS-validated OpenSSL (FLOSS acceptable) |
| **TLS Implementation** | Service communication | Yes | No | OpenSSL (FLOSS, use FIPS-validated build) |
| **Hypervisor** | VM isolation | Yes | No | KVM/QEMU (FLOSS) with SELinux |
| **Operating System** | Host OS | Yes | No | RHEL 9 / AlmaLinux 9 (FLOSS core, optional RHEL support) |
| **OpenStack** | Cloud platform | Yes | No | Community OpenStack via OSISM (FLOSS) |
| **Storage** | Distributed storage | Yes | No | Ceph (FLOSS) |
| **IdP/IAM** | Identity management | Yes | No | Keycloak (FLOSS) |
| **SIEM** | Security monitoring | Yes | No (but BSI may prefer evaluated product) | Wazuh (FLOSS) — acceptable if properly configured |
| **IDS/IPS** | Intrusion detection | Yes | No | Suricata (FLOSS) |
| **Endpoint Protection** | Malware/EDR on hosts | Partially | Recommended | ClamAV (FLOSS) + consider BSI-evaluated EDR |
| **Smartcard/eID** | Authentication tokens | No | Recommended | FIDO2 tokens from certified manufacturer |
| **Backup Encryption** | Backup encryption keys | Yes (with HSM) | HSM required for keys | Restic/BorgBackup (FLOSS) + HSM for keys |

### 11.2 Detailed Analysis: Where BSI Approval is Strictly Required

**1. Cross-site VPN (Kryptoprodukt fuer VS-NfD Datenuebertragung)**

The BSI maintains a list of "zugelassene IT-Sicherheitsprodukte" (approved IT security products) for processing classified information. For VS-NfD data transport over untrusted networks, you MUST use a product from this list. The main options are:

- **SINA** (Sichere Inter-Netzwerk Architektur) by **secunet Security Networks AG**: The standard solution across German federal government. Available as SINA Box (VPN gateway), SINA Workstation (secure endpoint), and SINA Virtual Workstation. Hardware-tamper-resistant with evaluated firmware.
- **genuscreen** by **genua GmbH**: Firewall and VPN gateway, BSI-approved for VS-NfD. Alternative to SINA for specific use cases.
- **Lancom VPN routers** (selected models): Some Lancom products have VS-NfD approval for specific scenarios.

A FLOSS VPN (e.g., strongSwan, WireGuard, OpenVPN) is **NOT acceptable** for transporting VS-NfD data over untrusted networks, regardless of the algorithms used. The BSI approval covers the entire product including hardware tamper resistance, side-channel protections, key handling, and the evaluated firmware.

**2. HSM (Hardware Security Module)**

While not every HSM usage requires BSI approval, for storing master encryption keys that protect VS-NfD data, the HSM should be:
- Common Criteria EAL4+ (ideally EAL4+ with AVA_VAN.5) certified
- Listed in the BSI's approved products list, or at minimum carrying a CC certificate recognized by BSI

Primary option: **Utimaco** (German company, CryptoServer/SecurityServer line). Alternative: **Thales Luna** (with European-evaluated firmware).

**3. Perimeter Security at VS-NfD Zone Boundary**

The firewall that separates the VS-NfD processing zone from any lower-classified or unclassified network must be a BSI-approved product. Within the VS-NfD zone, standard network security (FLOSS) is acceptable.

### 11.3 Detailed Analysis: Where FLOSS is Acceptable

**1. OpenStack Control Plane**

OpenStack is FLOSS and acceptable for VS-NfD processing. The BSI does not require the cloud management platform itself to be an approved product — rather, it requires the overall system (Gesamtsystem) to meet IT-Grundschutz requirements. OpenStack's code is publicly auditable, which is an advantage for security assessment.

Key hardening steps for OpenStack in VS-NfD context:
- All API endpoints behind TLS reverse proxy
- Fernet token encryption with rotating keys
- Policy.yaml review and restriction for every service
- Database encryption (MariaDB/PostgreSQL TDE or filesystem encryption)
- RabbitMQ with TLS and authentication
- Oslo.middleware for request/response security headers

**2. KVM Hypervisor**

KVM is the Linux kernel's built-in hypervisor and is trusted by multiple national security agencies worldwide. Combined with SELinux/sVirt, it provides strong VM isolation. BSI IT-Grundschutz module SYS.1.5 (Virtualisierung) covers the requirements, and KVM/QEMU meets them when properly hardened.

**3. Ceph Storage**

Ceph is acceptable for VS-NfD data storage when:
- Data at rest is encrypted (via Ceph-native encryption or dm-crypt on OSD disks)
- Ceph management network is physically separated from tenant network
- Access authenticated via CephX with restricted capabilities per OpenStack service
- Encryption keys managed through Barbican/HSM chain

**4. Linux Operating System**

FLOSS Linux distributions (RHEL, AlmaLinux, Ubuntu, openSUSE) are acceptable as the base OS. The BSI has published SiSyPHuS (Study of System Integrity Protection for Highly Performant Unique Systems) specifically analyzing Linux security, and IT-Grundschutz includes specific modules for Linux server hardening (SYS.1.3).

---

## 12. Data Sovereignty and Residency

### 12.1 Data Residency Controls

| Requirement | Implementation |
|---|---|
| All VS-NfD data stored in Germany | Both RZ sites within German territory; Ceph replication only between these sites |
| No data leakage via DNS/NTP/updates | Internal DNS resolvers; NTP from PTB (Physikalisch-Technische Bundesanstalt) or internal sources; package repositories mirrored internally |
| No foreign cloud service dependencies | No SaaS dependencies; all tools self-hosted |
| Metadata stays in Germany | OpenStack databases, Ceph metadata, logs all stored on-premise |
| Backups in Germany | Backup targets within the two German data centers only |
| No foreign remote access | Management access restricted to German IP ranges; operations staff on-premise or via SINA VPN from German territory |

### 12.2 Software Supply Chain Sovereignty

- **Package repositories:** Mirror all upstream repositories internally. Verify GPG signatures.
- **Container images:** Internal Harbor registry. All base images rebuilt from verified sources.
- **OpenStack source:** Use upstream OpenStack releases. Verify commit signatures.
- **No phone-home telemetry:** Disable all telemetry features in all software components.
- **Dependency scanning:** Use Trivy or Grype to scan for known vulnerabilities in all dependencies.
- **SBOM (Software Bill of Materials):** Maintain SBOM for all deployed components (per BSI TR-03183 guidance on SBOM in government IT).

### 12.3 Personnel Sovereignty

- All persons with administrative access to the VS-NfD cloud must be **German nationals**
- All administrators must hold at minimum **Ue1** security clearance (Sicherheitsueberpruefung nach SueG)
- For elevated administrative access (cloud-admin role): **Ue2** clearance recommended
- No remote access from outside Germany
- Third-party consultants (e.g., for OpenStack support) must be cleared and supervised; access only to non-classified components or under four-eye oversight

---

## 13. Backup, Disaster Recovery, and Business Continuity

### 13.1 Backup Architecture

| Component | Backup Tool | Target | Frequency | Retention |
|---|---|---|---|---|
| VM volumes | Cinder backup (Ceph-to-Ceph) | RZ-2 Ceph cluster | Daily incremental, weekly full | 30 days |
| OpenStack databases | MariaDB/PostgreSQL dump | Encrypted backup storage at RZ-2 | Every 4 hours | 30 days |
| Configuration | Ansible playbooks in Git | Internal GitLab (both sites) | Continuous (Git) | Indefinite |
| Secrets/Keys | HSM backup | Second HSM at RZ-2 | After every key change | Indefinite |
| Ceph data | Ceph RBD mirroring | RZ-2 Ceph cluster | Asynchronous continuous | N/A (mirrored) |
| Logs/SIEM | Wazuh/Elastic snapshot | WORM storage at RZ-2 | Daily | 5 years |

**Backup Encryption:** All backups encrypted with keys from the HSM-backed key hierarchy. Backup tool recommendation: **Restic** (FLOSS) or **BorgBackup** (FLOSS) for file-level backups, using AES-256 encryption.

### 13.2 Disaster Recovery

- **RTO (Recovery Time Objective):** 4 hours for critical services; 24 hours for full platform
- **RPO (Recovery Point Objective):** 1 hour for databases; 4 hours for VM volumes
- **DR strategy:** Active-Passive with asynchronous Ceph replication
  - RZ-1: Active site (all workloads)
  - RZ-2: Standby (DR control plane pre-deployed; storage continuously replicated)
  - Failover: Manual decision by operations team; automated technical execution via Ansible playbooks
- **DR testing:** Full DR test at least twice per year (BSI-Standard 200-4 requirement)

### 13.3 Business Continuity Management

Per **BSI-Standard 200-4**, the agency must maintain:
- Business Impact Analysis (BIA) identifying critical cloud services
- BCM plans documented and approved by agency leadership
- Regular BCM exercises (tabletop and live)
- BCM integration with IT-Grundschutz ISMS

---

## 14. Operational Security Processes

### 14.1 Patch Management

- **Security patches:** Applied within 72 hours for critical/high severity (CVSS >= 7.0)
- **Regular patches:** Applied within 30 days in scheduled maintenance windows
- **OpenStack upgrades:** Follows OSISM lifecycle; skip-level upgrades tested in staging environment
- **Kernel updates:** Live patching (kpatch/livepatch) for critical kernel vulnerabilities to avoid unscheduled reboots
- **Automated scanning:** Greenbone/OpenVAS runs weekly vulnerability scans against all infrastructure

### 14.2 Change Management

- All changes tracked in a ticketing system (e.g., OTRS/Znuny — FLOSS, German company)
- Changes to security-relevant configurations require ISB approval
- Infrastructure as Code: All changes committed to Git, reviewed via merge request (4-eye principle)
- Staging environment mirrors production for pre-deployment testing

### 14.3 Incident Response

- **Incident response plan** aligned with BSI IT-Grundschutz module DER.2.1 (Behandlung von Sicherheitsvorfaellen)
- **CERT contact:** Establish relationship with CERT-Bund and agency's internal CERT
- **Reporting obligations:** VS-NfD incidents must be reported to BSI within 24 hours
- **Forensics capability:** Trained personnel or retainer with BSI-certified forensics provider
- **Incident classification** per BSI categories (niedrig/mittel/hoch/sehr hoch)

### 14.4 Hardening and Compliance Scanning

- **OpenSCAP** (FLOSS) for automated compliance checking against:
  - BSI SiSyPHuS Linux profiles
  - CIS Benchmarks
  - DISA STIG (as supplementary reference)
- **Ansible hardening roles:** Apply and verify hardening configuration on every deployment
- **Quarterly compliance scans:** Report to ISB and agency leadership

---

## 15. BSI Accreditation Roadmap

### 15.1 Overview of the Accreditation Process

For a Bundesbehoerde operating VS-NfD IT systems, the following process applies:

```
Phase 1: Preparation (3-6 months)
    |
    +-- Appoint ISB (Informationssicherheitsbeauftragter)
    +-- Establish ISMS per BSI-Standard 200-1
    +-- Define information domain (Informationsverbund)
    +-- Perform structural analysis (Strukturanalyse)
    |
Phase 2: IT-Grundschutz Implementation (6-12 months)
    |
    +-- Apply IT-Grundschutz Kompendium modules
    +-- Perform protection needs assessment (Schutzbedarfsfeststellung)
    +-- Perform risk analysis per BSI-Standard 200-3
    +-- Implement security measures
    +-- Create security concept (Sicherheitskonzept)
    |
Phase 3: VS-NfD Freigabe (3-6 months)
    |
    +-- Submit security concept to BSI
    +-- BSI reviews and may request changes
    +-- Address BSI findings
    +-- Receive Freigabe (approval) for VS-NfD processing
    |
Phase 4: C5 Attestation (parallel, 6-9 months)
    |
    +-- Gap analysis against C5 criteria
    +-- Implement additional C5-specific controls
    +-- Engage BSI-certified C5 auditor
    +-- Type 1 attestation (design effectiveness)
    +-- Type 2 attestation (operational effectiveness, after 6+ months of operation)
    |
Phase 5: Ongoing Operations
    |
    +-- Annual IT-Grundschutz review
    +-- Annual C5 re-attestation
    +-- Continuous monitoring and improvement
    +-- Regular BSI reporting
```

### 15.2 Detailed Timeline

| Phase | Activity | Duration | Dependencies |
|---|---|---|---|
| **Month 1-3** | ISB appointment, ISMS setup, team training | 3 months | Budget approval |
| **Month 2-4** | Hardware procurement (long lead for SINA, HSM) | 2-3 months | Vendor selection |
| **Month 3-6** | Structural analysis, protection needs assessment | 3 months | ISB in place |
| **Month 4-8** | Platform architecture and build (staging) | 4-5 months | Hardware delivery |
| **Month 6-12** | Security concept documentation | 6 months | Runs parallel to build |
| **Month 8-10** | Platform hardening and security testing | 2-3 months | Staging complete |
| **Month 10-12** | Penetration testing (by BSI-certified provider) | 2 months | Platform stable |
| **Month 12-14** | Submit security concept to BSI for VS-NfD Freigabe | 2 months | All documentation ready |
| **Month 14-16** | BSI review and iteration | 2-3 months | BSI workload dependent |
| **Month 16-18** | VS-NfD Freigabe received; production launch | — | BSI approval |
| **Month 18-20** | C5 Type 1 audit | 2 months | Platform operational |
| **Month 24-26** | C5 Type 2 audit (after 6+ months operation) | 2 months | Sufficient operating history |

**Total estimated timeline: 18-24 months to VS-NfD Freigabe, 24-30 months for C5 Type 2.**

### 15.3 Key BSI Interactions

- **BSI Grundschutz-Berater:** Consider engaging a BSI-certified IT-Grundschutz consultant for the initial assessment
- **BSI Fachbereich:** The relevant BSI department (Fachbereich for Cloud Security or for VS-NfD) should be contacted early for guidance on specific requirements
- **Pre-consultation:** Request an informal pre-consultation with BSI before finalizing the architecture — they may flag issues early and save months of rework
- **BSI Tool Support:** Use the BSI's GSTOOL successor (likely Verinice or equivalent) for IT-Grundschutz documentation and management

---

## 16. C5 Attestation Process

### 16.1 C5 Criteria Overview

The BSI C5 (Cloud Computing Compliance Criteria Catalogue) is organized into 17 domains:

| C5 Domain | Key Requirements | Implementation in This Architecture |
|---|---|---|
| **C5-01: Organisation der Informationssicherheit** | ISMS, roles, responsibilities | ISB, ISMS per BSI 200-1 |
| **C5-02: Sicherheitsrichtlinien** | Security policies documented | Security concept document |
| **C5-03: Personal** | Background checks, training | Ue1/Ue2 clearances, security training |
| **C5-04: Asset Management** | Inventory, classification | CMDB (e.g., NetBox — FLOSS), VS-NfD labeling |
| **C5-05: Physische Sicherheit** | Data center security | EN 50600, BSI zone model |
| **C5-06: Regelbetrieb** | Operations procedures | Ansible IaC, change management |
| **C5-07: Identitaets- und Berechtigungsmanagement** | IAM, MFA, RBAC | Keycloak + Keystone + MFA |
| **C5-08: Kryptographie** | Encryption, key management | TR-02102 compliance, HSM, Barbican |
| **C5-09: Kommunikationssicherheit** | Network security | Segmentation, firewalls, IDS |
| **C5-10: Portabilitaet und Interoperabilitaet** | Data portability, no lock-in | OpenStack APIs, standard formats |
| **C5-11: Beschaffung, Entwicklung, Aenderung** | Secure development, change control | Git-based IaC, review process |
| **C5-12: Steuerung von Cloud-Diensten Dritter** | Supply chain management | SBOM, vendor assessment |
| **C5-13: Umgang mit Sicherheitsvorfaellen** | Incident management | Incident response plan, CERT-Bund |
| **C5-14: Kontinuitaet** | BCM, DR | BSI 200-4, dual-site DR |
| **C5-15: Compliance** | Regulatory compliance | Legal review, DPO involvement |
| **C5-16: Umgang mit Ermittlungsanfragen** | Law enforcement requests | Legal process documented |
| **C5-17: Transparenz** | Transparency for customers | Internal documentation for agency units |

### 16.2 C5 Audit Process

1. **Select C5 Auditor:** Must be a Wirtschaftspruefer (certified auditor) with BSI C5 competence. Major firms: KPMG, PwC, Deloitte, EY (all have German C5 audit practices), or specialized firms like HiSolutions, secunet consulting.

2. **Scope Definition:** Define exactly which cloud services are in scope. For this architecture: IaaS (compute, storage, network) provided by the OpenStack platform.

3. **Type 1 Attestation:** Auditor evaluates the *design* of controls at a point in time. This can be done shortly after go-live. Deliverable: SOC 2-style report confirming controls are suitably designed.

4. **Type 2 Attestation:** Auditor evaluates the *operating effectiveness* of controls over a period (minimum 6 months, typically 12 months). Requires evidence that controls were consistently applied. Deliverable: SOC 2-style report confirming controls operated effectively.

5. **Annual Re-attestation:** C5 Type 2 must be renewed annually.

### 16.3 C5 Additional Criteria for Government Use

C5 includes "Zusatzkriterien" (additional criteria) specifically for government cloud usage. These cover:

- Data residency requirements (met: Germany-only)
- Restrictions on foreign government access (met: no foreign-owned infrastructure)
- Enhanced transparency requirements
- Stricter personnel requirements

All additional criteria should be included in the attestation scope for a Bundesbehoerde.

---

## 17. IT-Grundschutz Implementation

### 17.1 Relevant IT-Grundschutz Modules (Bausteine)

The following IT-Grundschutz Kompendium modules are directly applicable:

**Infrastructure (INF):**
- INF.2: Rechenzentrum (Data Center)
- INF.5: Raum/Gebaeude (Building)

**Systems (SYS):**
- SYS.1.1: Allgemeiner Server
- SYS.1.3: Server unter Linux
- SYS.1.5: Virtualisierung
- SYS.1.6: Containerisierung (if applicable)
- SYS.1.8: Speichersysteme (Storage)
- SYS.1.9: Terminalserver (if used)

**Networks (NET):**
- NET.1.1: Netzarchitektur und -design
- NET.1.2: Netzmanagement
- NET.3.1: Router und Switches
- NET.3.2: Firewall
- NET.3.3: VPN
- NET.4.1: TK-Anlagen (if VoIP integrated)

**Applications (APP):**
- APP.4.4: Kubernetes (if Magnum used)
- APP.5.3: Cloud-Anwendungen (Cloud Applications)
- APP.6: Allgemeine Software

**Operations (OPS):**
- OPS.1.1.2: Ordnungsgemaesse IT-Administration
- OPS.1.1.3: Patch- und Aenderungsmanagement
- OPS.1.1.4: Schutz vor Schadprogrammen
- OPS.1.1.5: Protokollierung
- OPS.1.1.6: Software-Tests und -Freigaben
- OPS.1.2.4: Telearbeit (if remote admin)
- OPS.2.1: Outsourcing (if any external providers)
- OPS.2.2: Cloud-Nutzung (if consuming external cloud services)

**Detection and Response (DER):**
- DER.1: Detektion von sicherheitsrelevanten Ereignissen
- DER.2.1: Behandlung von Sicherheitsvorfaellen
- DER.3.1: Audits und Revisionen

**Overarching (ISMS, ORP, CON):**
- ISMS.1: Sicherheitsmanagement
- ORP.1: Organisation
- ORP.2: Personal
- ORP.3: Sensibilisierung und Schulung
- ORP.4: Identitaets- und Berechtigungsmanagement
- CON.1: Kryptokonzept
- CON.2: Datenschutz
- CON.3: Datensicherungskonzept
- CON.6: Loeschen und Vernichten
- CON.7: Informationssicherheit auf Reisen (for mobile admin)

### 17.2 Protection Needs Assessment (Schutzbedarfsfeststellung)

For VS-NfD processing, the protection needs are:

| Protection Goal | Level | Justification |
|---|---|---|
| **Confidentiality** | **HOCH** (High) | VS-NfD classification requires high confidentiality protection |
| **Integrity** | **HOCH** (High) | Compromise of data integrity could lead to incorrect government decisions |
| **Availability** | **NORMAL** to **HOCH** | Depends on specific agency mission; default NORMAL, upgrade to HOCH for mission-critical services |

Because confidentiality and integrity are rated HOCH, this triggers:
- All HOCH-level requirements in each applicable IT-Grundschutz module must be implemented
- A risk analysis per BSI-Standard 200-3 must be performed for any areas not fully covered by IT-Grundschutz modules
- Additional security measures beyond baseline IT-Grundschutz may be required

### 17.3 Documentation Requirements

The IT-Grundschutz process produces the following mandatory documentation:

1. **Sicherheitsleitlinie** (Security Policy): High-level security policy signed by agency head
2. **Strukturanalyse** (Structural Analysis): Complete inventory of all IT assets in the cloud platform
3. **Schutzbedarfsfeststellung** (Protection Needs Assessment): Classification of all assets
4. **Modellierung** (Modeling): Mapping of IT-Grundschutz modules to assets
5. **IT-Grundschutz-Check** (Compliance Check): Gap analysis against all applicable requirements
6. **Risikoanalyse** (Risk Analysis): Per BSI-Standard 200-3 for high-protection-need areas
7. **Realisierungsplan** (Implementation Plan): Plan to close identified gaps
8. **Sicherheitskonzept** (Security Concept): Comprehensive document combining all of the above

**Tooling recommendation:** Use **verinice** (FLOSS) for IT-Grundschutz documentation and management. Verinice is the de facto standard tool for German IT-Grundschutz compliance management and directly supports the BSI module structure.

---

## 18. Personnel and Organizational Requirements

### 18.1 Required Roles

| Role | Responsibility | Clearance | FTE Estimate |
|---|---|---|---|
| **ISB** (Informationssicherheitsbeauftragter) | Overall information security; reports to agency leadership | Ue2 | 1.0 |
| **Deputy ISB** | Backup for ISB | Ue2 | 0.5 |
| **Cloud Platform Architect** | Technical architecture, design decisions | Ue1 | 1.0 |
| **Cloud Operations Engineers** | Day-to-day platform operations | Ue1 | 4.0 (min. for 24/7 on-call) |
| **Security Operations (SecOps)** | SIEM monitoring, incident response | Ue1 | 2.0 |
| **Network Engineers** | Network infrastructure management | Ue1 | 2.0 |
| **Storage Engineers** | Ceph operations | Ue1 | 1.0 |
| **IAM Administrator** | Keycloak/Keystone management | Ue1 | 1.0 |
| **Compliance Manager** | IT-Grundschutz/C5 documentation | Ue1 | 1.0 |
| **Data Protection Officer (DSB)** | DSGVO compliance (if personal data processed) | — | 0.5 |

**Total core team: approximately 14 FTE**

### 18.2 Training Requirements

- All technical staff: BSI IT-Grundschutz Practitioner certification (or higher)
- ISB: BSI IT-Grundschutz Berater certification recommended
- Cloud engineers: OpenStack Administrator certification or equivalent experience
- Security staff: SANS/GIAC certifications or equivalent
- All staff: Annual VS-NfD handling training (Geheimschutzbelehrung)

### 18.3 Organizational Structure

```
Agency Leadership
    |
    +-- ISB (reports directly to leadership, NOT to IT department)
    |    |
    |    +-- Security Operations Team
    |    +-- Compliance Manager
    |
    +-- IT Department Head
         |
         +-- Cloud Platform Team
         |    +-- Architect
         |    +-- Operations Engineers
         |    +-- Network Engineers
         |    +-- Storage Engineers
         |
         +-- IAM Team
         +-- Service Management
```

The ISB must report independently from the IT department to ensure separation of operational and security oversight.

---

## 19. Supply Chain and Procurement Considerations

### 19.1 Procurement Framework

As a Bundesbehoerde, procurement must follow:
- **UVgO** (Unterschwellenvergabeordnung) or **VgV** (Vergabeverordnung) depending on contract value
- **EVB-IT** contract templates for IT procurement
- Security requirements must be part of the procurement specification (Leistungsbeschreibung)

### 19.2 Key Procurement Items

| Item | Estimated Quantity | Procurement Notes |
|---|---|---|
| **SINA Boxes** | 4 (2 per site) | Sole source: secunet. Government framework contract may exist (Rahmenvertrag) |
| **HSMs** | 4 (2 per site, HA pair) | Utimaco preferred; CC-certified |
| **Compute Servers** | 20-40 (depends on workload) | Standard hardware procurement; specify TPM 2.0 requirement |
| **Storage Servers** | 12-20 (Ceph nodes) | High-density storage; NVMe for performance tier |
| **Network Switches** | 10-20 | Leaf-spine topology; specify 25/100 GbE |
| **FIDO2 Tokens** | 50+ (all admin staff + spares) | YubiKey 5 or similar certified token |
| **Rack Infrastructure** | 10-15 racks per site | Locked cabinets with individual key/combination |
| **OpenStack Support** (optional) | Retainer contract | German OpenStack integrator (OSISM, B1 Systems, etc.) |
| **BSI C5 Audit** | Annual | Wirtschaftspruefer with C5 competence |
| **Penetration Testing** | Annual | BSI-certified penetration testing firm |

### 19.3 FLOSS Implications for Procurement

Using FLOSS significantly reduces software licensing costs but requires:
- Investment in personnel (higher skill requirements for operating community OpenStack vs. commercial distribution)
- Optional support contract with a German OpenStack integrator for escalation
- Internal capacity for security patching and updates (no vendor pushing patches)
- Contribution back to upstream communities (recommended but not required)

---

## 20. Migration and Deployment Strategy

### 20.1 Deployment Phases

**Phase 1: Foundation (Months 4-8)**
- Deploy base infrastructure: servers, network, storage hardware
- Install and harden base OS on all nodes
- Deploy Ceph storage cluster
- Deploy OpenStack control plane via OSISM
- Establish management network and security baseline

**Phase 2: Security Hardening (Months 8-10)**
- Configure encryption (volume encryption, TLS everywhere)
- Deploy HSMs and integrate with Barbican
- Deploy SINA VPN for cross-site connectivity
- Configure perimeter firewalls (BSI-approved)
- Deploy and configure Keycloak with MFA
- Deploy SIEM stack (Wazuh)
- Implement OpenSCAP compliance scanning
- Hardening per BSI SiSyPHuS and CIS Benchmarks

**Phase 3: Validation (Months 10-12)**
- Internal security testing
- External penetration test (BSI-certified provider)
- DR test (failover to RZ-2)
- Performance and capacity testing
- User acceptance testing

**Phase 4: Accreditation (Months 12-18)**
- Complete IT-Grundschutz documentation
- Submit security concept to BSI
- Address BSI feedback
- Receive VS-NfD Freigabe

**Phase 5: Production (Months 16-18+)**
- Migrate pilot workloads
- Gradual migration of VS-NfD workloads
- Monitor and optimize
- Prepare for C5 audit

### 20.2 Workload Migration

- Start with non-VS-NfD workloads as a pilot to validate platform stability
- Migrate VS-NfD workloads only after Freigabe is received
- Use lift-and-shift (P2V/V2V) for existing VMs where possible
- Re-platform applications to take advantage of cloud-native features as appropriate
- Maintain rollback capability for the first 3 months after each migration wave

---

## 21. Cost Estimation Framework

### 21.1 Capital Expenditure (CAPEX) Estimate

| Category | Estimated Cost Range (EUR) | Notes |
|---|---|---|
| **Compute servers (40 nodes)** | 800,000 - 1,200,000 | Enterprise servers with TPM, high RAM |
| **Storage servers (16 Ceph nodes)** | 600,000 - 1,000,000 | High-density NVMe storage |
| **Network infrastructure** | 300,000 - 500,000 | Leaf-spine switches, cables |
| **SINA Boxes (4x)** | 200,000 - 400,000 | BSI-approved VPN |
| **HSMs (4x)** | 100,000 - 200,000 | Utimaco CC-certified |
| **Perimeter Firewalls (4x)** | 150,000 - 300,000 | BSI-approved |
| **FIDO2 Tokens (50x)** | 5,000 - 10,000 | YubiKeys |
| **Data center fit-out (2 sites)** | 500,000 - 1,000,000 | If not using existing facilities |
| **TOTAL CAPEX** | **2,655,000 - 4,610,000** | |

### 21.2 Annual Operating Expenditure (OPEX)

| Category | Estimated Annual Cost (EUR) | Notes |
|---|---|---|
| **Personnel (14 FTE)** | 1,400,000 - 2,100,000 | Fully loaded cost |
| **Power and cooling** | 200,000 - 400,000 | Two data center sites |
| **OpenStack support contract** | 100,000 - 200,000 | Optional, with German integrator |
| **BSI C5 audit** | 150,000 - 300,000 | Annual Type 2 attestation |
| **Penetration testing** | 50,000 - 100,000 | Annual external test |
| **Hardware maintenance** | 200,000 - 350,000 | Warranty/support contracts |
| **SINA/HSM maintenance** | 50,000 - 100,000 | Vendor support contracts |
| **Training and certification** | 50,000 - 100,000 | Staff certifications |
| **Software (FLOSS = 0 license cost)** | 0 | FLOSS advantage |
| **TOTAL OPEX** | **2,200,000 - 3,650,000** | |

### 21.3 FLOSS Cost Advantage

Using FLOSS for the cloud platform (OpenStack, Ceph, Linux, Keycloak, Wazuh, etc.) eliminates:
- VMware vSphere licensing: saved ~500,000 EUR/year for this scale
- Commercial cloud platform licensing: saved ~300,000-800,000 EUR/year
- Commercial SIEM licensing: saved ~100,000-300,000 EUR/year
- Commercial storage licensing: saved ~200,000-500,000 EUR/year

**Estimated annual FLOSS savings: 1,100,000 - 2,100,000 EUR** compared to a fully proprietary stack. This offsets the higher personnel costs needed for FLOSS operations expertise.

---

## 22. Open Issues and Recommendations

### 22.1 Open Issues

| ID | Issue | Impact | Recommended Action |
|---|---|---|---|
| OI-01 | BSI pre-consultation not yet scheduled | May delay accreditation | Schedule BSI consultation in Month 1 |
| OI-02 | Data center site selection pending | Blocks hardware procurement | Evaluate ITZBund facilities or agency-owned options |
| OI-03 | Personnel recruitment for cloud team | Insufficient staff delays project | Begin recruitment immediately; German cloud engineers are scarce |
| OI-04 | SINA Box delivery lead time | May be 3-6 months | Order early in procurement phase |
| OI-05 | AMD SEV / Intel TDX hardware availability | Affects confidential computing capability | Verify hardware support during server selection |
| OI-06 | Sovereign Cloud Stack (SCS) alignment | SCS is the German government reference for sovereign cloud | Evaluate adopting SCS standards for interoperability with other agencies |
| OI-07 | BSI approval of Wazuh for VS-NfD SIEM | BSI may prefer evaluated SIEM | Discuss with BSI during pre-consultation |
| OI-08 | Backup encryption key escrow | Key loss means data loss | Define key escrow process with HSM vendor |

### 22.2 Key Recommendations

1. **Engage BSI early:** Request an informal pre-consultation with BSI before finalizing the architecture. This can prevent costly rework later.

2. **Adopt OSISM and Sovereign Cloud Stack (SCS) standards:** The German government is investing in SCS as the reference architecture for sovereign cloud. Aligning with SCS improves interoperability with other German government clouds and signals compliance awareness to BSI.

3. **Start procurement of BSI-approved products immediately:** SINA Boxes and HSMs have long lead times. These are on the critical path.

4. **Hire or contract a BSI IT-Grundschutz Berater:** The IT-Grundschutz process is documentation-heavy and has specific formatting requirements. An experienced consultant will save months of rework.

5. **Invest in automation:** Every manual process is a compliance risk. Ansible-driven Infrastructure as Code, automated compliance scanning (OpenSCAP), and automated patch management reduce both risk and operational burden.

6. **Plan for Kubernetes on Day 2:** While the initial platform is IaaS (VMs), tenant demand for container workloads is likely. Design the architecture to support adding Magnum (Kubernetes on OpenStack) later without re-accreditation of the base platform.

7. **Consider FLOSS as a sovereignty advantage:** The ability to fully audit the source code of the cloud platform is a security advantage for VS-NfD processing. This should be highlighted in the security concept submitted to BSI.

8. **Establish a security operations center (SOC):** Even a small-scale SOC (2 FTE) for the cloud platform will significantly improve the agency's ability to detect and respond to security incidents. This is effectively required by IT-Grundschutz module DER.1.

---

## 23. Appendices

### Appendix A: Glossary

| Term | Definition |
|---|---|
| **BSI** | Bundesamt fuer Sicherheit in der Informationstechnik (Federal Office for Information Security) |
| **C5** | Cloud Computing Compliance Criteria Catalogue |
| **CC** | Common Criteria (ISO/IEC 15408) |
| **DEK** | Data Encryption Key |
| **FLOSS** | Free/Libre Open Source Software |
| **HSM** | Hardware Security Module |
| **ISB** | Informationssicherheitsbeauftragter (Information Security Officer) |
| **IT-Grundschutz** | BSI's IT baseline protection methodology |
| **KEK** | Key Encryption Key |
| **KVM** | Kernel-based Virtual Machine |
| **mTLS** | Mutual TLS (both client and server authenticate) |
| **OSISM** | Open Source Infrastructure & Service Manager |
| **OVN** | Open Virtual Network |
| **RZ** | Rechenzentrum (Data Center) |
| **SCS** | Sovereign Cloud Stack |
| **SINA** | Sichere Inter-Netzwerk Architektur |
| **SueG** | Sicherheitsueberpruefungsgesetz (Security Clearance Act) |
| **TR** | Technische Richtlinie (Technical Guideline) |
| **Ue1/Ue2** | Sicherheitsueberpruefung Stufe 1/2 (Security clearance level 1/2) |
| **VS-NfD** | Verschlusssache — Nur fuer den Dienstgebrauch (Classified — For Official Use Only) |
| **VSA** | Verschlusssachenanweisung (Classified Information Directive) |

### Appendix B: Reference Documents

1. BSI IT-Grundschutz Kompendium (current edition): https://www.bsi.bund.de/grundschutz
2. BSI-Standard 200-1, 200-2, 200-3, 200-4: https://www.bsi.bund.de/200-Standards
3. BSI C5:2020: https://www.bsi.bund.de/C5
4. BSI TR-02102 (Kryptographische Verfahren): https://www.bsi.bund.de/TR-02102
5. BSI TR-03116 (Kryptographische Vorgaben): https://www.bsi.bund.de/TR-03116
6. BSI SiSyPHuS Linux: https://www.bsi.bund.de/SiSyPHuS
7. Sovereign Cloud Stack: https://scs.community
8. OSISM: https://osism.tech
9. Verschlusssachenanweisung (VSA): BMI publication
10. BSI List of Approved Products: https://www.bsi.bund.de/zugelasseneProdukte

### Appendix C: Architecture Decision Records (ADRs)

**ADR-001: Use OpenStack over VMware/Proprietary Cloud**
- Decision: OpenStack (FLOSS)
- Rationale: Full source code auditability, no foreign vendor lock-in, sovereignty, cost savings
- Trade-off: Higher operational complexity, need for specialized personnel

**ADR-002: Use OSISM for OpenStack Deployment**
- Decision: OSISM over Kolla-Ansible or manual deployment
- Rationale: German-developed, SCS-aligned, designed for sovereign cloud, includes lifecycle management
- Trade-off: Smaller community than Kolla-Ansible

**ADR-003: Use Ceph as Unified Storage Backend**
- Decision: Ceph for block (RBD), object (RGW), and file (CephFS) storage
- Rationale: Single FLOSS storage platform, proven at scale, OpenStack-native integration
- Trade-off: Operational complexity; requires dedicated storage expertise

**ADR-004: BSI-Approved VPN for Cross-Site (SINA)**
- Decision: SINA Box for cross-site VPN
- Rationale: BSI-mandated for VS-NfD data transport over untrusted networks; no FLOSS alternative acceptable
- Trade-off: High cost, vendor lock-in to secunet

**ADR-005: Utimaco HSM for Key Management**
- Decision: Utimaco CryptoServer / SecurityServer
- Rationale: German manufacturer, CC-certified, BSI-recognized, PKCS#11 support for Barbican integration
- Trade-off: Proprietary hardware; alternative (Thales) is non-German

**ADR-006: Keycloak for Identity Provider**
- Decision: Keycloak (FLOSS) over commercial IdP
- Rationale: FLOSS, full-featured IdP with SAML/OIDC, MFA support, LDAP federation
- Trade-off: Requires in-house expertise; Red Hat SSO (Keycloak commercial) available for support if needed

**ADR-007: Wazuh for SIEM**
- Decision: Wazuh (FLOSS) over commercial SIEM
- Rationale: FLOSS, good rule coverage, integrates with IT-Grundschutz requirements
- Trade-off: May need BSI discussion on acceptability; commercial alternative (e.g., Splunk with German hosting) as fallback

---

*End of Document*

*This architecture document should be reviewed by the agency's ISB, legal counsel, and BSI-certified consultant before finalization. It should be updated following BSI pre-consultation feedback.*
