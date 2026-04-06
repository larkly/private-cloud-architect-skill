# Private Cloud Architecture for VS-NfD Data Processing
## BSI IT-Grundschutz & C5-Compliant OpenStack Deployment for a German Federal Agency (Bundesbehoerde)

**Document Classification:** VS-NfD (once populated with agency-specific details)
**Version:** 1.0
**Date:** 2026-03-20
**Status:** Architecture Proposal

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Legal Framework](#2-regulatory-and-legal-framework)
3. [Threat Model and Security Objectives](#3-threat-model-and-security-objectives)
4. [Architecture Overview](#4-architecture-overview)
5. [Hardware and Physical Infrastructure](#5-hardware-and-physical-infrastructure)
6. [Network Architecture](#6-network-architecture)
7. [OpenStack Component Architecture](#7-openstack-component-architecture)
8. [FLOSS vs. BSI-Approved Products Decision Matrix](#8-floss-vs-bsi-approved-products-decision-matrix)
9. [Encryption and Key Management](#9-encryption-and-key-management)
10. [Identity, Authentication, and Access Control](#10-identity-authentication-and-access-control)
11. [Logging, Monitoring, and SIEM](#11-logging-monitoring-and-siem)
12. [Backup, Recovery, and Business Continuity](#12-backup-recovery-and-business-continuity)
13. [BSI IT-Grundschutz Compliance Mapping](#13-bsi-it-grundschutz-compliance-mapping)
14. [C5 Attestation Roadmap](#14-c5-attestation-roadmap)
15. [Accreditation Process Navigation](#15-accreditation-process-navigation)
16. [Operational Processes and Governance](#16-operational-processes-and-governance)
17. [Migration and Deployment Strategy](#17-migration-and-deployment-strategy)
18. [Risk Register](#18-risk-register)
19. [Appendices](#19-appendices)

---

## 1. Executive Summary

This document presents a reference architecture for a private cloud platform designed to process data classified as **VS-NfD (Verschlusssache - Nur fuer den Dienstgebrauch)** within a German Federal Agency (Bundesbehoerde). The platform is based on OpenStack and uses Free/Libre Open Source Software (FLOSS) wherever possible, while identifying specific layers where BSI-approved or BSI-certified products are mandatory or strongly recommended.

### Key Design Principles

- **Data sovereignty:** All data processing and storage occurs exclusively within the Federal Republic of Germany, in BSI-approved facilities.
- **Defense in depth:** Multiple independent security layers ensure that compromise of a single layer does not expose VS-NfD data.
- **BSI IT-Grundschutz compliance:** The architecture maps to the BSI IT-Grundschutz Kompendium (Edition 2025) at the "hoher Schutzbedarf" (high protection requirement) level.
- **C5 attestation readiness:** The architecture is designed from inception to satisfy all criteria of the BSI Cloud Computing Compliance Criteria Catalogue (C5:2020, updated 2024).
- **FLOSS preference with pragmatic exceptions:** Open source is used throughout the stack except where BSI accreditation explicitly requires certified products or where FLOSS alternatives cannot meet VS-NfD requirements.
- **Auditability and transparency:** Every component is auditable; FLOSS components provide full source-level transparency.

---

## 2. Regulatory and Legal Framework

### 2.1 Primary Regulations

| Regulation | Relevance | Mandatory |
|---|---|---|
| **Verschlusssachenanweisung (VSA)** | Governs handling of all classified information including VS-NfD | Yes |
| **BSI-Gesetz (BSIG)** | Legal basis for BSI authority over federal IT security | Yes |
| **BSI IT-Grundschutz Kompendium** | Mandatory standard for federal agency IT systems | Yes |
| **BSI C5 (Cloud Computing Compliance Criteria Catalogue)** | Required for cloud services used by federal agencies (per BMI circular) | Yes |
| **IT-Sicherheitsgesetz 2.0 / 3.0** | Defines critical infrastructure requirements | Yes (if KRITIS-relevant) |
| **DSGVO / BDSG** | Data protection for any personal data in the cloud | Yes |
| **Geheimschutzhandbuch** | Physical security requirements for classified environments | Yes |
| **Technische Leitlinie BSI TL-03402** | Cryptographic requirements for VS-NfD | Yes |
| **EVB-IT Cloud** | Procurement framework for cloud services in federal agencies | Yes |
| **Mindeststandards des BSI** | Minimum security standards for federal IT | Yes |

### 2.2 Key BSI Technical Guidelines

- **BSI TR-02102:** Cryptographic algorithms and key lengths
- **BSI TR-03116:** Cryptographic requirements for federal applications
- **BSI TL-03402:** IT security requirements for VS-NfD processing
- **BSI TR-03107:** Electronic identities and trust services
- **BSI TR-02103:** X.509 certificate profiles

### 2.3 Data Residency Requirements

For VS-NfD processing:
- All data must be stored and processed within Germany (no exceptions).
- Data centers must be on German soil, operated by security-cleared German personnel.
- No foreign intelligence service (FISA, Cloud Act, etc.) jurisdiction may apply to any component.
- Network traffic carrying VS-NfD data must not leave German territory, not even in transit.
- Hardware supply chain should be documented; BSI-approved hardware is preferred.

### 2.4 Personnel Requirements

- All administrators with access to VS-NfD systems require a **Sicherheitsueberpruefung (SUe1)** minimum, pursuant to the Sicherheitsueberpruefungsgesetz (SUeG).
- A dedicated **IT-Sicherheitsbeauftragte/r (IT-SiBe)** must be appointed.
- A **Geheimschutzbeauftragte/r** must oversee VS-NfD handling.

---

## 3. Threat Model and Security Objectives

### 3.1 Threat Actors

| Threat Actor | Capability | Relevance to VS-NfD |
|---|---|---|
| Foreign intelligence services | Very High | Primary threat |
| Organized cybercrime | High | Secondary threat |
| Insider threats | Medium-High | Primary threat |
| Hacktivists | Medium | Secondary threat |
| Supply chain compromise | High | Primary threat |

### 3.2 Security Objectives (Schutzziele)

Per BSI IT-Grundschutz, mapped to VS-NfD requirements:

| Objective | Protection Level | Description |
|---|---|---|
| **Vertraulichkeit (Confidentiality)** | Hoch (High) | VS-NfD data must be protected from unauthorized disclosure |
| **Integritaet (Integrity)** | Hoch (High) | Data and systems must be protected from unauthorized modification |
| **Verfuegbarkeit (Availability)** | Normal to Hoch | Service availability per agency SLA requirements |
| **Authentizitaet (Authenticity)** | Hoch (High) | Identity of all actors must be verifiable |
| **Nichtabstreitbarkeit (Non-repudiation)** | Hoch (High) | All actions on VS-NfD data must be attributable |

### 3.3 Attack Vectors Considered

- Network-based intrusion (perimeter and lateral movement)
- Hypervisor escape / VM breakout
- Supply chain attacks (hardware, firmware, software dependencies)
- Insider abuse of administrative privileges
- Side-channel attacks on shared compute resources
- Cryptographic attacks on data at rest and in transit
- Physical access to data center infrastructure
- API exploitation of OpenStack control plane

---

## 4. Architecture Overview

### 4.1 High-Level Architecture

```
+================================================================+
|                    ZONE 1: MANAGEMENT ZONE                      |
|  (Physically separated, dedicated management network)           |
|                                                                  |
|  +------------------+  +------------------+  +----------------+ |
|  | Deployment &     |  | Monitoring &     |  | Identity &     | |
|  | Configuration    |  | SIEM             |  | Access Mgmt    | |
|  | (Ansible/Salt)   |  | (Elastic/Wazuh)  |  | (Keycloak+     | |
|  |                  |  |                  |  |  FreeIPA)      | |
|  +------------------+  +------------------+  +----------------+ |
|  +------------------+  +------------------+  +----------------+ |
|  | Secrets Mgmt     |  | Backup Control   |  | PKI / CA       | |
|  | (HashiCorp Vault)|  | (BorgBackup)     |  | (EJBCA/DogTag) | |
|  +------------------+  +------------------+  +----------------+ |
+================================================================+
                              |
                    (Dedicated Mgmt Network)
                              |
+================================================================+
|                   ZONE 2: OPENSTACK CONTROL PLANE               |
|  (Dedicated hosts, not co-located with workloads)               |
|                                                                  |
|  +------------------+  +------------------+  +----------------+ |
|  | Keystone         |  | Nova API/        |  | Neutron        | |
|  | (AuthN/AuthZ)    |  | Scheduler/       |  | (Networking)   | |
|  |                  |  | Conductor        |  |                | |
|  +------------------+  +------------------+  +----------------+ |
|  +------------------+  +------------------+  +----------------+ |
|  | Glance           |  | Cinder           |  | Barbican       | |
|  | (Images)         |  | (Block Storage)  |  | (Key Mgmt)     | |
|  +------------------+  +------------------+  +----------------+ |
|  +------------------+  +------------------+  +----------------+ |
|  | Heat / Magnum    |  | Octavia          |  | Horizon        | |
|  | (Orchestration)  |  | (Load Balancing) |  | (Dashboard)    | |
|  +------------------+  +------------------+  +----------------+ |
|  +------------------+  +------------------+                     |
|  | Placement        |  | MariaDB Galera + |                     |
|  |                  |  | RabbitMQ Cluster |                     |
|  +------------------+  +------------------+                     |
+================================================================+
                              |
                    (Internal API Network)
                              |
+================================================================+
|                   ZONE 3: COMPUTE / DATA PLANE                  |
|  (Hypervisors running tenant workloads)                         |
|                                                                  |
|  +----------------------------------------------------------+  |
|  | Compute Nodes (KVM/libvirt on hardened Linux)             |  |
|  | - LUKS-encrypted local storage                            |  |
|  | - vTPM for guest VMs                                      |  |
|  | - SELinux/sVirt mandatory access control                  |  |
|  | - Secure Boot enabled                                     |  |
|  +----------------------------------------------------------+  |
|  +----------------------------------------------------------+  |
|  | Ceph Storage Cluster (encrypted OSDs)                     |  |
|  | - Separate cluster & public network                       |  |
|  | - Erasure coding + replication                            |  |
|  | - LUKS on all OSD drives                                  |  |
|  +----------------------------------------------------------+  |
+================================================================+
                              |
                    (Tenant Data Network)
                              |
+================================================================+
|                   ZONE 4: PERIMETER / DMZ                       |
|  (BSI-approved firewalls, IDS/IPS)                              |
|                                                                  |
|  +------------------+  +------------------+  +----------------+ |
|  | BSI-approved     |  | Reverse Proxy /  |  | VPN Gateway    | |
|  | Firewall (HA)    |  | WAF              |  | (BSI-approved) | |
|  +------------------+  +------------------+  +----------------+ |
+================================================================+
```

### 4.2 Zone Separation Philosophy

The architecture implements **strict zone separation** following BSI IT-Grundschutz building block **NET.1.1 (Netzarchitektur und -design)**:

- **Zone 1 (Management):** Highest trust level. Accessible only from dedicated management terminals in physically secured rooms. No direct internet connectivity.
- **Zone 2 (Control Plane):** OpenStack API and services. Accessible from Zone 1 (management) and Zone 3 (compute nodes for service communication). No direct user access.
- **Zone 3 (Data Plane):** Compute and storage resources. Tenant workload isolation via hypervisor, network segmentation, and mandatory access control.
- **Zone 4 (Perimeter):** Internet-facing components (if any). Strictly minimized. BSI-approved firewalls mandatory.

Each zone is separated by **dedicated physical firewalls** (not just VLANs). Inter-zone traffic is encrypted and authenticated.

---

## 5. Hardware and Physical Infrastructure

### 5.1 Data Center Requirements

Per the Geheimschutzhandbuch and BSI IT-Grundschutz building block **INF.2 (Rechenzentrum)**:

| Requirement | Specification |
|---|---|
| **Location** | Germany, federal or certified operator premises |
| **Physical security** | EN 50600 Class 3 minimum, or BSI-certified data center |
| **Access control** | Multi-factor physical access, man-traps, 24/7 video surveillance |
| **Zoning** | VS-NfD processing in dedicated, physically separated security zone |
| **Power** | Redundant power feeds, UPS, diesel generator (N+1) |
| **Cooling** | Redundant cooling (N+1), temperature/humidity monitoring |
| **Fire protection** | Gas-based fire suppression (no water in server rooms) |
| **Personnel** | All data center staff with SUe1 clearance minimum |

**Recommended data center operators (examples):**
- BWI (Bundeswehr IT-Dienstleister) -- already VS-NfD accredited
- FITKO-certified data centers
- ITZBund (Informationstechnikzentrum Bund) -- federal IT service center
- Agency-operated on-premises data center (if available and certifiable)

### 5.2 Server Hardware

| Component | Recommendation | Rationale |
|---|---|---|
| **Server platform** | European-manufactured or BSI-evaluated servers | Supply chain trust |
| **CPU** | AMD EPYC (with SEV-SNP) or Intel Xeon (with TDX) | Hardware-assisted VM isolation, memory encryption |
| **TPM** | TPM 2.0 (BSI-certified, e.g., Infineon SLB 9670) | Measured boot, key storage, platform integrity |
| **Firmware** | UEFI Secure Boot with custom KEK/DB | Prevent firmware-level tampering |
| **BMC/IPMI** | Dedicated, isolated management network; OpenBMC preferred | Reduce attack surface of out-of-band management |
| **Network** | 25/100 GbE with hardware offload | Performance for encrypted storage traffic |
| **Storage** | NVMe SSDs with hardware encryption (OPAL 2.0) | Performance + additional encryption layer |

**Note on hardware supply chain:** The BSI does not (as of 2026) mandate specific server vendors for VS-NfD, but the agency should document the hardware supply chain per BSI IT-Grundschutz building block **ORP.4 (Identitaets- und Berechtigungsmanagement)** and conduct a risk assessment per **ORP.1**. European vendors such as ATOS (Bull), Fujitsu (Augsburg), or certified configurations from Dell/HPE with verified supply chains are advisable.

### 5.3 Network Hardware

| Component | Recommendation | BSI Requirement |
|---|---|---|
| **Perimeter Firewall** | **BSI-approved product mandatory** (e.g., genugate by genua, Lancom R&S) | VS-NfD firewalls must be from the BSI list of approved products |
| **Internal Firewalls** | BSI-approved or evaluated products strongly recommended | High protection requirement zones |
| **Switches** | Managed switches with 802.1X, MACsec capable | Grundschutz NET.2.1 |
| **VPN Concentrator** | **BSI-approved VPN product mandatory** (e.g., genua genuconnect, SINA Box) | VS-NfD VPN requires BSI-approved products per VSA |

---

## 6. Network Architecture

### 6.1 Network Segments

```
+-------------------------------------------------------------------+
|                        PHYSICAL NETWORK LAYOUT                     |
+-------------------------------------------------------------------+
|                                                                     |
|  Network 1: Management (MGMT)                                     |
|  - VLAN 100, Subnet 10.100.0.0/24                                 |
|  - Purpose: BMC/IPMI, OS management, deployment                   |
|  - Access: Management terminals only                               |
|  - Encryption: MACsec (IEEE 802.1AE)                              |
|                                                                     |
|  Network 2: OpenStack Internal API                                 |
|  - VLAN 200, Subnet 10.200.0.0/24                                 |
|  - Purpose: Inter-service communication (Keystone, Nova, etc.)    |
|  - Access: Control plane hosts only                                |
|  - Encryption: TLS 1.3 mutual authentication                      |
|                                                                     |
|  Network 3: OpenStack Public API                                   |
|  - VLAN 300, Subnet 10.300.0.0/24                                 |
|  - Purpose: User/tenant access to OpenStack APIs                  |
|  - Access: Via firewall from authorized networks                   |
|  - Encryption: TLS 1.3                                             |
|                                                                     |
|  Network 4: Tenant Data (Overlay)                                  |
|  - VLAN 400, Subnet 10.400.0.0/16                                 |
|  - Purpose: VM-to-VM traffic (VXLAN/Geneve overlay)               |
|  - Access: Compute and network nodes                               |
|  - Encryption: IPsec or WireGuard tunnel encryption                |
|                                                                     |
|  Network 5: Storage (Ceph Public)                                  |
|  - VLAN 500, Subnet 10.500.0.0/24                                 |
|  - Purpose: Ceph client-to-OSD traffic                             |
|  - Access: Compute nodes and Ceph OSDs                             |
|  - Encryption: Ceph messenger v2 (on-wire encryption)              |
|                                                                     |
|  Network 6: Storage Replication (Ceph Cluster)                     |
|  - VLAN 600, Subnet 10.600.0.0/24                                 |
|  - Purpose: Ceph OSD-to-OSD replication                            |
|  - Access: Ceph OSD nodes only                                     |
|  - Encryption: Ceph messenger v2 encryption                        |
|                                                                     |
|  Network 7: External/DMZ (if required)                             |
|  - Dedicated physical interface, routed through BSI-approved FW    |
|  - Purpose: Minimal external connectivity                          |
|  - Encryption: BSI-approved VPN                                    |
|                                                                     |
+-------------------------------------------------------------------+
```

### 6.2 Firewall Rules Philosophy

- **Default deny** on all zone boundaries.
- **Allowlisting only** -- every permitted flow is explicitly documented.
- All inter-zone traffic passes through stateful inspection firewalls.
- No direct path from any external network to Zone 2 (Control Plane) or Zone 3 (Data Plane).
- All management access via dedicated jump hosts in Zone 1 with multi-factor authentication.

### 6.3 Microsegmentation

Within Zone 3, tenant isolation is enforced at multiple layers:

1. **Neutron Security Groups:** Per-port stateful firewall rules.
2. **OVN/OVS with IPsec:** Encrypted overlay tunnels between compute nodes.
3. **SELinux/sVirt:** Mandatory access control at the hypervisor level, preventing VM-to-VM interference even in case of hypervisor bugs.
4. **VXLAN/Geneve segmentation:** Separate broadcast domains per tenant.

### 6.4 DNS and NTP

- **DNS:** Internal DNS servers only (no external resolution from VS-NfD zones). DNSSEC mandatory. Use Unbound or BIND with DNSSEC validation.
- **NTP:** Dedicated internal NTP servers synchronized to DCF77 (PTB Braunschweig) or GPS with holdover. Critical for log correlation and forensics. Use chrony. NTP authentication (NTS) mandatory per BSI recommendations.

---

## 7. OpenStack Component Architecture

### 7.1 OpenStack Release and Deployment

| Parameter | Recommendation |
|---|---|
| **OpenStack Release** | Latest LTS-equivalent (e.g., 2025.2 "Epoxy" or current stable) |
| **Deployment Tool** | Kolla-Ansible or OpenStack-Ansible (FLOSS) |
| **OS Base** | Hardened Linux: RHEL 9 / AlmaLinux 9 (DISA STIG available) or Debian 12 (BSI SiSyPHuS profile) |
| **Container Runtime** | Podman (daemonless, rootless where possible) |
| **HA Strategy** | 3-node control plane with HAProxy + Keepalived |

**Note on operating system choice:** The BSI has conducted security analyses of Linux distributions under the **SiSyPHuS** project (currently focused on RHEL and derivatives). Using an OS with BSI-analyzed security configurations provides documented compliance evidence.

### 7.2 Component Selection and Configuration

#### 7.2.1 Keystone (Identity Service)

| Aspect | Configuration |
|---|---|
| **Backend** | LDAP integration with FreeIPA or agency Active Directory |
| **Token format** | Fernet tokens (no persistent token storage) |
| **MFA** | Mandatory for all administrative access (TOTP + certificate) |
| **Federation** | SAML 2.0 via Keycloak for cross-agency federation |
| **Policy** | Custom policy.yaml with principle of least privilege |
| **Audit** | CADF audit middleware enabled, all events to SIEM |

#### 7.2.2 Nova (Compute Service)

| Aspect | Configuration |
|---|---|
| **Hypervisor** | KVM/libvirt (FLOSS, BSI-evaluated under SiSyPHuS) |
| **VM isolation** | SELinux sVirt (mandatory), libvirt security driver |
| **vTPM** | Enabled for all VS-NfD guest VMs (swtpm) |
| **Live migration** | Encrypted live migration (TLS) over dedicated network |
| **CPU pinning** | Enabled to prevent side-channel attacks |
| **NUMA topology** | Configured for performance and isolation |
| **Overcommit** | Disabled for VS-NfD workloads (1:1 allocation) |
| **Secure Boot** | UEFI guests with Secure Boot by default |
| **Memory encryption** | AMD SEV-SNP or Intel TDX where available |

#### 7.2.3 Neutron (Networking Service)

| Aspect | Configuration |
|---|---|
| **Backend** | OVN (preferred) or ML2/OVS |
| **Overlay** | Geneve with IPsec encryption |
| **DVR** | Distributed routing for performance |
| **Security Groups** | Stateful, default-deny |
| **FWaaS** | Neutron FWaaS v2 for project-level firewall rules |
| **RBAC** | Network RBAC for cross-project sharing control |
| **Port security** | Anti-spoofing enabled (default) |

#### 7.2.4 Cinder (Block Storage)

| Aspect | Configuration |
|---|---|
| **Backend** | Ceph RBD (primary), optionally LVM for local ephemeral |
| **Encryption** | Cinder volume encryption enabled (LUKS via dm-crypt) |
| **Key management** | Keys stored in Barbican (see Section 9) |
| **Replication** | Ceph handles replication (3x replication or EC 4+2) |

#### 7.2.5 Glance (Image Service)

| Aspect | Configuration |
|---|---|
| **Backend** | Ceph RBD |
| **Image signing** | Mandatory image signature verification enabled |
| **Image encryption** | Encrypted image support via Barbican |
| **Allowed formats** | Restricted to qcow2 and raw (no proprietary formats) |
| **Image sources** | Only agency-approved, hardened base images |

#### 7.2.6 Barbican (Key Management Service)

| Aspect | Configuration |
|---|---|
| **Backend** | HSM (Hardware Security Module) -- **see Section 9 for BSI requirements** |
| **PKCS#11** | HSM integration via PKCS#11 plugin |
| **Key types** | AES-256 for volume encryption, RSA-4096 / ECC P-384 for asymmetric |
| **Access control** | Per-project key isolation, RBAC enforced |

#### 7.2.7 Octavia (Load Balancing)

| Aspect | Configuration |
|---|---|
| **Mode** | Amphora-based (dedicated LB VMs per tenant) |
| **TLS** | TLS 1.3 termination with Barbican-managed certificates |
| **Isolation** | Dedicated management network for amphora instances |

#### 7.2.8 Heat (Orchestration)

| Aspect | Configuration |
|---|---|
| **Purpose** | Infrastructure-as-Code for reproducible deployments |
| **Security** | Template validation, parameter constraints |
| **Integration** | With Barbican for secret injection |

#### 7.2.9 Horizon (Dashboard)

| Aspect | Configuration |
|---|---|
| **Access** | HTTPS only (TLS 1.3), accessible only from internal network |
| **Authentication** | Via Keystone with MFA |
| **Session** | Short timeouts (15 minutes), secure cookie attributes |
| **CSP** | Strict Content-Security-Policy headers |
| **Recommendation** | Consider restricting to CLI/API only for VS-NfD operations |

### 7.3 Supporting Services

| Service | Technology | FLOSS? |
|---|---|---|
| **Database** | MariaDB Galera Cluster (3 nodes) | Yes |
| **Message Queue** | RabbitMQ (clustered, TLS, mTLS) | Yes |
| **Cache** | Memcached or Redis (TLS, authentication) | Yes |
| **Load Balancer (internal)** | HAProxy + Keepalived | Yes |
| **DNS** | Designate + PowerDNS or BIND | Yes |
| **Container Registry** | Harbor (for container images if needed) | Yes |

---

## 8. FLOSS vs. BSI-Approved Products Decision Matrix

This is the critical section addressing the question of whether FLOSS can be used throughout or whether BSI-approved products are required at certain layers.

### 8.1 Decision Framework

The VSA (Verschlusssachenanweisung) and BSI technical guidelines establish clear requirements for certain product categories. The decision tree is:

1. **Is there a BSI requirement for an "approved product" (zugelassenes Produkt) at this layer?**
   - If yes: BSI-approved product is **mandatory**. No FLOSS alternative is acceptable unless it has BSI approval.
2. **Is there a BSI requirement for a "certified product" (zertifiziertes Produkt)?**
   - If yes: A product with BSI certification or Common Criteria certification (recognized by BSI) is **strongly recommended**. FLOSS can be used if it has obtained certification.
3. **Is there a BSI recommendation or evaluation?**
   - If yes: The BSI-evaluated FLOSS component may be used with appropriate hardening.
4. **Neither?**
   - FLOSS can be used with documented risk assessment and hardening per IT-Grundschutz.

### 8.2 Layer-by-Layer Analysis

| Layer | FLOSS Possible? | BSI-Approved Required? | Recommendation | Rationale |
|---|---|---|---|---|
| **Perimeter Firewall** | **No** | **Yes, mandatory** | genugate (genua GmbH), Lancom R&S Unified Firewall | VSA requires BSI-approved firewalls for VS-NfD network boundaries. BSI maintains list of "zugelassene IT-Sicherheitsprodukte" |
| **VPN Gateway** | **No** | **Yes, mandatory** | SINA Box (secunet), genuconnect (genua), NCP Secure VPN | VS-NfD data in transit over untrusted networks requires BSI-approved VPN. Per BSI TL-03402 |
| **Hardware Security Module (HSM)** | **No** | **Yes, mandatory** | Utimaco SecurityServer, Thales Luna (with BSI certification) | Cryptographic key material for VS-NfD must be protected by BSI-certified HSM (CC EAL4+ minimum) |
| **Smartcard / Token (MFA)** | **No** | **Yes, strongly recommended** | BSI-certified smartcards (e.g., Bundesdruckerei) | For VS-NfD admin authentication, BSI-certified authentication tokens recommended |
| **Operating System** | **Yes** | No, but BSI-evaluated preferred | RHEL 9 / AlmaLinux 9 with BSI SiSyPHuS hardening profile | BSI has analyzed RHEL under SiSyPHuS; hardening guides available. FLOSS fully acceptable |
| **Hypervisor (KVM)** | **Yes** | No | KVM/libvirt with SELinux sVirt | KVM is part of the Linux kernel, analyzed by BSI under SiSyPHuS. Fully acceptable for VS-NfD |
| **OpenStack Services** | **Yes** | No | All OpenStack services (Keystone, Nova, Neutron, etc.) | No BSI approval requirement for cloud management software. Must be hardened per Grundschutz |
| **Encryption at Rest** | **Yes (with caveats)** | HSM backend required | LUKS/dm-crypt for volume encryption, keys in BSI-certified HSM | The encryption software (dm-crypt/LUKS) is FLOSS and acceptable. Key management must use certified HSM |
| **Encryption in Transit (internal)** | **Yes** | No (internal networks) | TLS 1.3 (OpenSSL) with BSI-compliant cipher suites | OpenSSL is acceptable for internal TLS. Cipher suites must comply with BSI TR-02102 |
| **Encryption in Transit (external/cross-site)** | **No** | **Yes, mandatory** | BSI-approved VPN (see above) | Any VS-NfD data leaving the secure perimeter requires BSI-approved encryption |
| **Certificate Authority (PKI)** | **Yes** | No (but practices must comply) | EJBCA or DogTag PKI | FLOSS CA is acceptable. Certificate profiles must comply with BSI TR-02103 |
| **SIEM / Log Management** | **Yes** | No | Wazuh + Elasticsearch/OpenSearch | No BSI approval requirement. Must meet Grundschutz OPS.1.1.5 logging requirements |
| **Backup** | **Yes** | No | BorgBackup with encryption | Encryption keys managed via HSM. FLOSS backup tools are acceptable |
| **IDS/IPS** | **Yes (with caveats)** | Recommended at perimeter | Suricata (FLOSS) for internal; BSI-approved for perimeter | Internal IDS can be FLOSS. Perimeter IDS should be BSI-approved or part of approved firewall |
| **Container Runtime** | **Yes** | No | Podman (rootless) | FLOSS acceptable with hardening |
| **Configuration Management** | **Yes** | No | Ansible or SaltStack | FLOSS acceptable |
| **Network Switches** | **Partially** | Recommended | Managed switches from trusted vendors | No strict BSI approval for switches, but MACsec capability and supply chain trust required |
| **TPM** | **No** | **Yes, certified required** | TPM 2.0 with BSI / CC certification | TPM must be CC-certified (e.g., Infineon SLB 9670, CC EAL4+) |

### 8.3 Summary: Mandatory Non-FLOSS Components

The following components **cannot** be replaced by FLOSS alternatives for VS-NfD:

1. **Perimeter firewalls** -- BSI-approved product list (Zulassung)
2. **VPN gateways for cross-site/external links** -- BSI-approved product list
3. **Hardware Security Modules (HSMs)** -- BSI-certified (CC EAL4+)
4. **TPM chips** -- CC-certified hardware
5. **MFA tokens/smartcards for VS-NfD admin access** -- BSI-certified recommended

Everything else in the stack -- from the operating system through the hypervisor, OpenStack services, storage, monitoring, and automation -- **can be FLOSS** with proper hardening and documentation.

### 8.4 BSI-Approved Product Sources

- **BSI list of approved IT security products:** https://www.bsi.bund.de/zulassung
- **BSI-certified products (Common Criteria):** https://www.bsi.bund.de/zertifizierung
- **BSI Technical Guidelines:** https://www.bsi.bund.de/technische-richtlinien

---

## 9. Encryption and Key Management

### 9.1 Cryptographic Requirements (BSI TR-02102)

All cryptographic algorithms and key lengths must comply with BSI TR-02102-1 (general) and TR-02102-2 (TLS).

| Use Case | Algorithm | Key Length | BSI Compliance |
|---|---|---|---|
| Symmetric encryption (data at rest) | AES-256 | 256 bit | Compliant per TR-02102-1 |
| Symmetric encryption (data in transit) | AES-256-GCM / ChaCha20-Poly1305 | 256 bit | Compliant |
| Asymmetric key exchange | ECDHE (brainpoolP384r1 or P-384) | 384 bit | Compliant (BSI prefers Brainpool curves) |
| Digital signatures | ECDSA (brainpoolP384r1) or RSA-PSS | 384 bit / 4096 bit | Compliant |
| Hashing | SHA-384 or SHA-512 | -- | Compliant |
| Key derivation | HKDF-SHA-384 or PBKDF2-SHA-512 | -- | Compliant |
| TLS version | TLS 1.3 only | -- | Mandatory per TR-02102-2 |

**Important BSI-specific note:** The BSI recommends **Brainpool curves** (brainpoolP256r1, brainpoolP384r1) over NIST curves for German government applications. Ensure TLS configurations and certificates support Brainpool curves. OpenSSL 3.x supports Brainpool curves.

### 9.2 Encryption Architecture

```
+------------------------------------------------------------------+
|                    ENCRYPTION LAYERS                               |
+------------------------------------------------------------------+
|                                                                    |
|  Layer 1: Hardware Encryption                                     |
|  - Self-encrypting drives (OPAL 2.0)                              |
|  - TPM-sealed keys                                                 |
|  - AMD SEV-SNP / Intel TDX memory encryption                     |
|                                                                    |
|  Layer 2: Volume/Disk Encryption                                  |
|  - LUKS2 (dm-crypt) on all host OS disks                          |
|  - Cinder volume encryption (LUKS via dm-crypt)                   |
|  - Ceph OSD encryption (LUKS on block devices)                    |
|  - Keys sealed to TPM or stored in HSM via Barbican               |
|                                                                    |
|  Layer 3: Transport Encryption                                    |
|  - TLS 1.3 for all OpenStack API communication                   |
|  - mTLS between OpenStack services                                |
|  - Ceph messenger v2 encryption (on-wire)                         |
|  - IPsec / WireGuard for tenant overlay networks                  |
|  - BSI-approved VPN for cross-site links                          |
|  - MACsec on management network switches                          |
|                                                                    |
|  Layer 4: Application-Level Encryption                            |
|  - Barbican-managed encryption for secrets                        |
|  - Image encryption in Glance                                     |
|  - Database encryption (MariaDB TDE) for sensitive tables         |
|                                                                    |
+------------------------------------------------------------------+
```

### 9.3 Key Management Architecture

```
+------------------------------------------------------------------+
|                                                                    |
|  +-------------------+         +-----------------------------+    |
|  |   BSI-Certified   | PKCS#11 |       Barbican              |    |
|  |   HSM Cluster     |<------->|   (Key Management Service)  |    |
|  |   (Active/Active) |         |                             |    |
|  +-------------------+         +-----------------------------+    |
|          |                              |                          |
|          |                    +---------+---------+                |
|          |                    |         |         |                |
|          v                    v         v         v                |
|  +---------------+    +--------+ +--------+ +--------+           |
|  | TPM Key       |    | Cinder | | Glance | | Nova   |           |
|  | Sealing       |    | Vol    | | Image  | | vTPM   |           |
|  | (Host keys)   |    | Enc    | | Enc    | | Keys   |           |
|  +---------------+    +--------+ +--------+ +--------+           |
|                                                                    |
+------------------------------------------------------------------+
```

**HSM Requirements:**
- Minimum CC EAL4+ certification (BSI-recognized)
- FIPS 140-2 Level 3 or equivalent
- Clustered/HA configuration (at least 2 HSMs)
- Located within the secure data center zone
- Tamper-evident and tamper-responsive
- Key ceremony procedures documented and witnessed

**Recommended HSMs:**
- Utimaco CryptoServer Se (German manufacturer, BSI-certified)
- Utimaco SecurityServer (for larger deployments)
- Thales Luna Network HSM (with relevant BSI/CC certification)

---

## 10. Identity, Authentication, and Access Control

### 10.1 Identity Architecture

```
+------------------------------------------------------------------+
|                                                                    |
|  +------------------+    LDAP    +------------------+             |
|  |    FreeIPA        |<--------->|   Keystone        |             |
|  |  (Identity Store) |           | (OpenStack AuthN) |             |
|  +------------------+            +------------------+             |
|          |                              |                          |
|          | Kerberos                     | SAML 2.0 / OIDC          |
|          v                              v                          |
|  +------------------+            +------------------+             |
|  |  Admin Terminals  |           |    Keycloak       |             |
|  |  (SSH, mgmt)      |           |  (Federation/SSO) |             |
|  +------------------+            +------------------+             |
|                                         |                          |
|                                    (Cross-Agency)                  |
|                                         v                          |
|                                  +------------------+             |
|                                  | Partner Agency   |             |
|                                  | IdP (SAML)       |             |
|                                  +------------------+             |
|                                                                    |
+------------------------------------------------------------------+
```

### 10.2 Authentication Requirements

| Access Type | Authentication Method | MFA Required? |
|---|---|---|
| OpenStack API (admin) | Certificate + TOTP | **Yes, mandatory** |
| OpenStack API (user) | Username/Password + TOTP | **Yes, mandatory** |
| Horizon Dashboard | Username/Password + TOTP | **Yes, mandatory** |
| SSH to management hosts | SSH key + smartcard | **Yes, mandatory** |
| BMC/IPMI access | Dedicated credentials + network isolation | Network-level isolation |
| VPN access | Certificate + smartcard | **Yes, mandatory** |
| Physical data center | Badge + biometric | **Yes, mandatory** |

### 10.3 Role-Based Access Control (RBAC)

OpenStack Keystone RBAC is configured with the following principle:

- **System-scoped roles** for infrastructure administration
- **Domain-scoped roles** for organizational unit management
- **Project-scoped roles** for tenant workload management
- **Principle of least privilege** enforced in all policy.yaml files
- **Separation of duties** between security administration and operations

| Role | Scope | Permissions |
|---|---|---|
| `cloud_admin` | System | Full platform administration (2-person rule for critical actions) |
| `security_admin` | System | Security configuration, audit log access, policy management |
| `network_admin` | System | Network infrastructure management |
| `project_admin` | Project | Full control within assigned project |
| `project_member` | Project | VM and volume management within project |
| `project_reader` | Project | Read-only access to project resources |
| `auditor` | System | Read-only access to all resources and audit logs |

### 10.4 Privileged Access Management (PAM)

- All administrative actions via **jump hosts** in the management zone.
- Session recording for all administrative sessions (using `tlog` or similar).
- 4-eyes principle for critical operations (e.g., HSM key ceremonies, security policy changes).
- Time-limited elevated privileges (just-in-time access).
- Administrative credentials stored in HashiCorp Vault with audit trail.

---

## 11. Logging, Monitoring, and SIEM

### 11.1 Logging Architecture

Per BSI IT-Grundschutz **OPS.1.1.5 (Protokollierung)** and **DER.1 (Detektion von sicherheitsrelevanten Ereignissen)**:

```
+------------------------------------------------------------------+
|                                                                    |
|  Log Sources:                                                     |
|  - OpenStack services (oslo.log, CADF audit)                     |
|  - Linux system logs (journald, auditd)                           |
|  - Firewall logs (BSI-approved FW)                                |
|  - Network flow data (NetFlow/sFlow)                              |
|  - Hypervisor events (libvirt)                                    |
|  - Ceph cluster logs                                              |
|  - Authentication events (Keystone, FreeIPA, Keycloak)            |
|  - HSM audit logs                                                 |
|  - Physical access control logs                                   |
|                                                                    |
|         | syslog-TLS / Filebeat (encrypted)                       |
|         v                                                          |
|  +------------------+    +------------------+                     |
|  |   Wazuh Manager   |    |  Elasticsearch /  |                     |
|  |   (HIDS + SIEM)   |--->|  OpenSearch       |                     |
|  +------------------+    +------------------+                     |
|         |                         |                                |
|         v                         v                                |
|  +------------------+    +------------------+                     |
|  |  Alert Engine     |    |  Kibana /         |                     |
|  |  (Wazuh Rules +   |    |  OpenSearch       |                     |
|  |   Custom Rules)   |    |  Dashboards       |                     |
|  +------------------+    +------------------+                     |
|         |                                                          |
|         v                                                          |
|  +------------------+                                             |
|  | Incident Response |                                             |
|  | (CERT/CSIRT)      |                                             |
|  +------------------+                                             |
|                                                                    |
+------------------------------------------------------------------+
```

### 11.2 Mandatory Log Events

The following events must be logged with immutable, timestamped records:

- All authentication attempts (success and failure)
- All authorization decisions (CADF audit events)
- All administrative actions
- All changes to security configurations
- All access to VS-NfD classified data/resources
- All network connections crossing zone boundaries
- All cryptographic key operations
- All changes to VM lifecycle (create, delete, migrate, snapshot)
- All changes to network configuration
- All physical access events

### 11.3 Log Retention

| Log Type | Retention Period | Rationale |
|---|---|---|
| Security events | 24 months minimum | BSI Grundschutz / investigation requirements |
| Audit trail | 10 years | Legal compliance (VS-NfD audit trail) |
| Operational logs | 6 months | Troubleshooting |
| Network flow data | 12 months | Forensic analysis |

### 11.4 Log Integrity

- All logs are signed using append-only mechanisms.
- Log storage is write-once (immutable) for the retention period.
- Log integrity is verified by cryptographic hash chains.
- Log storage is on separate infrastructure from production systems.
- Access to log data requires `security_admin` or `auditor` role.

---

## 12. Backup, Recovery, and Business Continuity

### 12.1 Backup Architecture

| Component | Tool | Frequency | Retention |
|---|---|---|---|
| VM disk images | Cinder backup to secondary Ceph | Daily incremental, weekly full | 30 days |
| OpenStack databases | MariaDB logical backup (mysqldump) | Every 4 hours | 14 days |
| Configuration | Git-based (Ansible/Salt state) | On every change | Indefinite |
| Secrets/Keys | HSM key backup (to secondary HSM) | On every change | Per key lifecycle policy |
| Audit logs | Replicated to offline storage | Continuous | Per retention policy |
| Ceph metadata | RBD pool metadata backup | Daily | 30 days |

### 12.2 Backup Encryption

- All backups encrypted with AES-256 before storage.
- Backup encryption keys managed by HSM (separate from primary data keys).
- Backup media (if physical) stored in VS-NfD-approved safe/vault.

### 12.3 Recovery Objectives

| Metric | Target | Notes |
|---|---|---|
| **RPO (Recovery Point Objective)** | 4 hours | Maximum data loss |
| **RTO (Recovery Time Objective)** | 8 hours | Maximum downtime for full platform recovery |
| **Single VM recovery** | 1 hour | Individual workload recovery |

### 12.4 Disaster Recovery

- **Primary site:** Main data center (active)
- **Secondary site:** Geographically separated data center within Germany (standby)
- **Data replication:** Asynchronous Ceph replication (rbd-mirror) to secondary site
- **Network:** BSI-approved VPN link between sites (encrypted, dedicated line preferred)
- **Failover:** Documented manual failover procedure (automated failover increases attack surface)
- **Testing:** DR test conducted at least annually, documented per BSI Grundschutz **DER.4 (Notfallmanagement)**

---

## 13. BSI IT-Grundschutz Compliance Mapping

### 13.1 Relevant IT-Grundschutz Building Blocks (Bausteine)

The following building blocks from the BSI IT-Grundschutz Kompendium are directly applicable:

#### Process-Level Building Blocks (ISMS, ORP, CON, OPS, DER)

| Building Block | Title | Applicability |
|---|---|---|
| **ISMS.1** | Sicherheitsmanagement | Mandatory -- ISMS foundation |
| **ORP.1** | Organisation | Mandatory -- organizational security measures |
| **ORP.2** | Personal | Mandatory -- personnel security |
| **ORP.3** | Sensibilisierung und Schulung | Mandatory -- awareness and training |
| **ORP.4** | Identitaets- und Berechtigungsmanagement | Mandatory -- IAM |
| **CON.1** | Kryptokonzept | Mandatory -- cryptographic concept |
| **CON.2** | Datenschutz | Mandatory -- data protection |
| **CON.3** | Datensicherungskonzept | Mandatory -- backup concept |
| **CON.6** | Loeschen und Vernichten | Mandatory -- secure deletion |
| **OPS.1.1.2** | Ordnungsgemaesse IT-Administration | Mandatory -- proper IT admin |
| **OPS.1.1.3** | Patch- und Aenderungsmanagement | Mandatory -- patch management |
| **OPS.1.1.5** | Protokollierung | Mandatory -- logging |
| **OPS.1.1.6** | Software-Tests und -Freigaben | Mandatory -- software testing |
| **OPS.1.2.4** | Telearbeit | If remote admin is permitted |
| **OPS.1.2.5** | Fernwartung | Mandatory -- remote maintenance |
| **OPS.2.2** | Cloud-Nutzung | Mandatory for cloud architecture |
| **DER.1** | Detektion von sicherheitsrelevanten Ereignissen | Mandatory -- detection |
| **DER.2.1** | Behandlung von Sicherheitsvorfaellen | Mandatory -- incident handling |
| **DER.4** | Notfallmanagement | Mandatory -- emergency management |

#### System-Level Building Blocks (APP, SYS, NET, INF)

| Building Block | Title | Applicability |
|---|---|---|
| **APP.4.4** | Kubernetes (if used) | Conditional |
| **SYS.1.1** | Allgemeiner Server | Mandatory for all servers |
| **SYS.1.3** | Server unter Linux und Unix | Mandatory -- Linux servers |
| **SYS.1.5** | Virtualisierung | Mandatory -- hypervisor layer |
| **SYS.1.6** | Containerisierung | If containers used |
| **SYS.1.8** | Speicherloesungen | Mandatory -- storage (Ceph) |
| **SYS.1.9** | Terminalserver | If jump hosts are terminal servers |
| **NET.1.1** | Netzarchitektur und -design | Mandatory -- network architecture |
| **NET.1.2** | Netzmanagement | Mandatory -- network management |
| **NET.3.1** | Router und Switches | Mandatory |
| **NET.3.2** | Firewall | Mandatory -- firewalls |
| **NET.3.3** | VPN | Mandatory -- VPN |
| **NET.4.1** | TLS | Mandatory -- TLS configuration |
| **INF.1** | Allgemeines Gebaeude | Mandatory -- building security |
| **INF.2** | Rechenzentrum sowie Serverraum | Mandatory -- data center security |

### 13.2 Protection Level Determination (Schutzbedarfsfeststellung)

For VS-NfD processing, the protection requirements are:

| Asset Category | Confidentiality | Integrity | Availability |
|---|---|---|---|
| VS-NfD data at rest | **Hoch** | **Hoch** | Normal-Hoch |
| VS-NfD data in transit | **Hoch** | **Hoch** | Normal-Hoch |
| OpenStack control plane | Hoch | **Hoch** | **Hoch** |
| Identity/authentication system | **Hoch** | **Hoch** | **Hoch** |
| Cryptographic keys | **Sehr hoch** | **Sehr hoch** | Hoch |
| Audit logs | Hoch | **Sehr hoch** | Hoch |
| Network infrastructure | Hoch | Hoch | **Hoch** |

### 13.3 IT-Grundschutz Approach

Given that VS-NfD data has **high** and in some cases **very high** protection requirements, a **standard IT-Grundschutz approach is not sufficient.** The agency must:

1. Conduct a full **Schutzbedarfsfeststellung** (protection needs assessment).
2. Model the information domain (Informationsverbund) in a BSI-compliant tool (e.g., verinice).
3. Implement all applicable building blocks with their requirements (Anforderungen).
4. For "Hoch" (high) protection needs, implement all BASIS, STANDARD, and **additional requirements for high protection** (Anforderungen bei erhoehtem Schutzbedarf).
5. Conduct a **supplementary risk analysis** (ergaenzende Risikoanalyse) per BSI Standard 200-3 for all areas where standard building block measures are insufficient.
6. Document residual risks and obtain formal acceptance from the agency's IT-SiBe and management.

---

## 14. C5 Attestation Roadmap

### 14.1 C5 Overview

The BSI Cloud Computing Compliance Criteria Catalogue (C5:2020) defines 17 categories with a total of approximately 121 criteria. For federal agencies, C5 attestation has become effectively mandatory per BMI guidance.

Even though this is a **private cloud** (not a commercial offering), undergoing C5 attestation demonstrates compliance and may be required if other agencies consume services from this platform.

### 14.2 C5 Criteria Categories

| # | Category | Key Requirements | Architecture Mapping |
|---|---|---|---|
| 1 | **Organisation der Informationssicherheit (OIS)** | ISMS, security organization | ISMS.1, ORP.1 |
| 2 | **Sicherheitsrichtlinien und Arbeitsanweisungen (SP)** | Security policies | Documented policies |
| 3 | **Personal (HR)** | Background checks, training | SUe1 clearance, ORP.2, ORP.3 |
| 4 | **Asset Management (AM)** | Inventory, classification | CMDB, verinice |
| 5 | **Physische Sicherheit (PS)** | Data center security | INF.2, physical security zones |
| 6 | **Regelbetrieb (OPS)** | Operations, change management | Ansible automation, change control |
| 7 | **Identitaets- und Berechtigungsmanagement (IDM)** | IAM, MFA, RBAC | Keystone, FreeIPA, Keycloak |
| 8 | **Kryptographie und Schluesselmanagement (CRY)** | Encryption, key management | Barbican + HSM, TLS, LUKS |
| 9 | **Kommunikationssicherheit (COS)** | Network security | Zone separation, BSI firewalls |
| 10 | **Portabilitaet und Interoperabilitaet (PI)** | Vendor lock-in avoidance | OpenStack = open APIs, FLOSS stack |
| 11 | **Beschaffung, Entwicklung und Aenderung (DEV)** | Secure SDLC, change management | GitOps, CI/CD with security gates |
| 12 | **Steuerung und Ueberwachung von Dienstleistern (SSO)** | Supply chain management | Vendor assessment, SLA monitoring |
| 13 | **Umgang mit Sicherheitsvorfaellen (SIM)** | Incident management | Wazuh, incident response process |
| 14 | **Kontinuitaet des Geschaeftsbetriebs und Notfallmanagement (BCM)** | Business continuity | DR plan, backup/recovery |
| 15 | **Compliance (COM)** | Regulatory compliance | This document + ongoing compliance |
| 16 | **Umgang mit Ermittlungsanfragen (INQ)** | Law enforcement requests | Process documented, German law only |
| 17 | **Produktsicherheit (PSS)** | Product security | Vulnerability management |

### 14.3 C5 Attestation Process

**Phase 1: Preparation (6-9 months)**
1. Appoint a C5 project manager.
2. Engage a Wirtschaftspruefungsgesellschaft (WPG -- auditing firm) experienced in C5 audits (e.g., PwC, KPMG, Deloitte, BDO, or specialized firms like HiSolutions).
3. Conduct a **gap analysis** against all C5 criteria.
4. Remediate identified gaps.
5. Prepare documentation:
   - System description (Systembeschreibung)
   - Security concept (Sicherheitskonzept)
   - Control descriptions for all 17 categories
   - Evidence collection

**Phase 2: Type 1 Attestation (3-6 months)**
- Auditor assesses design and implementation of controls at a **point in time**.
- Results in a **C5 Type 1 report** (SOC 2-equivalent).
- This is the initial attestation.

**Phase 3: Type 2 Attestation (6-12 months after Type 1)**
- Auditor assesses **operating effectiveness** of controls over a period (typically 6-12 months).
- Results in a **C5 Type 2 report**.
- This is the definitive attestation.
- Must be **renewed annually**.

### 14.4 C5 Attestation-Specific Advantages of This Architecture

| C5 Criterion | Advantage |
|---|---|
| Transparency (all categories) | FLOSS components provide full source code auditability |
| Data location (PS-05, PS-06) | On-premises in Germany, fully under agency control |
| Vendor independence (PI-01 to PI-04) | OpenStack + FLOSS avoids vendor lock-in; standard APIs |
| Encryption (CRY-01 to CRY-04) | End-to-end encryption with BSI-certified HSM |
| Access control (IDM-01 to IDM-07) | Multi-layer RBAC with MFA and audit trail |

---

## 15. Accreditation Process Navigation

### 15.1 Overview of the Accreditation Path

For a Bundesbehoerde deploying a VS-NfD-capable private cloud, the accreditation involves multiple parallel tracks:

```
+------------------------------------------------------------------+
|                    ACCREDITATION TIMELINE                          |
+------------------------------------------------------------------+
|                                                                    |
|  Month 1-3: Foundation                                            |
|  +------------------------------------------------------------+  |
|  | - Appoint IT-SiBe, Geheimschutzbeauftragte/r               |  |
|  | - Engage BSI early (Vorabstimmung/Beratung)                |  |
|  | - Establish ISMS per BSI Standard 200-1                    |  |
|  | - Begin Strukturanalyse and Schutzbedarfsfeststellung       |  |
|  | - Procure BSI-approved products (long lead times!)          |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Month 3-9: Design and Implementation                             |
|  +------------------------------------------------------------+  |
|  | - Complete architecture and security concept                |  |
|  | - Model Informationsverbund in verinice                     |  |
|  | - Implement building block requirements                     |  |
|  | - Deploy infrastructure (hardware, network, OpenStack)      |  |
|  | - Implement all security controls                           |  |
|  | - Begin documentation of all controls                       |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Month 9-12: Internal Audit and Remediation                       |
|  +------------------------------------------------------------+  |
|  | - Conduct internal audit against IT-Grundschutz             |  |
|  | - Perform supplementary risk analysis (BSI Standard 200-3) |  |
|  | - Remediate findings                                        |  |
|  | - Conduct penetration test (BSI-certified provider)         |  |
|  | - Prepare Realisierungsplan for residual risks              |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Month 12-15: BSI Audit / Certification                           |
|  +------------------------------------------------------------+  |
|  | - Submit documentation to BSI (if ISO 27001 on             |  |
|  |   IT-Grundschutz certification is pursued)                 |  |
|  | - BSI audit (Dokumentenpruefung + Vor-Ort-Audit)            |  |
|  | - Address audit findings                                    |  |
|  | - Receive ISO 27001 certificate on IT-Grundschutz basis     |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Month 12-18: C5 Attestation (parallel track)                    |
|  +------------------------------------------------------------+  |
|  | - Engage WPG for C5 audit                                   |  |
|  | - C5 Type 1 attestation                                     |  |
|  | - Begin operating period for Type 2                         |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Month 18-24: VS-NfD Freigabe                                    |
|  +------------------------------------------------------------+  |
|  | - Submit Sicherheitskonzept to BSI for VS-NfD Freigabe     |  |
|  | - BSI review and potential on-site inspection               |  |
|  | - Formal VS-NfD accreditation (Freigabe/Betriebserlaubnis) |  |
|  | - Begin operational use for VS-NfD data                     |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  Ongoing:                                                         |
|  +------------------------------------------------------------+  |
|  | - Annual C5 Type 2 re-attestation                           |  |
|  | - Triennial ISO 27001 re-certification                     |  |
|  | - Continuous monitoring and improvement                     |  |
|  | - Regular BSI coordination meetings                         |  |
|  +------------------------------------------------------------+  |
|                                                                    |
+------------------------------------------------------------------+
```

### 15.2 Critical Success Factors for BSI Accreditation

1. **Engage BSI early and often.** The BSI offers advisory services (Beratung) for federal agencies. Initiating contact at project start can prevent costly missteps. Request a "Vorabstimmung" (preliminary coordination).

2. **Use verinice.** The BSI's recommended ISMS tool (verinice, FLOSS edition or verinice.PRO) is the de facto standard for documenting IT-Grundschutz compliance. Auditors expect verinice exports.

3. **Document everything.** German accreditation is documentation-heavy. Every control must be documented with:
   - Description of the measure
   - Implementation evidence
   - Responsible person
   - Review date
   - Effectiveness assessment

4. **Prepare for BSI-approved product lead times.** BSI-approved firewalls and VPN products can have procurement lead times of 3-6 months. Start procurement early.

5. **Hire experienced consultants.** BSI IT-Grundschutz audits are specialized. Engage IT-Grundschutz-Auditoren (certified auditors) for the internal pre-audit.

6. **Penetration testing.** Conduct penetration tests through a BSI-certified service provider (BSI-zertifizierter IS-Penetrationstester). Results feed into the supplementary risk analysis.

7. **VS-NfD-specific requirements.** Beyond standard IT-Grundschutz, the VS-NfD Freigabe requires:
   - Formal security concept (Sicherheitskonzept) per VS-NfD IT-Richtlinie
   - Use of BSI-approved products where mandated
   - Personnel clearances (SUe1) for all administrators
   - Physical security per Geheimschutzhandbuch
   - Incident reporting to BSI CERT-Bund

### 15.3 BSI Contact Points

| Purpose | BSI Division |
|---|---|
| General IT-Grundschutz consultation | Referat WG 14 (IT-Grundschutz) |
| VS-NfD accreditation | Referat S 13 (VS-IT) |
| Cryptographic approval | Referat KT 12 (Kryptotechnik) |
| C5 attestation questions | Referat WG 15 (Cloud-Sicherheit) |
| Incident reporting | CERT-Bund |
| Product certification | Referat SZ 14 (Zertifizierung) |

---

## 16. Operational Processes and Governance

### 16.1 Change Management

- All changes classified as Standard, Normal, or Emergency per ITIL framework.
- Normal and Emergency changes require documented approval from IT-SiBe.
- All infrastructure changes via GitOps (Ansible/Salt in Git, reviewed via merge requests).
- No manual changes to production systems (immutable infrastructure approach for control plane).
- Change Advisory Board (CAB) meets weekly.

### 16.2 Patch Management

Per BSI Grundschutz **OPS.1.1.3 (Patch- und Aenderungsmanagement)**:

| Component | Patch Frequency | Testing Required | Emergency Patch SLA |
|---|---|---|---|
| OS security patches | Within 7 days of release | Staging environment test | 24 hours for critical (CVSS >= 9.0) |
| OpenStack security patches | Within 14 days of OSSA | Staging environment test | 48 hours for critical |
| BSI-approved products (FW, VPN) | Per vendor recommendation | Staging/lab test | Per vendor advisory |
| Firmware updates | Quarterly maintenance window | Lab test | 72 hours for critical |

### 16.3 Vulnerability Management

- Continuous vulnerability scanning using OpenVAS (FLOSS).
- Weekly automated scans of all infrastructure.
- Integration with CERT-Bund vulnerability feeds.
- OpenStack Security Advisories (OSSA) monitored and triaged.
- CVE tracking and remediation SLAs.

### 16.4 Incident Response

Per BSI Grundschutz **DER.2.1 (Behandlung von Sicherheitsvorfaellen)**:

1. **Detection:** Wazuh/SIEM alerts, user reports, BSI CERT-Bund notifications.
2. **Classification:** Severity levels (critical/high/medium/low) with defined response times.
3. **Containment:** Isolation procedures for compromised components (documented playbooks).
4. **Eradication:** Root cause analysis and remediation.
5. **Recovery:** Restoration from known-good state.
6. **Lessons learned:** Post-incident review, process improvement.

**VS-NfD incident reporting:** Security incidents involving VS-NfD data must be reported to:
- BSI (mandatory)
- BfV (Bundesamt fuer Verfassungsschutz) if espionage suspected
- Agency Geheimschutzbeauftragte/r

### 16.5 Secure Decommissioning

Per BSI Grundschutz **CON.6 (Loeschen und Vernichten)**:

- VS-NfD data must be securely erased using BSI-approved methods.
- Storage media: Cryptographic erasure (destroy encryption keys) + physical destruction (DIN 66399, Security Level H-5 minimum).
- Server hardware: BIOS/firmware reset, TPM clear, physical destruction if decommissioned.
- Documented chain of custody for all decommissioned hardware.

---

## 17. Migration and Deployment Strategy

### 17.1 Phased Deployment

**Phase 0: Lab/Proof of Concept (Month 1-3)**
- Deploy OpenStack in a non-classified lab environment.
- Validate architecture, performance, and security controls.
- Test BSI-approved product integration (firewalls, VPN, HSM).
- Develop Ansible/Salt automation playbooks.

**Phase 1: Staging Environment (Month 3-6)**
- Deploy in the target data center with full security controls.
- Use non-classified test data only.
- Conduct security testing and penetration testing.
- Begin IT-Grundschutz documentation.

**Phase 2: Production - Non-Classified (Month 6-9)**
- Move non-classified workloads to the platform.
- Operate in production mode to build operational maturity.
- Collect evidence for C5 attestation.
- Finalize all security documentation.

**Phase 3: VS-NfD Accreditation (Month 9-18)**
- Submit security concept to BSI.
- Undergo BSI audit.
- Obtain VS-NfD Freigabe.

**Phase 4: VS-NfD Production (Month 18+)**
- Migrate VS-NfD workloads to the platform.
- Begin C5 Type 2 attestation period.

### 17.2 Deployment Automation

```
+------------------------------------------------------------------+
|                    DEPLOYMENT PIPELINE                             |
+------------------------------------------------------------------+
|                                                                    |
|  Git Repository (GitLab CE, on-premises)                          |
|  |                                                                 |
|  +--> Code Review (4-eyes principle for infrastructure code)      |
|  |                                                                 |
|  +--> CI Pipeline (automated testing)                             |
|  |    - YAML linting                                               |
|  |    - Ansible syntax check                                       |
|  |    - Security policy validation                                 |
|  |    - Integration tests in staging                               |
|  |                                                                 |
|  +--> Approval Gate (IT-SiBe sign-off for production)             |
|  |                                                                 |
|  +--> CD Pipeline (Ansible deployment)                            |
|       - Staged rollout (canary/rolling)                            |
|       - Automated rollback on failure                              |
|       - Post-deployment verification                               |
|                                                                    |
+------------------------------------------------------------------+
```

### 17.3 Hardening Checklist Summary

| Component | Hardening Measure |
|---|---|
| Linux OS | CIS Benchmark Level 2 + BSI SiSyPHuS profile |
| SSH | Key-only, no root login, AllowUsers, port change, Fail2ban |
| Kernel | Hardened kernel parameters (sysctl), module loading restricted |
| SELinux | Enforcing mode on all hosts (no exceptions) |
| Firewall (host) | nftables with default-deny, minimal open ports |
| OpenStack APIs | TLS 1.3 only, rate limiting, input validation |
| MariaDB | TLS, least-privilege user accounts, no remote root |
| RabbitMQ | TLS, vhost isolation, credential rotation |
| Ceph | CephX authentication, encryption in transit, encrypted OSDs |
| Containers | Rootless Podman, read-only filesystems, no privileged mode |

---

## 18. Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| R1 | Hypervisor escape from tenant VM | Low | Very High | SELinux sVirt, CPU pinning, no overcommit, AMD SEV-SNP | Low |
| R2 | Compromise of OpenStack control plane | Medium | Very High | Dedicated control plane hosts, mTLS, RBAC, audit logging | Medium-Low |
| R3 | Insider threat (privileged admin) | Medium | Very High | 4-eyes principle, session recording, JIT access, SUe1 clearance | Medium |
| R4 | Supply chain compromise (hardware) | Low-Medium | Very High | European vendors, TPM attestation, Secure Boot, firmware verification | Medium |
| R5 | Supply chain compromise (software) | Medium | High | FLOSS (source auditable), dependency scanning, signed packages | Medium-Low |
| R6 | Cryptographic algorithm compromise | Very Low | Very High | BSI TR-02102 compliance, crypto-agility in design, HSM key protection | Low |
| R7 | Data leakage via side channels | Low | High | CPU pinning, memory encryption (SEV-SNP), no overcommit | Low |
| R8 | Loss of BSI-approved product vendor | Low | High | Documented alternatives, modular architecture | Medium |
| R9 | Operational error causing outage | Medium | Medium | Automation, change management, staging environment | Medium-Low |
| R10 | BSI accreditation delayed/denied | Medium | High | Early BSI engagement, experienced consultants, thorough preparation | Medium-Low |

---

## 19. Appendices

### Appendix A: BSI-Approved Products Reference List (Examples, 2026)

**Note:** Always verify current approvals at https://www.bsi.bund.de/zulassung as the list is updated regularly.

| Category | Product | Vendor | Approval Level |
|---|---|---|---|
| Firewall | genugate | genua GmbH (Bundesdruckerei) | VS-NfD |
| Firewall | LANCOM R&S Unified Firewall | LANCOM Systems (Rohde & Schwarz) | VS-NfD |
| VPN | genuconnect | genua GmbH | VS-NfD |
| VPN | SINA Box | secunet Security Networks AG | VS-NfD (and higher) |
| VPN | NCP Secure VPN GovNet Box | NCP engineering GmbH | VS-NfD |
| Encryption | SINA Workstation | secunet Security Networks AG | VS-NfD (and higher) |
| HSM | CryptoServer Se Gen2 | Utimaco IS GmbH | CC EAL4+ |
| Smartcard | Various | Bundesdruckerei | BSI-certified |

### Appendix B: TLS 1.3 Cipher Suite Configuration

Per BSI TR-02102-2, the following cipher suites are recommended:

```
# OpenSSL cipher string for BSI TR-02102-2 compliance
# TLS 1.3 only
ssl_protocols = TLSv1.3;

# Preferred cipher suites (TLS 1.3)
# TLS_AES_256_GCM_SHA384
# TLS_CHACHA20_POLY1305_SHA256
# TLS_AES_128_GCM_SHA256 (acceptable but 256-bit preferred)

# Key exchange: ECDHE with Brainpool or NIST curves
# Signature: ECDSA with brainpoolP384r1 or RSA-PSS 4096

# Example OpenSSL configuration
[openssl_init]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
MinProtocol = TLSv1.3
Ciphersuites = TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
```

### Appendix C: Recommended FLOSS Tools Summary

| Function | Tool | License |
|---|---|---|
| Cloud Platform | OpenStack | Apache 2.0 |
| Hypervisor | KVM / libvirt | GPL |
| Storage | Ceph | LGPL |
| Container Runtime | Podman | Apache 2.0 |
| OS | AlmaLinux / Debian | GPL |
| Configuration Management | Ansible | GPL |
| Identity Management | FreeIPA | GPL |
| SSO / Federation | Keycloak | Apache 2.0 |
| Certificate Authority | EJBCA CE / DogTag | LGPL / GPL |
| Secrets Management | HashiCorp Vault (BSL) / OpenBao (FLOSS fork) | BSL / MPL |
| SIEM / HIDS | Wazuh | GPL |
| Log Search | OpenSearch | Apache 2.0 |
| Vulnerability Scanner | OpenVAS / Greenbone CE | GPL |
| Backup | BorgBackup | BSD |
| Git / CI-CD | GitLab CE | MIT (EE is proprietary) |
| ISMS Documentation | verinice | GPL (CE) |
| Monitoring | Prometheus + Grafana | Apache 2.0 |
| DNS | PowerDNS / BIND | GPL / MPL |
| NTP | chrony | GPL |
| IDS/IPS | Suricata | GPL |
| Reverse Proxy | nginx / HAProxy | BSD / GPL |

**Note on HashiCorp licensing:** HashiCorp Vault switched to the Business Source License (BSL) in 2023. For strict FLOSS requirements, consider **OpenBao** (the community fork under Linux Foundation, MPL-licensed) as an alternative.

### Appendix D: Estimated Bill of Materials (Sizing Example)

For a medium-scale deployment (~200 VMs, ~50TB usable storage):

| Component | Quantity | Notes |
|---|---|---|
| Control plane servers | 3 | HA cluster (2x CPU, 256GB RAM, 2x NVMe) |
| Compute nodes | 8-12 | 2x CPU, 512GB-1TB RAM, local NVMe |
| Ceph OSD nodes | 5-7 | 12x NVMe SSD per node, 2x 100GbE |
| Ceph MON/MGR | 3 | Can be co-located with control plane |
| Management servers | 3 | SIEM, monitoring, deployment |
| BSI-approved firewalls | 4 | 2x perimeter (HA), 2x internal (HA) |
| BSI-approved VPN | 2 | HA pair for cross-site/remote |
| HSM appliances | 2 | HA pair |
| Network switches | 8-12 | ToR + spine (25/100 GbE) |
| Management switches | 2-4 | Dedicated management network |

### Appendix E: Glossary

| Abbreviation | Full Term |
|---|---|
| BSI | Bundesamt fuer Sicherheit in der Informationstechnik |
| C5 | Cloud Computing Compliance Criteria Catalogue |
| CADF | Cloud Auditing Data Federation |
| CC | Common Criteria |
| EAL | Evaluation Assurance Level |
| FLOSS | Free/Libre Open Source Software |
| HSM | Hardware Security Module |
| IT-SiBe | IT-Sicherheitsbeauftragte/r |
| KRITIS | Kritische Infrastrukturen |
| OSSA | OpenStack Security Advisory |
| SEV-SNP | Secure Encrypted Virtualization - Secure Nested Paging |
| SiSyPHuS | Studie zu Systemaufbau, Protokollierung, Haertung und Sicherheitsfunktionen in Linux |
| SINA | Sichere Inter-Netzwerk Architektur |
| SUe1 | Sicherheitsueberpruefung Stufe 1 |
| SUeG | Sicherheitsueberpruefungsgesetz |
| TDX | Trust Domain Extensions (Intel) |
| TPM | Trusted Platform Module |
| VS-NfD | Verschlusssache - Nur fuer den Dienstgebrauch |
| VSA | Verschlusssachenanweisung |
| WPG | Wirtschaftspruefungsgesellschaft |

---

## Document Control

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-03-20 | Architecture Team | Initial architecture document |

---

*This document should be reviewed and updated at least annually or upon significant changes to the architecture, BSI regulations, or threat landscape. All BSI product approvals and certifications referenced herein should be verified against current BSI publications at the time of implementation.*
