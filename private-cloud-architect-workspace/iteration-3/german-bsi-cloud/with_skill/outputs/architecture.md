# Private Cloud Architecture for VS-NfD Processing
## German Federal Agency (Bundesbehoerde) -- BSI IT-Grundschutz and C5 Compliant

**Document Classification**: VS-NfD (Verschlusssache -- Nur fuer den Dienstgebrauch)
**Version**: 1.0
**Date**: 2026-03-20
**Author**: Cloud Architecture Team
**Status**: Architecture Proposal -- For Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Compliance Framework](#2-regulatory-and-compliance-framework)
3. [Classification and Data Sovereignty Requirements](#3-classification-and-data-sovereignty-requirements)
4. [Architecture Overview](#4-architecture-overview)
5. [Compute Layer](#5-compute-layer)
6. [Storage Architecture](#6-storage-architecture)
7. [Network Architecture](#7-network-architecture)
8. [Identity, Access Management, and Cryptography](#8-identity-access-management-and-cryptography)
9. [OpenStack Platform Design](#9-openstack-platform-design)
10. [FLOSS vs. BSI-Approved Components Analysis](#10-floss-vs-bsi-approved-components-analysis)
11. [Security Architecture](#11-security-architecture)
12. [Monitoring, Logging, and Audit](#12-monitoring-logging-and-audit)
13. [Automation and Infrastructure as Code](#13-automation-and-infrastructure-as-code)
14. [Disaster Recovery and Business Continuity](#14-disaster-recovery-and-business-continuity)
15. [BSI Accreditation Process and Roadmap](#15-bsi-accreditation-process-and-roadmap)
16. [Physical Security](#16-physical-security)
17. [Personnel and Operational Security](#17-personnel-and-operational-security)
18. [Supply Chain Security](#18-supply-chain-security)
19. [International Context: EU and NATO Alignment](#19-international-context-eu-and-nato-alignment)
20. [Risk Register](#20-risk-register)
21. [Implementation Roadmap](#21-implementation-roadmap)
22. [Architectural Decision Records](#22-architectural-decision-records)
23. [Appendices](#23-appendices)

---

## 1. Executive Summary

This document describes the architecture for a private cloud platform designed to process VS-NfD (Verschlusssache -- Nur fuer den Dienstgebrauch) classified data for a German federal agency (Bundesbehoerde). The platform is based on OpenStack, uses FLOSS components where permissible, and integrates BSI-approved products where regulatory requirements mandate them.

**Key design principles:**

- **Classification drives architecture**: VS-NfD requirements define every design decision, from cryptographic module selection to physical facility standards.
- **BSI IT-Grundschutz as the primary hardening baseline**: All systems are hardened according to BSI IT-Grundschutz Bausteine (building blocks), not DISA STIGs or CIS Benchmarks as the primary reference (these may be used as supplementary guidance only).
- **C5 attestation readiness**: The architecture is designed from day one to satisfy the BSI Cloud Computing Compliance Criteria Catalogue (C5:2020), enabling attestation by an independent auditor.
- **Data sovereignty**: All data processing and storage occurs within Germany, in BSI-approved or agency-controlled facilities. No data leaves German territory.
- **FLOSS-first, BSI-approved where required**: FLOSS components form the backbone of the platform. BSI-approved (VS-NfD-zugelassene) products are used specifically where the BSI mandates approved solutions -- primarily for cryptographic modules, VPN gateways, and certain security functions.
- **Automation and continuous compliance**: Infrastructure as Code, automated compliance scanning, and continuous monitoring are integral to the design, supporting both operational excellence and the continuous accreditation lifecycle.

**Target availability**: 99.99% (52.6 minutes unplanned downtime per year)
**Target data locations**: Two geographically separated data centers within Germany
**Target classification**: VS-NfD (the lowest German classification level, but still requiring specific BSI-mandated controls)

---

## 2. Regulatory and Compliance Framework

### 2.1 Primary German Frameworks

#### BSI IT-Grundschutz

IT-Grundschutz is the comprehensive security standard published by the Bundesamt fuer Sicherheit in der Informationstechnik (BSI). It is the **primary** security baseline for this architecture.

**Relevant components:**

- **IT-Grundschutz Kompendium**: The catalogue of Bausteine (building blocks) organized into process and system layers:
  - ISMS (Informationssicherheitsmanagementsystem)
  - ORP (Organisation und Personal)
  - CON (Konzeption und Vorgehensweisen)
  - OPS (Betrieb) -- operations including patch management, logging, backup
  - DER (Detektion und Reaktion) -- detection and incident response
  - INF (Infrastruktur) -- physical and facility security
  - NET (Netze und Kommunikation) -- network security
  - SYS (IT-Systeme) -- system hardening for servers, clients, virtualisation
  - APP (Anwendungen) -- application-level security
- **BSI-Standard 200-1**: Information Security Management Systems (ISMS)
- **BSI-Standard 200-2**: IT-Grundschutz Methodology (basis for structuring the security concept)
- **BSI-Standard 200-3**: Risk Analysis (supplementary risk analysis for high-protection needs beyond standard Grundschutz)
- **BSI-Standard 200-4**: Business Continuity Management (BCM)

**Relevant Bausteine for this architecture include (non-exhaustive):**

| Baustein | Topic | Relevance |
|----------|-------|-----------|
| SYS.1.1 | Allgemeiner Server | All servers |
| SYS.1.3 | Server unter Linux und Unix | All Linux hosts |
| SYS.1.5 | Virtualisierung | OpenStack hypervisors (KVM/libvirt) |
| SYS.1.6 | Containerisierung | Container workloads if deployed |
| SYS.1.8 | Speicherloesungen | Ceph storage cluster |
| NET.1.1 | Netzarchitektur und -design | Overall network design |
| NET.1.2 | Netzmanagement | Network operations |
| NET.3.1 | Router und Switches | Network infrastructure |
| NET.3.2 | Firewall | Perimeter and internal firewalls |
| NET.3.3 | VPN | Site-to-site and remote access VPN |
| OPS.1.1.3 | Patch- und Aenderungsmanagement | Patching and change management |
| OPS.1.1.5 | Protokollierung | Logging |
| OPS.1.2.2 | Archivierung | Long-term log and data retention |
| OPS.2.2 | Cloud-Nutzung | Cloud usage (applicable if extending) |
| DER.1 | Detektion von sicherheitsrelevanten Ereignissen | Security event detection |
| DER.2.1 | Behandlung von Sicherheitsvorfaellen | Incident handling |
| INF.1 | Allgemeines Gebaeude | Facility security |
| INF.2 | Rechenzentrum sowie Serverraum | Data center physical security |
| CON.1 | Kryptokonzept | Cryptographic concept |
| CON.3 | Datensicherungskonzept | Backup concept |

#### BSI C5 (Cloud Computing Compliance Criteria Catalogue)

C5:2020 is the BSI's attestation framework for cloud service providers. Even as an internal private cloud, achieving C5 attestation demonstrates that the cloud platform meets BSI-recognized security standards.

**C5 criteria domains:**

1. **Organisation der Informationssicherheit (OIS)** -- Security organization
2. **Sicherheitsrichtlinien und Arbeitsanweisungen (SP)** -- Security policies
3. **Personal (HR)** -- Personnel security
4. **Asset Management (AM)** -- Asset management
5. **Physische Sicherheit (PS)** -- Physical security
6. **Regelbetrieb (OPS)** -- Regular operations
7. **Identitaets- und Berechtigungsmanagement (IDM)** -- Identity and access management
8. **Kryptographie und Schluesselmanagement (CRY)** -- Cryptography and key management
9. **Kommunikationssicherheit (COS)** -- Communication security
10. **Portabilitaet und Interoperabilitaet (PI)** -- Portability and interoperability
11. **Beschaffung, Entwicklung und Aenderung von Informationssystemen (DEV)** -- Procurement, development, change management
12. **Steuerung und Ueberwachung von Dienstleistern und Lieferanten (SSO)** -- Supplier management
13. **Umgang mit Sicherheitsvorfaellen (SIM)** -- Security incident management
14. **Kontinuitaet des Geschaeftsbetriebs und Notfallmanagement (BCM)** -- BCM
15. **Compliance (COM)** -- Compliance
16. **Umgang mit Ermittlungsanfragen staatlicher Stellen (INQ)** -- Handling government investigation requests
17. **Produktsicherheit (PSS)** -- Product security

**C5 attestation types:**

- **Typ 1**: Design and implementation of controls at a point in time
- **Typ 2**: Operating effectiveness of controls over a period (typically 6-12 months) -- **this is the target**

#### VS-NfD Specific Requirements

VS-NfD is the lowest German classification level, roughly equivalent to NATO RESTRICTED or EU RESTRICTED. Key requirements:

- Data must be processed and stored on systems approved for VS-NfD
- Cryptographic protection of data in transit must use BSI-approved VS-NfD encryption (zugelassene Produkte)
- Personnel handling VS-NfD data must be instructed (belehrt) in accordance with the Verschlusssachenanweisung (VSA)
- Physical areas where VS-NfD data is processed must meet defined access control standards
- BSI-approved VPN products are mandatory for any VS-NfD data in transit over untrusted networks
- Audit trails must be maintained for all access to VS-NfD data

### 2.2 Governing Laws and Regulations

| Regulation | Relevance |
|------------|-----------|
| Verschlusssachenanweisung (VSA) | Handling procedures for classified material |
| BSI-Gesetz (BSIG) | BSI authority and obligations for federal agencies |
| IT-Sicherheitsgesetz 2.0 | Critical infrastructure IT security |
| Onlinezugangsgesetz (OZG) | Digitalization of government services |
| BDSG (Bundesdatenschutzgesetz) | German data protection law |
| GDPR / DSGVO | EU data protection regulation |
| NIS2 Directive (EU 2022/2555) | Network and information security for essential entities |
| eIDAS Regulation | Electronic identification and trust services |

### 2.3 International Context

As a German Bundesbehoerde, the following international frameworks apply contextually:

- **EU**: NIS2 Directive obligations (Germany transposed via NIS2UmsuCG), EUCS (EU Cybersecurity Certification Scheme for Cloud Services -- emerging, track adoption), GDPR/DSGVO.
- **NATO**: If the agency handles NATO RESTRICTED material alongside VS-NfD, NATO interoperability requirements per C-M(2002)49 apply. The architecture should not preclude future NATO RESTRICTED processing.
- **Common Criteria**: BSI is a major Common Criteria evaluation body. Where products with CC certification (particularly EAL4+ or higher) are available, they should be preferred for security-critical components.

---

## 3. Classification and Data Sovereignty Requirements

### 3.1 Data Classification

The platform processes data at the VS-NfD level. This means:

- **VS-NfD (Verschlusssache -- Nur fuer den Dienstgebrauch)**: Information whose unauthorized disclosure could be disadvantageous to the interests of the Federal Republic of Germany or one of its Laender. This is the lowest classification level but still requires specific protective measures.
- The platform does NOT process VS-VERTRAULICH, GEHEIM, or STRENG GEHEIM data. Those levels require significantly more stringent controls (physical isolation, TEMPEST, etc.) that are beyond the scope of this design.

### 3.2 Data Sovereignty

**Absolute requirements:**

1. All data at rest resides within Germany (physical location of servers and storage within German borders)
2. All data in transit between sites uses BSI-approved VS-NfD encryption and remains within German network infrastructure
3. No cloud service with data processing outside Germany is used
4. No foreign-government-accessible cloud platform (hyperscaler) is used for VS-NfD data
5. Hardware is sourced through trusted supply chains (see Section 18)
6. Support and administration are performed by personnel located in Germany with appropriate VS-NfD Belehrung

### 3.3 Data Flow Boundaries

```
+------------------------------------------------------------------+
|                      GERMAN TERRITORY                             |
|                                                                   |
|  +---------------------------+   BSI-approved   +--------------+  |
|  |   Data Center 1 (Primary) |   VPN (VS-NfD)  | Data Center 2|  |
|  |   (e.g., Frankfurt area)  |<==============>  | (e.g., Berlin|  |
|  |                           |   Dark fiber or  |    area)     |  |
|  |   OpenStack Region 1      |   MPLS backbone  | OpenStack    |  |
|  |   Full compute + storage  |                  | Region 2     |  |
|  +---------------------------+                  +--------------+  |
|                                                                   |
|  +---------------------------+                                    |
|  |   Management / Admin Zone |                                    |
|  |   (within DC1 or separate |                                    |
|  |    secured facility)      |                                    |
|  +---------------------------+                                    |
|                                                                   |
+------------------------------------------------------------------+
                    NO DATA CROSSES THIS BOUNDARY
```

---

## 4. Architecture Overview

### 4.1 High-Level Architecture

The architecture follows a two-region active-active design within Germany, with OpenStack as the IaaS platform, Ceph as the unified storage backend, and a layered security model aligned to BSI IT-Grundschutz and C5.

```
+=========================================================================+
|                        MANAGEMENT PLANE                                  |
|  Ansible (AWX) | OpenTofu | GitLab | NetBox | Monitoring | SIEM/SOC    |
+==========================================================================+
|                        SECURITY PLANE                                    |
|  BSI-approved VPN | Firewall | IDS/IPS | PKI | Key Management          |
+==========================================================================+
|                        OPENSTACK CONTROL PLANE                           |
|  Keystone | Nova | Neutron | Cinder | Glance | Heat | Octavia          |
|  Horizon | Barbican | Designate                                         |
+==========================================================================+
|                        COMPUTE PLANE                                     |
|  KVM/libvirt hypervisors | Nova compute nodes                           |
|  Optional: Kubernetes/KubeVirt for container workloads                  |
+==========================================================================+
|                        STORAGE PLANE                                     |
|  Ceph (RADOS) -- RBD for block, CephFS for file, RGW for object        |
|  Encrypted OSDs (LUKS/dm-crypt with BSI-approved algorithms)            |
+==========================================================================+
|                        NETWORK PLANE                                     |
|  Spine-leaf fabric | OVN/OVS for overlay | VXLAN                        |
|  Hardware firewalls at perimeter | Internal segmentation                 |
+==========================================================================+
|                        PHYSICAL INFRASTRUCTURE                           |
|  BSI-compliant data center | Redundant power/cooling | Access control   |
+==========================================================================+
```

### 4.2 Component Summary

| Layer | Primary Technology | FLOSS? | BSI Approval Required? |
|-------|-------------------|--------|----------------------|
| IaaS Platform | OpenStack (Kolla-Ansible deployment) | Yes | No (but must be hardened per IT-Grundschutz) |
| Hypervisor | KVM/libvirt | Yes | No |
| Block/Object/File Storage | Ceph | Yes | No |
| Network Overlay | OVN (Open Virtual Network) | Yes | No |
| Physical Network | Spine-leaf (vendor-specific switches) | No | Switches themselves no, but... |
| VPN (site-to-site, VS-NfD) | **BSI-approved product** (e.g., genuscreen, SINA) | **No** | **Yes -- mandatory** |
| Firewall (perimeter) | **BSI-approved or CC-certified** | Depends | **Recommended** |
| Encryption at rest | LUKS/dm-crypt (AES-256) | Yes | Algorithm approval yes, specific product no |
| Encryption in transit (TLS) | OpenSSL/GnuTLS | Yes | Must use BSI-approved algorithms |
| VPN (remote admin) | **BSI-approved product** | **No** | **Yes for VS-NfD access** |
| Identity/AuthN | Keycloak + FreeIPA | Yes | No (but must implement Grundschutz controls) |
| Key Management | Barbican + HSM | Barbican yes, HSM no | HSM should be CC-certified |
| Monitoring | Prometheus + Grafana + Loki | Yes | No |
| SIEM | Wazuh | Yes | No |
| Automation | Ansible (AWX) + OpenTofu | Yes | No |
| Backup | Bareos + Ceph replication | Yes | No |
| IDS/IPS | Suricata | Yes | No (but BSI-approved IDS is recommended) |
| Log Management | Loki + Grafana | Yes | No |
| Container Platform (optional) | Kubernetes (RKE2 or kubeadm) | Yes | No (hardened per Grundschutz SYS.1.6) |

---

## 5. Compute Layer

### 5.1 Hardware Selection

**Server platform**: Standardized 2U rack servers from a European or trusted vendor. Consider:
- Fujitsu PRIMERGY (German/Japanese, widely used in German government)
- Dell PowerEdge or HPE ProLiant (established supply chains, available with EU-specific firmware options)
- Open Compute Project (OCP) designs from European integrators

**Minimum specifications per compute node:**

| Component | Specification |
|-----------|--------------|
| CPU | 2x AMD EPYC 9004 series or Intel Xeon 5th Gen (128+ cores total) |
| RAM | 1 TB DDR5 ECC (expandable to 2 TB) |
| Boot | 2x 480 GB SSD (RAID 1, OS only) |
| Local cache | 2x 3.84 TB NVMe (optional, for nova local disk or Ceph journal/WAL) |
| NIC | 2x 25 GbE (compute traffic) + 2x 25 GbE (storage/Ceph) + 1 GbE (IPMI/iLO) |
| BMC | iLO/iDRAC/IPMI with dedicated management network |

**Node counts (initial deployment per region):**

| Role | Count | Purpose |
|------|-------|---------|
| OpenStack control plane | 3 | HA control plane (Keystone, Nova API, Neutron, etc.) |
| Compute nodes | 12-20 | VM hosting via Nova/KVM |
| Ceph nodes (OSD) | 5-9 | Distributed storage (min 5 for resilience) |
| Ceph MON/MGR | 3 (collocated on control plane or dedicated) | Ceph monitors and managers |
| Network nodes | 2-3 | Neutron agents, DVR, or dedicated gateway nodes |
| Infrastructure services | 3 | GitLab, AWX, monitoring, SIEM, NetBox |
| **Total per region** | **28-41** | |

### 5.2 Hypervisor

**Technology**: KVM with libvirt, managed by OpenStack Nova.

**Hardening (per BSI SYS.1.5 Virtualisierung):**

- Host OS: Hardened Linux (RHEL 9, Debian 12 Stable, or Ubuntu 22.04 LTS with BSI IT-Grundschutz hardening applied)
- SELinux in enforcing mode (RHEL) or AppArmor (Debian/Ubuntu) for mandatory access control
- sVirt for VM isolation through MAC labeling
- No unnecessary services on hypervisor hosts
- UEFI Secure Boot enabled
- TPM 2.0 for measured boot and attestation
- Firmware and microcode updates via controlled, tested pipeline
- NUMA-aware VM placement for performance
- CPU pinning and hugepages for latency-sensitive workloads
- Live migration restricted to encrypted channels (TLS with BSI-approved cipher suites)

### 5.3 Optional: Kubernetes for Container Workloads

If containerized workloads are required, deploy Kubernetes alongside OpenStack:

- **Distribution**: RKE2 (FIPS-capable, hardened by default) or kubeadm with manual hardening per BSI SYS.1.6
- **Integration**: Kubernetes nodes run as VMs on OpenStack (Nova), or on dedicated bare-metal nodes provisioned by Ironic
- **KubeVirt**: Consider for converging VM and container workloads on a single platform in later phases
- **Registry**: Self-hosted Harbor with Trivy scanning, air-gap capable
- **GitOps**: ArgoCD or Flux for declarative cluster management

---

## 6. Storage Architecture

### 6.1 Ceph Unified Storage

Ceph provides block (RBD), object (RGW), and file (CephFS) storage from a single cluster, eliminating the need for proprietary SAN/NAS.

**Ceph cluster design (per region):**

| Component | Count | Specification |
|-----------|-------|--------------|
| OSD nodes | 5-9 | Each with 8-12x 7.68 TB NVMe SSDs + 2x NVMe for WAL/DB |
| MON/MGR | 3 | Collocated on control nodes or dedicated (low resource) |
| MDS (if CephFS used) | 2 (active/standby) | Metadata servers for file storage |
| RGW (if object storage needed) | 2+ (load balanced) | S3-compatible object gateway |

**Storage configuration:**

- Replication factor: 3 (size=3, min_size=2)
- Failure domain: host (single-region), rack (if sufficient racks)
- Encryption at rest: OSD-level encryption using dm-crypt/LUKS with AES-256-XTS
- Key management: Keys stored in Barbican (OpenStack Key Manager), backed by HSM
- Erasure coding: For cold/archive data pools (e.g., 8+3 profile), replicated pools for hot data
- Network: Dedicated storage network (25 GbE minimum, 100 GbE recommended for large clusters) separated from public/tenant traffic

**Integration with OpenStack:**

- **Cinder**: RBD backend for block volumes
- **Glance**: RBD backend for VM images
- **Nova**: RBD for ephemeral disks (live migration enabled)
- **Manila** (optional): CephFS backend for shared file systems

### 6.2 Encryption at Rest

All data at rest is encrypted using dm-crypt/LUKS:

- **Algorithm**: AES-256-XTS (BSI TR-02102-1 approved)
- **Key derivation**: PBKDF2 or Argon2
- **Key storage**: Keys managed by OpenStack Barbican, master keys in HSM (CC EAL4+ certified)
- **Scope**: Every Ceph OSD, every boot disk, every local disk on every node

This is a critical VS-NfD requirement. Even though VS-NfD does not mandate encryption at rest in all scenarios (unlike higher classification levels), it is a defense-in-depth measure and is required by multiple IT-Grundschutz Bausteine.

---

## 7. Network Architecture

### 7.1 Physical Network Design

**Topology**: Spine-leaf architecture for each data center.

```
                    +--------+    +--------+
                    | Spine1 |    | Spine2 |
                    +---++---+    +---++---+
                        ||            ||
          +-------------++------------++-------------+
          |              |            |              |
     +----+---+    +----+---+   +----+---+    +----+---+
     | Leaf 1 |    | Leaf 2 |   | Leaf 3 |    | Leaf 4 |
     +----+---+    +----+---+   +----+---+    +----+---+
          |              |            |              |
    Compute Rack 1  Compute Rack 2  Storage Rack  Mgmt/Ctrl
```

**Switch selection**: Select switches that support VXLAN-EVPN and are available through trusted supply chains. Options include:

- **Cumulus Linux / NVIDIA Spectrum** (FLOSS NOS): Open networking, Ansible-automatable, no vendor lock-in
- **Arista EOS**: Strong automation support, widely deployed in European government DCs
- **Cisco Nexus 9000** (NX-OS mode): If the agency has existing Cisco expertise and SmartNet contracts

**Network segmentation (VLANs / VRFs):**

| Network | VLAN/VRF | Purpose | Security Zone |
|---------|----------|---------|---------------|
| Management | VLAN 10 | BMC/IPMI, out-of-band management | Restricted, admin only |
| OpenStack Internal API | VLAN 20 | Control plane internal communication | Restricted |
| OpenStack Public API | VLAN 30 | Tenant-facing API endpoints (Keystone, Nova, etc.) | Controlled |
| Tenant Overlay | VXLAN (dynamic) | Tenant VM traffic (via OVN) | Tenant-isolated |
| Storage (Ceph cluster) | VLAN 40 | Ceph OSD replication, MON heartbeat | Restricted, storage only |
| Storage (Ceph public) | VLAN 41 | Ceph client access (Nova, Cinder) | Controlled |
| External/Provider | VLAN 50 | Floating IPs, external connectivity | DMZ |
| Site-to-site WAN | Dedicated | Inter-DC replication and failover | Encrypted (VS-NfD VPN) |

### 7.2 Software-Defined Networking

**Technology**: OVN (Open Virtual Network) as the OpenStack Neutron ML2 plugin backend.

- OVN provides distributed virtual routing, ACLs, DHCP, and load balancing
- Runs on Open vSwitch (OVS) on each compute and network node
- Eliminates the need for centralized Neutron L3 agents (distributed by design)
- VXLAN encapsulation for tenant overlay networks
- Security groups implemented as OVN ACLs (stateful firewall per port)

**Alternative considered**: Cisco ACI (via the ACI Neutron plugin). This was evaluated but rejected for this deployment because:
- Adds significant proprietary dependency and licensing cost
- OVN provides equivalent functionality for this scale
- ACI is more appropriate for very large multi-tenant environments or where Cisco ACI is already deployed
- The agency does not have existing ACI expertise

### 7.3 Perimeter Security

**Firewall:**

The perimeter firewall is a critical security boundary. For VS-NfD, a BSI-approved or CC-certified firewall is strongly recommended.

**Options:**

1. **genugate** (genua GmbH, Germany) -- BSI-approved, CC-certified (EAL4+), designed for German government use. **Recommended for perimeter.**
2. **SINA (Sichere Inter-Netzwerk Architektur)** by secunet -- BSI-approved, widely used in German government for VS-NfD environments. Can serve as firewall + VPN.
3. **OPNsense/pfSense** (FLOSS) -- Suitable for internal segmentation firewalls, but **not BSI-approved for perimeter protection of VS-NfD networks**. Use only for non-VS-NfD zones or as supplementary internal firewalls.

**Recommendation**: genugate or SINA at the perimeter; OPNsense for internal segmentation between non-classified management zones.

### 7.4 VPN (Site-to-Site)

**This is the single most critical BSI-approval requirement.**

For transporting VS-NfD data between data centers over any network that is not exclusively physically controlled by the agency, a **BSI-approved VS-NfD VPN solution** is **mandatory**.

**BSI-approved VS-NfD VPN products include:**

| Product | Vendor | Type | Notes |
|---------|--------|------|-------|
| genuscreen | genua GmbH | VPN gateway + firewall | German company, CC EAL4+, BSI VS-NfD approved |
| SINA Box | secunet Security Networks AG | VPN gateway + secure workstation | German company, widely used in Bundesverwaltung |
| R&S Trusted VPN | Rohde & Schwarz Cybersecurity | VPN gateway | German company, BSI VS-NfD approved |
| LANCOM VPN | LANCOM Systems (Rohde & Schwarz subsidiary) | VPN router | German company, for smaller deployments |

**Recommendation**: Deploy genuscreen or SINA Box appliances at each data center for site-to-site VPN. These appliances handle:
- IPsec VPN with BSI-approved cryptographic algorithms
- VS-NfD compliant key management
- Integrated firewall functionality

**You cannot replace these with WireGuard, OpenVPN, strongSwan, or any other FLOSS VPN for VS-NfD traffic.** This is a non-negotiable regulatory requirement. The BSI maintains the list of approved products (VS-NfD-zugelassene Produkte) and only products on that list may be used.

### 7.5 DNS and DHCP

- **Internal DNS**: PowerDNS or CoreDNS (FLOSS), integrated with OpenStack Designate
- **DHCP**: Kea DHCP (FLOSS), managed by Neutron/OVN for tenant networks
- **External DNS**: Split-horizon DNS, external zone served only for necessary public-facing services (if any)

---

## 8. Identity, Access Management, and Cryptography

### 8.1 Identity Architecture

```
+-------------------+        +------------------+
|    Keycloak       |<------>|    FreeIPA        |
|  (OIDC/SAML IdP) |  LDAP  | (Kerberos, LDAP, |
|  Web SSO, MFA    |        |  DNS, PKI/CA)     |
+--------+----------+        +--------+---------+
         |                            |
         v                            v
+--------+----------+        +--------+---------+
| OpenStack Keystone|        | Linux hosts      |
| (federation)      |        | (SSSD, Kerberos) |
+-------------------+        +------------------+
```

**Components:**

- **FreeIPA**: Central identity, Kerberos authentication, LDAP directory, integrated CA, DNS, HBAC (host-based access control), sudo rules. Deployed as 3-node replica set for HA.
- **Keycloak**: OIDC/SAML identity provider for web-based SSO. Provides MFA (TOTP, FIDO2/WebAuthn). Federates to FreeIPA via LDAP.
- **OpenStack Keystone**: Federated to Keycloak via OIDC. Projects and roles mapped from Keycloak groups.
- **Linux hosts**: Joined to FreeIPA domain via SSSD. Kerberos for authentication, HBAC for authorization.

**Multi-factor authentication**: Mandatory for all administrative access. FIDO2 hardware tokens (e.g., YubiKey) preferred; TOTP as fallback. This satisfies BSI IT-Grundschutz ORP.4 (Identitaets- und Berechtigungsmanagement) and C5 IDM requirements.

### 8.2 Cryptographic Concept (Kryptokonzept -- per CON.1)

BSI TR-02102-1 (Kryptographische Verfahren: Empfehlungen und Schluessellaengen) is the **primary reference** for all cryptographic algorithm selection.

**Approved algorithms (per BSI TR-02102-1, current edition):**

| Use Case | Algorithm | Key Length | Notes |
|----------|-----------|------------|-------|
| Symmetric encryption | AES | 128, 192, or 256 bit | 256-bit preferred |
| Block cipher mode (disk) | XTS | -- | For full-disk encryption |
| Block cipher mode (network) | GCM, CCM | -- | For TLS and IPsec |
| Hash functions | SHA-256, SHA-384, SHA-512, SHA3 | -- | SHA-1 prohibited |
| Key exchange | ECDHE (P-256, P-384, brainpoolP256r1, brainpoolP384r1) | -- | BSI prefers Brainpool curves |
| Digital signatures | ECDSA, EdDSA, RSA | RSA >= 3000 bit (from 2024) | ECDSA with Brainpool preferred |
| TLS | TLS 1.2 (with restricted cipher suites) or TLS 1.3 | -- | TLS 1.0/1.1 prohibited |
| VPN | IPsec IKEv2 | -- | Only with BSI-approved products for VS-NfD |

**Critical note on Brainpool curves**: BSI explicitly recommends Brainpool curves (brainpoolP256r1, brainpoolP384r1, brainpoolP512r1) alongside NIST curves. For VS-NfD, using Brainpool curves demonstrates alignment with BSI preferences. Ensure OpenSSL/GnuTLS is compiled with Brainpool support.

**Key management:**

- OpenStack Barbican as the cloud key management service
- Hardware Security Module (HSM) for master key protection -- use a **CC EAL4+ certified HSM** (e.g., Utimaco SecurityServer, Thales Luna Network HSM, or Bundesdruckerei D-Trust HSM)
- Key rotation policies: symmetric keys rotated annually minimum, asymmetric keys per certificate validity (max 2 years for TLS)
- All key material for VS-NfD VPN managed within the BSI-approved VPN appliances' own key management (not externally managed)

### 8.3 TLS Everywhere

All internal communication is encrypted with TLS:

- OpenStack internal API endpoints: TLS 1.3 (or TLS 1.2 with BSI-approved cipher suites)
- Ceph cluster communication: messenger v2 with TLS
- Database replication (MariaDB Galera): TLS
- RabbitMQ cluster: TLS
- Monitoring (Prometheus scraping): TLS with mutual authentication
- All certificates issued by FreeIPA integrated CA or a dedicated internal CA

---

## 9. OpenStack Platform Design

### 9.1 Deployment Method

**Kolla-Ansible** is the recommended deployment tool for this environment:

- Deploys all OpenStack services as containers (Docker/Podman)
- Provides reproducible, version-controlled deployments
- Supports rolling upgrades between OpenStack releases
- Well-tested, actively maintained upstream
- Alternative considered: Kayobe (wraps Kolla-Ansible, adds bare-metal provisioning) -- use if bare-metal lifecycle management via OpenStack Bifrost is desired

### 9.2 OpenStack Services

| Service | Project | Purpose | HA Model |
|---------|---------|---------|----------|
| Identity | Keystone | AuthN/AuthZ, federation to Keycloak | 3-node active-active behind HAProxy |
| Compute | Nova | VM lifecycle, scheduling | 3 API nodes, N compute agents |
| Networking | Neutron + OVN | Virtual networking, security groups | 3 API nodes, distributed OVN |
| Block Storage | Cinder | Volume management, Ceph RBD backend | 3 API nodes, Ceph backend |
| Image | Glance | VM image management, Ceph RBD backend | 3 nodes, images in Ceph |
| Dashboard | Horizon | Web UI for tenants and admins | 3 nodes behind HAProxy |
| Orchestration | Heat | Stack templates, autoscaling | 3 nodes |
| Load Balancer | Octavia | LBaaS for tenants | 3 API nodes, amphora VMs |
| DNS | Designate | DNSaaS | 3 nodes, PowerDNS backend |
| Key Manager | Barbican | Secret and key management | 3 nodes, HSM backend |
| Bare Metal (optional) | Ironic | Bare-metal provisioning | 3 nodes |
| Object Storage (optional) | Swift or Ceph RGW | S3-compatible object storage | Ceph RGW recommended |

### 9.3 High Availability

**Control plane HA:**

- 3 control plane nodes per region
- HAProxy + keepalived (or kube-vip) for API VIP
- MariaDB Galera cluster (3 nodes) for database
- RabbitMQ cluster (3 nodes, quorum queues) for messaging
- Memcached (3 nodes) for token caching

**Compute HA:**

- Nova automatically reschedules VMs if a compute node fails (requires shared storage via Ceph)
- Masakari (OpenStack instance HA) for automatic VM evacuation on host failure
- Live migration for planned maintenance (zero-downtime patching of hypervisors)

**Storage HA:**

- Ceph replication factor 3 ensures data survives loss of any 2 OSDs (or 1 OSD with min_size=2)
- MON quorum requires 2 of 3 monitors
- No single point of failure in the storage path

### 9.4 Multi-Region Design

Two OpenStack regions, one per data center:

- **Shared Keystone**: Single Keystone identity service (replicated across both DCs via Galera replication over VS-NfD VPN), or federated Keystone with Keycloak as the common IdP
- **Independent control planes**: Each region has its own Nova, Neutron, Cinder, Glance, etc.
- **Ceph replication**: Asynchronous RBD mirroring between Ceph clusters in each region for DR
- **Shared Horizon**: Can access both regions from a single dashboard

**Recommendation**: Federated Keystone with a shared Keycloak IdP is simpler to operate and avoids cross-DC database replication for the identity layer.

---

## 10. FLOSS vs. BSI-Approved Components Analysis

This is the critical analysis for the agency's question about where FLOSS can be used and where BSI-approved products are required.

### 10.1 Where BSI-Approved Products Are MANDATORY

These are non-negotiable requirements for VS-NfD:

| Function | Requirement | Reason |
|----------|-------------|--------|
| **VPN for VS-NfD data in transit** | BSI VS-NfD-zugelassenes VPN product (genuscreen, SINA Box, R&S Trusted VPN) | VSA and BSI regulations mandate approved crypto products for VS-NfD data over non-agency-controlled networks |
| **Cryptographic algorithms** | Must conform to BSI TR-02102-1 | While the algorithms themselves are implemented in FLOSS (OpenSSL), the specific algorithm selection must follow BSI guidance |
| **VS-NfD remote access** | BSI-approved remote access solution (SINA Workstation, genuconnect) | If personnel access VS-NfD data remotely, the endpoint and VPN must be BSI-approved |

### 10.2 Where BSI-Approved Products Are STRONGLY RECOMMENDED

Not strictly mandatory, but expected by auditors and significantly simplifying the accreditation process:

| Function | Recommendation | Reason |
|----------|---------------|--------|
| **Perimeter firewall** | BSI-approved or CC EAL4+ certified (genugate, SINA) | IT-Grundschutz NET.3.2 expects certified products at security boundaries |
| **Hardware Security Module** | CC EAL4+ certified HSM | CON.1 Kryptokonzept expects certified key storage for sensitive cryptographic keys |
| **IDS/IPS at perimeter** | BSI-recommended or CC-certified IDS | DER.1 expects reliable detection capabilities; certified products simplify the argument |
| **Smartcard/token for MFA** | CC-certified hardware token (e.g., BSI-certified smartcard) | Strong authentication for VS-NfD access benefits from certified tokens |

### 10.3 Where FLOSS Is Fully Appropriate

These components can use FLOSS without BSI-approval concerns, provided they are hardened per IT-Grundschutz:

| Function | FLOSS Component | Grundschutz Hardening Required |
|----------|----------------|-------------------------------|
| IaaS Platform | OpenStack | Yes (SYS.1.5, APP-specific) |
| Hypervisor | KVM/libvirt | Yes (SYS.1.5) |
| Host OS | Linux (RHEL, Debian, Ubuntu) | Yes (SYS.1.3) |
| Storage | Ceph | Yes (SYS.1.8) |
| Network overlay | OVN/OVS | Yes (NET.1.1) |
| Identity/Directory | FreeIPA | Yes (ORP.4) |
| SSO/Federation | Keycloak | Yes (ORP.4) |
| Monitoring | Prometheus + Grafana | Yes (OPS.1.1.5) |
| Log aggregation | Loki or ELK/Wazuh | Yes (OPS.1.1.5, DER.1) |
| SIEM | Wazuh | Yes (DER.1) |
| IaC/Automation | Ansible (AWX), OpenTofu | Yes (OPS.1.1.3) |
| Backup | Bareos | Yes (CON.3) |
| Container runtime | containerd, CRI-O | Yes (SYS.1.6) |
| Container orchestration | Kubernetes (RKE2/kubeadm) | Yes (SYS.1.6) |
| Container registry | Harbor | Yes (SYS.1.6) |
| DNS | PowerDNS, CoreDNS | Yes (NET.1.1) |
| DHCP | Kea | Yes (NET.1.1) |
| DCIM/IPAM | NetBox | Yes (OPS) |
| Database | MariaDB, PostgreSQL | Yes (APP.4.3, APP.4.4 equivalent) |
| Load balancer | HAProxy | Yes (NET) |
| Internal firewall/segmentation | OPNsense, iptables/nftables | Yes (NET.3.2 for internal use) |
| Certificate management | FreeIPA CA, cert-manager | Yes (CON.1) |

### 10.4 Summary Decision Matrix

```
+-------------------------------------------------------------------+
|                    BSI APPROVAL DECISION TREE                      |
|                                                                    |
|  Does the component handle VS-NfD data                            |
|  crossing a network security boundary?                             |
|       |                                                            |
|       +-- YES --> Is it cryptographic protection                   |
|       |           of data in transit?                              |
|       |              |                                             |
|       |              +-- YES --> BSI-APPROVED PRODUCT MANDATORY    |
|       |              |           (VPN, crypto appliance)           |
|       |              |                                             |
|       |              +-- NO --> FLOSS acceptable, but              |
|       |                         harden per IT-Grundschutz          |
|       |                                                            |
|       +-- NO --> FLOSS acceptable, harden per IT-Grundschutz      |
|                  CC-certified products preferred for               |
|                  security-critical functions (HSM, firewall)       |
+-------------------------------------------------------------------+
```

---

## 11. Security Architecture

### 11.1 Defense in Depth

The security architecture implements multiple layers:

**Layer 1 -- Physical**: BSI-compliant data center (see Section 16)
**Layer 2 -- Perimeter**: BSI-approved firewall + VPN, IDS/IPS (Suricata or BSI-approved)
**Layer 3 -- Network segmentation**: VLAN isolation, OVN security groups, microsegmentation
**Layer 4 -- Host hardening**: IT-Grundschutz SYS.1.3 hardened Linux, SELinux/AppArmor, Secure Boot
**Layer 5 -- Application security**: TLS everywhere, input validation, secure coding
**Layer 6 -- Data protection**: Encryption at rest (LUKS), encryption in transit (TLS), key management (Barbican + HSM)
**Layer 7 -- Identity and access**: FreeIPA + Keycloak, MFA, RBAC, need-to-know enforcement
**Layer 8 -- Monitoring and detection**: Wazuh SIEM, Suricata IDS, Prometheus alerting, audit logging
**Layer 9 -- Incident response**: DER.2.1 compliant incident response procedures

### 11.2 Hardening Baseline

**Primary baseline**: BSI IT-Grundschutz Kompendium (current edition)

For each system type, apply the corresponding Baustein:
- Linux servers: SYS.1.3 plus SYS.1.1
- Virtualization hosts: SYS.1.5
- Network equipment: NET.3.1
- Firewalls: NET.3.2

**Supplementary references** (not primary):
- CIS Benchmarks (for additional technical detail where IT-Grundschutz is less prescriptive)
- DISA STIGs (informational only, not authoritative in German context)
- BSI SiSyPHuS studies (BSI's own analysis of Linux/Windows security, very detailed)

**Automated compliance scanning:**
- OpenSCAP with BSI IT-Grundschutz profiles (where available)
- Custom Ansible playbooks implementing IT-Grundschutz Umsetzungshinweise
- Regular automated scans with results fed to SIEM/dashboards
- Deviation alerts trigger investigation per change management process

### 11.3 Network Security

- **Default deny**: All firewall rules and security groups start from default-deny
- **Microsegmentation**: OVN ACLs enforce per-port security groups; tenant networks are fully isolated
- **IDS/IPS**: Suricata deployed on network gateway nodes, monitoring north-south traffic; consider BSI-approved IDS for perimeter
- **East-west monitoring**: Network flow logs from OVN, analyzed by Wazuh
- **DDoS protection**: Rate limiting at perimeter firewall; relevant primarily if any services are internet-facing
- **No direct internet access**: VS-NfD processing zones have no direct internet connectivity. Any required connectivity (e.g., package updates) goes through a controlled proxy/transfer zone

### 11.4 Vulnerability Management

- Regular vulnerability scanning (OpenVAS/Greenbone, FLOSS)
- Patch management per OPS.1.1.3 (defined patch windows, emergency patching procedures)
- CVE monitoring for all deployed components (OpenStack, Ceph, Linux kernel, etc.)
- Coordinated disclosure process for any vulnerabilities discovered internally

---

## 12. Monitoring, Logging, and Audit

### 12.1 Monitoring Stack

```
+------------------+     +------------------+     +------------------+
|  Prometheus      |     |  Loki            |     |  Wazuh Manager   |
|  (metrics)       |     |  (logs)          |     |  (SIEM/IDS)      |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         v                        v                        v
+--------+------------------------+------------------------+---------+
|                         Grafana                                     |
|              (unified dashboards, alerting)                         |
+--------------------------------------------------------------------+
```

**Components:**

| Component | Purpose | Retention |
|-----------|---------|-----------|
| Prometheus | Infrastructure and application metrics | 90 days hot, Thanos for long-term |
| Alertmanager | Alert routing (email, ticketing system integration) | -- |
| Loki | Log aggregation from all hosts and services | 1 year hot, archive to Ceph object storage |
| Wazuh | SIEM, host-based IDS, compliance monitoring | 3+ years (BSI DER.1 / C5 retention requirements) |
| Grafana | Dashboards, unified query interface | -- |
| Thanos | Long-term metric storage, multi-region metric federation | 2+ years |
| NetBox | Asset inventory, DCIM, IPAM | Permanent |

### 12.2 Audit Logging Requirements

Per IT-Grundschutz OPS.1.1.5 and C5 requirements:

- **All authentication events** (success and failure) logged and forwarded to SIEM
- **All authorization changes** (role assignments, project membership) logged
- **All API calls** to OpenStack services logged (Keystone audit middleware)
- **All SSH sessions** logged (session recording recommended for administrative access)
- **All changes to security configurations** (firewall rules, security groups, crypto config)
- **All access to VS-NfD data** must be attributable to an individual user (no shared accounts)
- **Log integrity**: Logs are signed or hashed to detect tampering; forwarded to a separate, restricted log aggregation system
- **Log retention**: Minimum 1 year for operational logs, 3+ years for security-relevant audit logs

### 12.3 Alerting

Critical alerts requiring immediate response:

- Authentication failures exceeding threshold
- Unauthorized access attempts to VS-NfD resources
- Security group or firewall rule modifications
- Ceph cluster health degradation
- OpenStack control plane service failures
- Host intrusion detection events (Wazuh)
- Network anomaly detection (Suricata)
- Certificate expiration within 30 days
- Compliance scan failures

---

## 13. Automation and Infrastructure as Code

### 13.1 Automation Strategy

**Principle**: Every aspect of the infrastructure is defined in code, version-controlled, and deployed via automated pipelines. Manual changes are prohibited in production.

```
+------------------+     +------------------+     +------------------+
|   GitLab         |---->|   AWX            |---->|   Target Systems |
|   (Git repos,    |     |   (Ansible       |     |   (OpenStack,    |
|    CI/CD)        |     |    execution)    |     |    hosts, Ceph)  |
+------------------+     +------------------+     +------------------+
         |
         +--------------->+------------------+
                          |   OpenTofu       |---->  OpenStack resources
                          |   (provisioning) |      (VMs, networks,
                          +------------------+       volumes, etc.)
```

### 13.2 Tooling

| Tool | Purpose | Scope |
|------|---------|-------|
| **Ansible (AWX)** | Configuration management, OS hardening, OpenStack deployment (via Kolla-Ansible), day-2 operations | All hosts, all services |
| **OpenTofu** | Declarative provisioning of OpenStack resources (tenants, networks, VMs, security groups) | OpenStack tenant resources |
| **GitLab** (self-hosted) | Version control, CI/CD pipelines, merge request reviews, audit trail | All IaC code, documentation |
| **Packer** | Golden image creation (hardened Linux base images) | VM images for Glance |
| **Cloud-init** | Instance bootstrapping (first-boot configuration, FreeIPA join) | All tenant VMs |

### 13.3 GitOps Workflow

1. All changes are proposed via GitLab merge requests
2. Automated CI pipeline runs: linting, syntax checks, dry-run/plan
3. Peer review by a second administrator (four-eyes principle, required by IT-Grundschutz)
4. Merge triggers automated deployment via AWX job template or OpenTofu apply
5. Post-deployment validation (automated tests, compliance scans)
6. All actions logged and attributable to individual users

### 13.4 Golden Image Pipeline

```
Upstream ISO --> Packer build --> IT-Grundschutz hardening (Ansible) -->
Vulnerability scan (OpenVAS) --> Compliance scan (OpenSCAP) -->
Upload to Glance --> Available for tenant use
```

Images are rebuilt monthly (or on critical CVE) and versioned. Old images are deprecated on a defined schedule.

---

## 14. Disaster Recovery and Business Continuity

### 14.1 DR Strategy

| Objective | Target |
|-----------|--------|
| RPO (Recovery Point Objective) | 1 hour (asynchronous replication lag) |
| RTO (Recovery Time Objective) | 4 hours (failover to secondary region) |
| Backup retention | 30 days incremental, 1 year monthly full |

### 14.2 Replication

- **Ceph RBD mirroring**: Asynchronous replication of critical volumes between regions (RPO approximately 5-15 minutes depending on change rate)
- **Database replication**: MariaDB Galera for synchronous replication within a region; asynchronous replication (or backup restore) for cross-region
- **Configuration replication**: All IaC in GitLab, which is replicated to both sites (GitLab Geo)

### 14.3 Backup

- **Tool**: Bareos (FLOSS, fork of Bacula, actively maintained, suitable for enterprise)
- **Strategy**:
  - Full backup weekly
  - Incremental backup daily
  - Ceph snapshots hourly for critical volumes
  - Backup data encrypted with AES-256 and stored on separate Ceph pool or dedicated backup storage
  - Backup copies at both data centers
- **Testing**: Monthly backup restore tests, documented per BSI-Standard 200-4 (BCM)

### 14.4 DR Testing

- Quarterly DR drills simulating primary site failure
- Annual full failover test
- All DR tests documented with results reviewed by the Informationssicherheitsbeauftragter (ISB -- Information Security Officer)
- DR runbooks maintained in GitLab, version-controlled, reviewed quarterly

---

## 15. BSI Accreditation Process and Roadmap

This section describes how to navigate the BSI accreditation process for achieving IT-Grundschutz compliance and C5 attestation.

### 15.1 Overview of the Accreditation Lifecycle

The general accreditation pattern applies (Categorize -> Select Controls -> Implement -> Assess -> Authorize -> Monitor), adapted to the German BSI framework:

```
+------------------------------------------------------------------+
|  Phase 1: CATEGORIZE (Strukturanalyse + Schutzbedarfsfeststellung)|
|  - Inventory all information assets (Informationsverbund)         |
|  - Determine protection needs (Schutzbedarf: normal/hoch/sehr hoch)|
|  - Document in a Strukturanalyse                                  |
+------------------------------------------------------------------+
           |
           v
+------------------------------------------------------------------+
|  Phase 2: SELECT (Modellierung nach IT-Grundschutz)               |
|  - Map Bausteine to each component in the Informationsverbund    |
|  - Identify applicable Anforderungen (requirements) per Baustein |
|  - For "hoch" and "sehr hoch" Schutzbedarf: perform supplementary|
|    risk analysis per BSI-Standard 200-3                          |
+------------------------------------------------------------------+
           |
           v
+------------------------------------------------------------------+
|  Phase 3: IMPLEMENT (Umsetzung)                                   |
|  - Implement all Basis- and Standard-Anforderungen                |
|  - For "hoch" Schutzbedarf: implement additional requirements     |
|  - Document implementation in the IT-Grundschutz-Check            |
|  - Build the Sicherheitskonzept (security concept document)       |
+------------------------------------------------------------------+
           |
           v
+------------------------------------------------------------------+
|  Phase 4: ASSESS (Audit)                                          |
|  - For IT-Grundschutz: BSI-certified auditor (BSI-zertifizierter |
|    IS-Revisor or ISO 27001 auditor auf Basis von IT-Grundschutz) |
|  - For C5: Wirtschaftspruefer (public auditor) performs C5 audit  |
|    (ISAE 3402 / ISAE 3000 based attestation)                    |
|  - Auditors review documentation, interview staff, inspect systems|
+------------------------------------------------------------------+
           |
           v
+------------------------------------------------------------------+
|  Phase 5: CERTIFY/ATTEST                                          |
|  - IT-Grundschutz: BSI issues ISO 27001 certificate based on     |
|    IT-Grundschutz (valid 3 years, annual surveillance audits)    |
|  - C5: Auditor issues C5 attestation report (Typ 1 or Typ 2)    |
+------------------------------------------------------------------+
           |
           v
+------------------------------------------------------------------+
|  Phase 6: CONTINUOUS MONITORING (Aufrechterhaltung und Verbesserung)|
|  - Continuous compliance scanning                                 |
|  - Annual surveillance audits                                     |
|  - Re-certification every 3 years                                 |
|  - Significant changes trigger re-assessment                      |
+------------------------------------------------------------------+
```

### 15.2 Phase 1: Strukturanalyse and Schutzbedarfsfeststellung

**Strukturanalyse** (structural analysis):

Document every component of the Informationsverbund (the scope of the IT system being accredited):

- All physical assets: servers, switches, firewalls, HSMs, storage, racks, cabling
- All software components: OpenStack services, Ceph, Linux OS, Keycloak, FreeIPA, etc.
- All network connections: VLANs, VPN tunnels, internet links, inter-DC links
- All data flows: where VS-NfD data is processed, stored, and transmitted
- All personnel roles: administrators, operators, tenants, auditors
- All physical locations: data centers, management offices

**Schutzbedarfsfeststellung** (protection needs assessment):

For each information asset, determine the Schutzbedarf across three protection goals:

| Protection Goal | Normal | Hoch (High) | Sehr hoch (Very high) |
|----------------|--------|-------------|----------------------|
| Vertraulichkeit (Confidentiality) | -- | VS-NfD data: **hoch** | -- |
| Integritaet (Integrity) | -- | Cloud control plane: **hoch** | -- |
| Verfuegbarkeit (Availability) | -- | Core services: **hoch** | -- |

For VS-NfD, the confidentiality Schutzbedarf is typically "hoch" (high), which triggers additional requirements beyond the Basis-Anforderungen in each Baustein.

### 15.3 Phase 2: Modellierung (Modeling)

Map each IT-Grundschutz Baustein to the appropriate components:

| Component | Applicable Bausteine |
|-----------|---------------------|
| All Linux servers | SYS.1.1, SYS.1.3 |
| KVM hypervisors | SYS.1.5 |
| Ceph storage | SYS.1.8 |
| OpenStack control plane | APP (application-specific), SYS.1.1, SYS.1.3 |
| Network infrastructure | NET.1.1, NET.1.2, NET.3.1 |
| Perimeter firewall | NET.3.2 |
| VPN | NET.3.3 |
| Data center facility | INF.2 |
| Overall ISMS | ISMS.1 |
| Organization | ORP.1, ORP.2, ORP.3, ORP.4 |
| Cryptographic concept | CON.1 |
| Backup | CON.3 |
| Incident response | DER.2.1 |
| Logging/detection | DER.1, OPS.1.1.5 |

For each Baustein, the Basis-Anforderungen are mandatory. For components with "hoch" Schutzbedarf, the Standard-Anforderungen and Anforderungen bei erhoehtem Schutzbedarf also apply.

### 15.4 Phase 3: Umsetzung (Implementation)

This is where the architecture described in this document is built. Key activities:

1. **Build the platform** per this architecture document
2. **Apply IT-Grundschutz hardening** per each applicable Baustein's Umsetzungshinweise (implementation notes)
3. **Document everything** in the Sicherheitskonzept:
   - Strukturanalyse results
   - Schutzbedarfsfeststellung results
   - Modellierung (Baustein mapping)
   - IT-Grundschutz-Check results (status of each Anforderung: ja/teilweise/nein/entbehrlich)
   - Supplementary risk analysis (per BSI-Standard 200-3) for any gaps or areas with "hoch/sehr hoch" Schutzbedarf where standard Grundschutz requirements may not suffice
   - Residual risk acceptance (signed by agency leadership)
4. **Perform the IT-Grundschutz-Check**: For every Anforderung in every applicable Baustein, assess the implementation status:
   - **ja** (yes): fully implemented
   - **teilweise** (partially): partially implemented, with remediation plan
   - **nein** (no): not implemented, with justification or remediation plan
   - **entbehrlich** (dispensable): not applicable with documented justification

### 15.5 Phase 4: Audit

**For IT-Grundschutz certification:**

- Engage a BSI-licensed audit team (BSI-zertifizierte Auditoren fuer ISO 27001 auf Basis von IT-Grundschutz)
- The audit verifies:
  - ISMS is in place and operational (BSI-Standard 200-1)
  - Grundschutz methodology was correctly applied (BSI-Standard 200-2)
  - Risk analysis was performed where required (BSI-Standard 200-3)
  - All applicable Anforderungen are implemented or justified
  - Operational procedures are being followed
- The BSI grants the certificate "ISO 27001 auf Basis von IT-Grundschutz"

**For C5 attestation:**

- Engage a Wirtschaftspruefungsgesellschaft (public audit firm) qualified for C5 attestation (e.g., KPMG, Deloitte, PwC, EY, or specialized firms)
- The audit is performed according to ISAE 3402 (for Typ 2)
- The auditor examines:
  - All 17 C5 criteria domains
  - Environmental parameters (Umgebungsparameter) -- these are aspects the cloud customer (even if internal) must be aware of
  - System descriptions and control descriptions
  - For Typ 2: operating effectiveness over a 6-12 month observation period
- The auditor issues a C5 attestation report (Pruefungsbericht)

**Recommended sequence:**

1. Start with IT-Grundschutz certification (establishes the ISMS and security baseline)
2. Then pursue C5 Typ 1 attestation (validates control design)
3. After 6-12 months of operation, pursue C5 Typ 2 attestation (validates operating effectiveness)

### 15.6 Phase 5: Certification and Attestation

- IT-Grundschutz certificate is valid for 3 years with annual surveillance audits
- C5 attestation (Typ 2) covers a defined observation period and is typically renewed annually
- Both are presented to internal stakeholders, other government agencies (who may consume the cloud services), and oversight bodies as evidence of security posture

### 15.7 Phase 6: Continuous Monitoring

Design for this from day one:

- **Automated compliance scanning**: OpenSCAP with IT-Grundschutz profiles, Ansible compliance playbooks, Wazuh policy monitoring
- **Continuous monitoring dashboards**: Grafana dashboards showing compliance status, open findings, remediation progress
- **Change management**: All changes through GitOps pipeline with four-eyes principle, automatically triggering compliance re-scans
- **Annual surveillance audits**: Prepare documentation and evidence continuously, not as a last-minute effort
- **Significant change management**: If the architecture changes significantly (e.g., adding a new service, changing a security-critical component), trigger a re-assessment of the affected Bausteine

### 15.8 Key BSI Contacts and Resources

| Resource | URL / Contact |
|----------|--------------|
| BSI IT-Grundschutz Kompendium | https://www.bsi.bund.de/grundschutz |
| BSI C5 Criteria Catalogue | https://www.bsi.bund.de/C5 |
| BSI Technical Guidelines (TR) | https://www.bsi.bund.de/technische-richtlinien |
| BSI VS-NfD approved products | Consult BSI directly (list not publicly available in full) |
| BSI-certified auditors directory | https://www.bsi.bund.de (search for certified auditors) |
| BSI advisory service for federal agencies | Direct contact via agency's BSI liaison |

**Important**: As a Bundesbehoerde, the agency has a direct relationship with the BSI. Engage the BSI early and often -- they provide advisory services to federal agencies and can guide the accreditation process. The BSI is not just an auditor; they are a partner for federal agencies.

---

## 16. Physical Security

### 16.1 Data Center Requirements (per INF.2)

Both data centers must meet:

- **Access control**: Multi-factor physical access (badge + PIN/biometric), visitor escort policy, access logging
- **Perimeter security**: Fencing, CCTV, intrusion detection
- **Environmental controls**: Redundant cooling (N+1 minimum), fire detection and suppression (gas-based, not water, in server rooms), water leak detection, temperature and humidity monitoring
- **Power**: Redundant power feeds (2N), UPS (battery, minimum 15 minutes for graceful shutdown or generator start), diesel generator with minimum 72-hour fuel supply
- **Zutrittsschutz** (access protection): Defined security zones with escalating access requirements:
  - Zone 1: Building perimeter
  - Zone 2: Technical areas (corridors)
  - Zone 3: Server rooms (restricted to authorized personnel)
  - Zone 4: High-security areas (e.g., HSM room, VS-NfD processing zone)

### 16.2 Geographic Separation

The two data centers must be geographically separated to survive regional disasters:

- Minimum 100 km apart (BSI recommendation for georedundancy)
- Connected via dedicated dark fiber or MPLS with BSI-approved VPN
- Each data center capable of running the full workload independently (active-active or active-warm-standby)

### 16.3 TEMPEST Considerations

For VS-NfD, full TEMPEST protection (NATO SDIP-27 Zone A/B) is **not required**. However:

- Basic emanation security hygiene should be observed (no classified processing near windows facing public areas)
- If the threat assessment identifies TEMPEST risks (e.g., hostile intelligence services in proximity), additional measures may be warranted
- For higher classification levels (VS-VERTRAULICH and above), TEMPEST zones become mandatory -- this architecture does not address those levels

---

## 17. Personnel and Operational Security

### 17.1 Personnel Requirements

| Role | VS-NfD Requirement | Additional |
|------|-------------------|------------|
| All staff handling VS-NfD | Formal VS-NfD Belehrung (instruction on handling classified material) per VSA | Documented acknowledgment |
| System administrators | VS-NfD Belehrung + background suitability check | Privileged access management |
| Security officer (ISB) | VS-NfD Belehrung + ISMS expertise | BSI IT-Grundschutz Praktiker/Experte certification recommended |
| External contractors | VS-NfD Belehrung + contractual obligations | NDA, supervision requirements |

**Note**: VS-NfD does not require a formal Sicherheitsueberprufung (security clearance vetting) like VS-VERTRAULICH and above. The Belehrung (instruction) and acknowledgment are sufficient.

### 17.2 Operational Security (ORP Bausteine)

- **Four-eyes principle**: All changes to production systems require review by a second administrator (enforced via GitOps merge request approval)
- **Least privilege**: Role-based access via FreeIPA HBAC and OpenStack Keystone roles; no standing admin privileges (just-in-time access via Keycloak)
- **Separation of duties**: Distinct roles for OpenStack admin, network admin, security admin, storage admin
- **No shared accounts**: Every action attributable to an individual (ORP.4)
- **Secure administration**: Administrative access only from dedicated management workstations on the management network (SYS.1.1 "Allgemeiner Server" Anforderung A18: use of dedicated admin workstations)

---

## 18. Supply Chain Security

### 18.1 Hardware Procurement

- Prefer European manufacturers (Fujitsu, Bull/Atos, European OCP integrators)
- If using US-manufactured hardware (Dell, HPE, Cisco), source through trusted German/EU distributors
- Verify hardware integrity upon delivery: tamper-evident packaging, serial number verification
- Firmware verification: check firmware hashes against vendor-published values
- BIOS/UEFI settings: disable unnecessary features, enable Secure Boot, configure TPM 2.0
- Document chain of custody for all hardware

### 18.2 Software Supply Chain

- All FLOSS packages sourced from official upstream repositories or trusted mirrors
- GPG signature verification for all packages
- SBOM (Software Bill of Materials) generated for all deployed components
- Container images built from known base images, scanned with Trivy/Grype
- No unapproved third-party repositories
- Dependency tracking and CVE monitoring for all software components
- Air-gap-friendly: maintain an internal mirror of all required repositories (Pulp, Aptly) to avoid runtime dependency on external networks from VS-NfD zones

---

## 19. International Context: EU and NATO Alignment

### 19.1 EU Framework Compliance

| Framework | Status | Impact |
|-----------|--------|--------|
| GDPR/DSGVO | Mandatory | Data protection controls built into the architecture |
| NIS2 Directive | Mandatory (transposed as NIS2UmsuCG) | Incident reporting, risk management, supply chain security |
| EUCS (EU Cybersecurity Certification Scheme) | Emerging | Monitor developments; C5 is likely to be recognized under EUCS |
| eIDAS | May apply | If the agency issues or consumes electronic identities/signatures |

### 19.2 NATO Considerations

If the agency processes NATO RESTRICTED alongside national VS-NfD:

- NATO RESTRICTED can typically be processed on VS-NfD-accredited systems (with bilateral security agreement)
- Verify with the agency's Geheimschutzbeauftragter (security officer) and the BSI
- NATO-specific markings and handling procedures must be observed
- NATO-approved crypto may be required for NATO-classified data exchange with allied nations

### 19.3 Cross-Border Data Sharing

- VS-NfD data can be shared with equivalent classification levels in partner nations (e.g., NATO RESTRICTED, EU RESTRICTED) subject to bilateral/multilateral security agreements
- The architecture supports this via the BSI-approved VPN infrastructure, which can establish connections to partner agencies
- All cross-border data transfers must be approved and documented

---

## 20. Risk Register

| Risk ID | Risk | Likelihood | Impact | Mitigation |
|---------|------|-----------|--------|------------|
| R01 | OpenStack upgrade introduces breaking changes | Medium | High | Staging environment, rolling upgrades via Kolla-Ansible, LTS releases |
| R02 | Ceph data corruption or cluster failure | Low | Very High | Replication factor 3, cross-region async replication, regular scrubbing |
| R03 | BSI-approved VPN product has limited throughput | Medium | Medium | Capacity planning, multiple parallel VPN tunnels, benchmark during PoC |
| R04 | FLOSS component lacks required security feature for Grundschutz | Low | Medium | Identify gaps early during modeling phase, compensating controls, or substitute component |
| R05 | Skilled personnel shortage (OpenStack + BSI expertise) | High | High | Training program, engage specialized consulting firms (e.g., B1 Systems, OSISM, Cleura), document everything |
| R06 | Single site failure | Low | High | Active-active multi-region design, DR procedures tested quarterly |
| R07 | Supply chain compromise (hardware or software) | Low | Very High | Trusted procurement, firmware verification, SBOM, signature verification |
| R08 | C5 audit finds significant gaps | Medium | High | Pre-audit internal review, engage BSI advisory early, continuous compliance scanning |
| R09 | Regulatory changes (new BSI requirements) | Medium | Medium | Monitor BSI publications, maintain relationship with BSI liaison, flexible architecture |
| R10 | Key personnel departure | Medium | High | Documentation-first culture, cross-training, IaC reduces bus factor |

---

## 21. Implementation Roadmap

### Phase 0: Planning and Procurement (Months 1-3)

- Finalize Strukturanalyse and Schutzbedarfsfeststellung
- Procure hardware (long lead times for government procurement -- Vergaberecht)
- Procure BSI-approved products (VPN, firewall, HSM)
- Engage BSI advisory service
- Recruit/train team
- Set up GitLab, begin IaC development in lab environment

### Phase 1: Foundation (Months 4-6)

- Data center preparation (if not using existing facility)
- Physical network deployment (spine-leaf fabric)
- Management infrastructure: GitLab, AWX, NetBox, monitoring
- FreeIPA and Keycloak deployment
- BSI-approved VPN and firewall deployment
- Begin IT-Grundschutz hardening documentation

### Phase 2: OpenStack Deployment (Months 7-10)

- Deploy OpenStack Region 1 via Kolla-Ansible
- Deploy Ceph cluster Region 1
- Integrate Barbican with HSM
- Implement TLS everywhere
- Deploy monitoring and SIEM
- Internal testing and hardening
- Begin Sicherheitskonzept documentation

### Phase 3: Second Region and DR (Months 11-14)

- Deploy OpenStack Region 2
- Deploy Ceph cluster Region 2
- Configure Ceph RBD mirroring
- Configure federated Keystone/Keycloak
- DR testing
- Backup implementation and testing

### Phase 4: Hardening and Pre-Audit (Months 15-18)

- Complete IT-Grundschutz-Check (all Anforderungen assessed)
- Remediate all identified gaps
- Complete Sicherheitskonzept documentation
- Internal pre-audit (self-assessment)
- Penetration testing by qualified provider
- Engage BSI-certified auditor for IT-Grundschutz certification

### Phase 5: Certification and Attestation (Months 19-24)

- IT-Grundschutz certification audit
- Begin C5 observation period (6-12 months for Typ 2)
- Onboard pilot workloads
- C5 Typ 1 attestation (point-in-time)
- Continue operations, building evidence for Typ 2

### Phase 6: C5 Typ 2 and Full Operations (Months 25-30)

- C5 Typ 2 attestation audit (after 6-12 month observation)
- Full workload onboarding
- Continuous improvement cycle begins

**Total timeline: approximately 24-30 months from start to full C5 Typ 2 attestation.**

This is realistic for a German government agency considering procurement timelines (Vergaberecht), BSI engagement, and the depth of IT-Grundschutz documentation requirements.

---

## 22. Architectural Decision Records

### ADR-001: OpenStack as IaaS Platform

**Status**: Accepted
**Context**: Need an IaaS platform for VM-centric government workloads. Alternatives: VMware vSphere, Nutanix, Proxmox VE.
**Decision**: OpenStack, deployed via Kolla-Ansible.
**Rationale**: FLOSS (no licensing costs, no vendor lock-in), API-driven self-service, mature ecosystem, widely deployed in European government and research institutions (CERN, European Weather Centre, various NRENs). Avoids VMware Broadcom licensing uncertainty. Larger community and ecosystem than Proxmox for IaaS scale.
**Consequences**: Requires OpenStack expertise (training/hiring). More operationally complex than Nutanix/VMware. Offset by strong IaC and automation practices.

### ADR-002: Ceph for Unified Storage

**Status**: Accepted
**Context**: Need block, object, and optionally file storage. Alternatives: proprietary SAN (NetApp, Pure), vSAN, GlusterFS.
**Decision**: Ceph, integrated with OpenStack via native drivers.
**Rationale**: FLOSS, proven at scale, native OpenStack integration, unified platform for block/object/file, no per-TB licensing. Strong community and commercial support options (Red Hat, SUSE, Clyso).
**Consequences**: Requires Ceph expertise. Operational complexity higher than appliance-based storage. Mitigated by automation and monitoring.

### ADR-003: BSI-Approved VPN for VS-NfD Transit

**Status**: Accepted
**Context**: VS-NfD data must be protected in transit between data centers. FLOSS VPN options exist (WireGuard, strongSwan).
**Decision**: BSI-approved VPN product (genuscreen or SINA Box).
**Rationale**: Regulatory requirement -- non-negotiable. BSI VS-NfD regulations mandate the use of BSI-approved cryptographic products for classified data in transit. No FLOSS product is on the BSI approval list.
**Consequences**: Proprietary cost and vendor dependency for this specific component. Limited to BSI-approved product capabilities and throughput. Mitigated by capacity planning and PoC testing.

### ADR-004: OVN over Cisco ACI for Network Overlay

**Status**: Accepted
**Context**: Need SDN for tenant network isolation. Alternatives: Cisco ACI (with Neutron plugin), pure OVS, Tungsten Fabric.
**Decision**: OVN (Open Virtual Network) as Neutron ML2 backend.
**Rationale**: FLOSS, native integration with OpenStack Neutron, distributed architecture (no central controller bottleneck), sufficient functionality for this deployment scale, no additional licensing. ACI rejected due to cost and complexity for the agency's scale.
**Consequences**: No ACI microsegmentation features. Mitigated by OVN ACLs and Suricata IDS. ACI could be reconsidered for future scale-out.

### ADR-005: Keycloak + FreeIPA over Active Directory

**Status**: Accepted
**Context**: Need identity management, SSO, MFA. Alternatives: Microsoft Active Directory + Azure AD, proprietary IAM.
**Decision**: FreeIPA for directory/Kerberos, Keycloak for OIDC/SAML SSO and MFA.
**Rationale**: FLOSS, no per-user licensing, full control over identity infrastructure, Kerberos for Linux-native authentication, Keycloak provides modern OIDC federation for OpenStack and web services.
**Consequences**: Requires FreeIPA/Keycloak expertise. If integration with existing AD is required (e.g., desktop environment), FreeIPA can establish AD trust relationships.

### ADR-006: Kolla-Ansible for OpenStack Deployment

**Status**: Accepted
**Context**: Need a reliable, repeatable OpenStack deployment method. Alternatives: TripleO (deprecated), Kayobe, Sunbeam, manual.
**Decision**: Kolla-Ansible.
**Rationale**: Mature, actively maintained, containerized services (easy upgrades), large community, supports all required OpenStack services, Ansible-based (aligns with agency automation strategy).
**Consequences**: Tied to Ansible for OpenStack lifecycle. Acceptable given Ansible is already the primary automation tool.

### ADR-007: Wazuh for SIEM

**Status**: Accepted
**Context**: Need SIEM for security event detection and compliance. Alternatives: Splunk, IBM QRadar, ELK stack alone.
**Decision**: Wazuh (FLOSS).
**Rationale**: FLOSS, includes HIDS, log analysis, compliance monitoring (PCI-DSS, GDPR profiles -- adaptable for IT-Grundschutz), active community, integrates with ELK/OpenSearch for log storage. Avoids Splunk per-GB licensing.
**Consequences**: Requires customization for IT-Grundschutz-specific compliance rules. FLOSS means self-supported (or engage Wazuh Inc. for commercial support).

---

## 23. Appendices

### Appendix A: BSI Technical Guidelines Referenced

| Guideline | Title | Relevance |
|-----------|-------|-----------|
| BSI TR-02102-1 | Kryptographische Verfahren: Empfehlungen und Schluessellaengen | Cryptographic algorithm and key length selection |
| BSI TR-02102-2 | Verwendung von Transport Layer Security (TLS) | TLS configuration requirements |
| BSI TR-02102-3 | Verwendung von Internet Protocol Security (IPsec) und Internet Key Exchange (IKEv2) | IPsec/VPN configuration |
| BSI TR-02102-4 | Verwendung von Secure Shell (SSH) | SSH hardening |
| BSI TR-03116 | Kryptographische Vorgaben fuer Projekte der Bundesregierung | Crypto requirements for federal projects |
| BSI TR-03138 | Ersetzendes Scannen (RESISCAN) | If document scanning is required |

### Appendix B: Relevant BSI IT-Grundschutz Bausteine (Detailed)

The full list of applicable Bausteine must be determined during the Modellierung phase. The following is an indicative list for the major components of this architecture:

**Process layer:**
- ISMS.1 -- Sicherheitsmanagement
- ORP.1 -- Organisation
- ORP.2 -- Personal
- ORP.3 -- Sensibilisierung und Schulung zur Informationssicherheit
- ORP.4 -- Identitaets- und Berechtigungsmanagement
- CON.1 -- Kryptokonzept
- CON.3 -- Datensicherungskonzept
- CON.6 -- Loeschen und Vernichten
- CON.8 -- Software-Entwicklung (if custom development occurs)
- OPS.1.1.2 -- Ordnungsgemaesse IT-Administration
- OPS.1.1.3 -- Patch- und Aenderungsmanagement
- OPS.1.1.5 -- Protokollierung
- OPS.1.1.6 -- Software-Tests und -Freigaben
- OPS.1.2.2 -- Archivierung
- DER.1 -- Detektion von sicherheitsrelevanten Ereignissen
- DER.2.1 -- Behandlung von Sicherheitsvorfaellen
- DER.4 -- Notfallmanagement

**System layer:**
- SYS.1.1 -- Allgemeiner Server
- SYS.1.3 -- Server unter Linux und Unix
- SYS.1.5 -- Virtualisierung
- SYS.1.6 -- Containerisierung
- SYS.1.8 -- Speicherloesungen (NAS/SAN) -- applicable to Ceph
- APP.3.4 -- Samba (if applicable)
- APP.4.3 -- Relationale Datenbanken (MariaDB)

**Network layer:**
- NET.1.1 -- Netzarchitektur und -design
- NET.1.2 -- Netzmanagement
- NET.3.1 -- Router und Switches
- NET.3.2 -- Firewall
- NET.3.3 -- VPN

**Infrastructure layer:**
- INF.1 -- Allgemeines Gebaeude
- INF.2 -- Rechenzentrum sowie Serverraum

### Appendix C: Estimated Bill of Materials (Per Region)

| Item | Quantity | Est. Unit Cost | Est. Total |
|------|----------|---------------|------------|
| Compute servers (2U, 2x EPYC, 1 TB RAM) | 20 | 25,000 EUR | 500,000 EUR |
| Ceph storage nodes (2U, 12x NVMe) | 9 | 40,000 EUR | 360,000 EUR |
| Control plane servers | 3 | 15,000 EUR | 45,000 EUR |
| Infrastructure servers | 3 | 15,000 EUR | 45,000 EUR |
| Network switches (spine, 100G) | 2 | 30,000 EUR | 60,000 EUR |
| Network switches (leaf, 25G) | 4 | 15,000 EUR | 60,000 EUR |
| BSI-approved VPN appliance (genuscreen/SINA) | 2 | 25,000 EUR | 50,000 EUR |
| BSI-approved perimeter firewall (genugate) | 2 (HA pair) | 40,000 EUR | 80,000 EUR |
| HSM (CC EAL4+) | 2 (HA pair) | 30,000 EUR | 60,000 EUR |
| Management switches (1G) | 2 | 3,000 EUR | 6,000 EUR |
| Cabling, optics, PDUs, racks | lot | -- | 80,000 EUR |
| **Subtotal per region** | | | **~1,346,000 EUR** |
| **Two regions** | | | **~2,692,000 EUR** |
| **Software licensing** | | | **0 EUR (FLOSS)** |
| **BSI-approved product maintenance (annual)** | | | **~50,000 EUR** |
| **Consulting/integration (one-time)** | | | **~300,000-500,000 EUR** |
| **Training** | | | **~50,000-100,000 EUR** |
| **C5 audit (annual)** | | | **~80,000-150,000 EUR** |
| **IT-Grundschutz certification (initial + annual)** | | | **~50,000-80,000 EUR** |

**Note**: These are rough order-of-magnitude estimates. Actual costs depend on procurement framework agreements (Rahmenvertraege), government pricing, and specific vendor negotiations. Government procurement (Vergaberecht) may require public tender above EU thresholds.

### Appendix D: Team Skill Requirements

| Skill Area | Required Competency | Training Source |
|------------|-------------------|----------------|
| OpenStack administration | Deployment, operations, upgrades | Upstream training, B1 Systems, OSISM |
| Ceph administration | Deployment, monitoring, troubleshooting | Red Hat/SUSE training, upstream docs |
| Linux system administration | RHEL/Debian hardening, SELinux, performance | RHCE/LPIC, BSI SiSyPHuS |
| BSI IT-Grundschutz | Methodology, modeling, documentation | BSI IT-Grundschutz Praktiker/Experte certification |
| Network engineering | Spine-leaf, VXLAN, OVN, VPN | Vendor training, CNCF |
| Ansible/IaC | Playbook development, AWX operation | Red Hat training, upstream docs |
| Security operations | SIEM, IDS, incident response | SANS, BSI training |
| Kubernetes (if needed) | Cluster operations, security hardening | CKA/CKS certification |

### Appendix E: Glossary of German Terms

| German Term | English Translation | Context |
|-------------|-------------------|---------|
| Bundesbehoerde | Federal agency | Government organization |
| Bundesamt fuer Sicherheit in der Informationstechnik (BSI) | Federal Office for Information Security | The national cybersecurity authority |
| Verschlusssache (VS) | Classified information | Overarching term |
| VS-NfD (Nur fuer den Dienstgebrauch) | Classified -- For official use only | Lowest classification level |
| VS-VERTRAULICH | Classified -- Confidential | Second classification level |
| GEHEIM | Secret | Third classification level |
| STRENG GEHEIM | Top Secret | Highest classification level |
| Verschlusssachenanweisung (VSA) | Classified information handling directive | Procedures for handling classified material |
| IT-Grundschutz Kompendium | IT baseline protection compendium | BSI's security standard catalogue |
| Baustein | Building block | A module in the IT-Grundschutz catalogue |
| Anforderung | Requirement | A specific security requirement within a Baustein |
| Schutzbedarf | Protection need | The assessed level of protection required |
| Schutzbedarfsfeststellung | Protection needs assessment | Formal assessment process |
| Strukturanalyse | Structural analysis | Inventory and documentation of IT assets |
| Modellierung | Modeling | Mapping of Bausteine to assets |
| IT-Grundschutz-Check | IT baseline protection check | Assessment of requirement implementation status |
| Sicherheitskonzept | Security concept | The overarching security documentation |
| Informationssicherheitsbeauftragter (ISB) | Information security officer | Responsible for ISMS |
| Geheimschutzbeauftragter | Classified information security officer | Responsible for VS handling |
| Belehrung | Instruction/briefing | Formal instruction on VS handling procedures |
| Sicherheitsueberprufung | Security clearance vetting | Background check (not required for VS-NfD) |
| Wirtschaftspruefer | Public auditor | Performs C5 attestation |
| Vergaberecht | Procurement law | Government procurement regulations |
| Rahmenvertrag | Framework agreement | Pre-negotiated procurement contract |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-20 | Cloud Architecture Team | Initial architecture proposal |

**Review required by:**
- Informationssicherheitsbeauftragter (ISB)
- Geheimschutzbeauftragter
- IT-Leitung (IT leadership)
- BSI advisory (recommended)

**Next steps:**
1. Internal review and approval of this architecture document
2. Engage BSI advisory service for early feedback
3. Begin procurement process (Vergabeverfahren)
4. Start Phase 0 activities in parallel with procurement

---

*This document was prepared following BSI IT-Grundschutz methodology (BSI-Standard 200-2) and addresses the requirements for processing VS-NfD classified data in a private cloud environment. All national German standards (BSI IT-Grundschutz, BSI TR series, C5) serve as the primary reference. US standards (NIST, DISA STIGs) are referenced only for supplementary technical detail where explicitly noted, and are not authoritative in this context.*
