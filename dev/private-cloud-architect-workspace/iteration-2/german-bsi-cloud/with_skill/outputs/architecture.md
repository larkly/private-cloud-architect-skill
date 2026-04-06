# Private Cloud Architecture for VS-NfD Processing
# BSI IT-Grundschutz and C5 Compliant OpenStack Environment

**Classification**: VS-NfD (Verschlusssache - Nur fuer den Dienstgebrauch)
**Client**: Bundesbehoerde (German Federal Agency)
**Date**: 2026-03-20
**Version**: 1.0
**Status**: Architecture Proposal

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Compliance Framework](#2-regulatory-and-compliance-framework)
3. [Classification-Driven Architecture Principles](#3-classification-driven-architecture-principles)
4. [FLOSS vs. BSI-Approved Products Decision Matrix](#4-floss-vs-bsi-approved-products-decision-matrix)
5. [Reference Architecture](#5-reference-architecture)
6. [Compute Architecture](#6-compute-architecture)
7. [Network Architecture](#7-network-architecture)
8. [Storage Architecture](#8-storage-architecture)
9. [Identity, Authentication, and Access Control](#9-identity-authentication-and-access-control)
10. [Encryption and Cryptography](#10-encryption-and-cryptography)
11. [Logging, Monitoring, and Audit](#11-logging-monitoring-and-audit)
12. [Automation and Infrastructure as Code](#12-automation-and-infrastructure-as-code)
13. [Disaster Recovery and Business Continuity](#13-disaster-recovery-and-business-continuity)
14. [Physical Security and Data Center Requirements](#14-physical-security-and-data-center-requirements)
15. [BSI IT-Grundschutz Implementation Roadmap](#15-bsi-it-grundschutz-implementation-roadmap)
16. [C5 Attestation Process](#16-c5-attestation-process)
17. [Supply Chain and Procurement](#17-supply-chain-and-procurement)
18. [Operational Model and Personnel](#18-operational-model-and-personnel)
19. [Landing Zone Design](#19-landing-zone-design)
20. [Risk Assessment](#20-risk-assessment)
21. [Implementation Phases](#21-implementation-phases)
22. [Architectural Decision Records](#22-architectural-decision-records)
23. [Appendix: BSI IT-Grundschutz Module Mapping](#23-appendix-bsi-it-grundschutz-module-mapping)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform designed to process VS-NfD (Verschlusssache - Nur fuer den Dienstgebrauch) classified data for a German federal agency. The platform is built on OpenStack and uses FLOSS components wherever security and compliance permit, with BSI-approved products mandated at specific layers -- most critically for cryptographic functions and network boundary protection.

**Key architectural decisions:**

- **OpenStack** as the IaaS platform, deployed via Kolla-Ansible, providing full API-driven self-service for compute, networking, and storage
- **Ceph** as the unified software-defined storage backend (block, object, file)
- **FLOSS throughout most of the stack**, with targeted use of BSI-approved/VS-NfD-certified products for: VPN/encryption gateways, hardware security modules, and network perimeter firewalls
- **Data residency strictly within Germany**, in BSI-compliant data center facilities
- **BSI IT-Grundschutz** as the primary security framework (targeting "Standard" protection level with elevation to "Hoch" for specific information assets)
- **C5 attestation** as the cloud-specific compliance target, pursued in parallel with IT-Grundschutz certification
- **99.99% availability** through multi-zone design within a single German data center campus (with DR to a second German location)

**What must NOT be FLOSS (BSI-approved products required):**

| Layer | Requirement | Reason |
|-------|------------|--------|
| VPN gateways | VS-NfD-approved VPN appliance | BSI mandates approved crypto for VS-NfD network transit |
| Hardware Security Modules | BSI-approved HSM (e.g., Utimaco, Rohde & Schwarz) | Key material for VS-NfD must be protected in approved HSMs |
| Perimeter firewall | BSI-certified firewall appliance | Boundary protection for VS-NfD network zone |
| Smartcard/token authentication | BSI-approved PKI tokens | Personnel authentication for VS-NfD access |

**What CAN be FLOSS:**

OpenStack itself, Ceph storage, Linux operating system (hardened), internal OVN/OVS networking, Prometheus/Grafana monitoring, Ansible/OpenTofu automation, Keycloak (as internal IdP, not replacing BSI-approved auth tokens), HAProxy load balancers, and the vast majority of the platform stack.

---

## 2. Regulatory and Compliance Framework

### 2.1 German Classification System

The German classification system relevant to this deployment:

| Level | German Term | Description |
|-------|------------|-------------|
| Open | OFFEN | No classification |
| Restricted | VS-NfD | Nur fuer den Dienstgebrauch -- for official use only |
| Confidential | VS-VERTRAULICH | Confidential |
| Secret | GEHEIM | Secret |
| Top Secret | STRENG GEHEIM | Strictly secret |

**VS-NfD** is the lowest classification level. It does not require full air-gapped operation, but it does impose specific controls on cryptography, access, personnel, and data handling. This is the level targeted by this architecture.

### 2.2 BSI IT-Grundschutz

BSI IT-Grundschutz is Germany's comprehensive information security framework, published and maintained by the Bundesamt fuer Sicherheit in der Informationstechnik (BSI). It is among the most prescriptive frameworks globally.

**Key components:**

- **BSI-Standard 200-1**: Information Security Management Systems (ISMS) -- defines the management framework
- **BSI-Standard 200-2**: IT-Grundschutz Methodology -- defines how to model the IT landscape and select controls
- **BSI-Standard 200-3**: Risk Analysis -- defines the risk assessment methodology when standard controls are insufficient
- **BSI-Standard 200-4**: Business Continuity Management
- **IT-Grundschutz Compendium**: The catalog of building blocks (Bausteine) organized into process layers and system layers. Each Baustein contains threats, requirements (Anforderungen), and implementation guidance.

**Protection levels:**

- **Basis** (Basic): Minimum security measures, suitable for low-risk systems
- **Standard**: Adequate security for normal protection needs -- **this is our baseline target**
- **Hoch** (High): Enhanced security measures for systems with elevated protection needs -- **required for components directly handling VS-NfD data**

**Certification options:**

- **IT-Grundschutz-Zertifikat** (Certificate): Full BSI audit and certification, valid for 3 years with annual surveillance audits
- **ISO 27001 auf der Basis von IT-Grundschutz**: An ISO 27001 certificate that uses IT-Grundschutz as the control set (more rigorous than standard ISO 27001)

For a Bundesbehoerde processing VS-NfD, the **IT-Grundschutz-Zertifikat** at Standard/Hoch level is the appropriate target.

### 2.3 BSI C5 (Cloud Computing Compliance Criteria Catalogue)

C5 is the BSI's attestation framework specifically designed for cloud services. It was updated to C5:2020 and defines requirements across 17 domains:

| # | Domain | ID |
|---|--------|----|
| 1 | Organisation of Information Security | OIS |
| 2 | Security Policies and Work Instructions | SP |
| 3 | Personnel | HR |
| 4 | Asset Management | AM |
| 5 | Physical Security | PS |
| 6 | Operations Security | OPS |
| 7 | Identity and Access Management | IDM |
| 8 | Cryptography and Key Management | CKM |
| 9 | Communication Security | COS |
| 10 | Portability and Interoperability | PI |
| 11 | Procurement and Configuration of Cloud Services | DEV |
| 12 | Operating Procedures of Cloud Services | SSO |
| 13 | Identity and Rights Management of Cloud Services | SIM |
| 14 | Monitoring and Logging of Cloud Services | LOG |
| 15 | Compliance | COM |
| 16 | Data Protection | DPR |
| 17 | Business Continuity | BCM |

**C5 attestation types:**

- **Type 1**: Attestation of design -- controls are suitably designed at a point in time
- **Type 2**: Attestation of design AND operating effectiveness over a period (minimum 6 months) -- **this is the target**

C5 attestation is performed by an independent auditor (Wirtschaftspruefer) following ISAE 3402/IDW PS 951 standards.

### 2.4 Additional Regulatory Requirements

- **VSA (Verschlusssachenanweisung)**: The general administrative instruction for handling classified information in federal agencies. This dictates procedural controls.
- **BDSG and GDPR**: Data protection requirements. While VS-NfD data may not always be personal data, the platform will likely also process personal data, so GDPR compliance is mandatory.
- **SueSi (Sichere und Souveraene IT)**: The German government's strategy for digital sovereignty -- aligns well with FLOSS adoption.
- **BSI TR-02102**: Technical guideline for cryptographic algorithms and key lengths.
- **BSI TR-03116**: Cryptographic requirements for federal government projects.

---

## 3. Classification-Driven Architecture Principles

Following the cardinal rule that **classification drives architecture**, the VS-NfD level imposes these requirements:

### 3.1 Core Principles

1. **Data sovereignty**: All data must reside within Germany at all times. No data replication, backup, or transit outside German borders. No use of cloud services with foreign-jurisdiction data access (CLOUD Act risk).

2. **Network separation**: VS-NfD processing zones must be logically separated from unclassified networks. Physical separation is not mandated at VS-NfD (unlike VS-VERTRAULICH and above), but strong logical separation with BSI-approved encryption for inter-site transit is required.

3. **BSI-approved encryption for network transit**: Any VS-NfD data traversing network segments outside the physically controlled perimeter must use BSI-approved VS-NfD encryption products. Internal data center traffic within a controlled zone can use standard TLS, but WAN links require approved VPN.

4. **Personnel security**: All administrators with access to VS-NfD systems must hold appropriate Sicherheitsueberpruefung (security clearance), specifically Ue1 (basic check) at minimum, though Ue2 (extended check) is recommended for administrators.

5. **Need-to-know enforcement**: Access to VS-NfD data is restricted to personnel with both appropriate clearance AND a demonstrated need-to-know.

6. **Comprehensive audit trails**: All administrative actions, data access, and security-relevant events must be logged with attribution to individual users. Logs must be tamper-evident and retained per VSA requirements.

7. **Approved cryptography**: Cryptographic modules protecting VS-NfD data must either be BSI-approved for VS-NfD use or use algorithms conforming to BSI TR-02102 with FIPS 140-2/3 or Common Criteria validated implementations.

8. **Physical security**: Data center facilities must meet BSI requirements for VS-NfD processing, including access control, intrusion detection, and environmental protection.

### 3.2 What VS-NfD Does NOT Require (Unlike Higher Levels)

Understanding the boundaries prevents over-engineering:

- **Full air gap**: Not required. VS-NfD systems can be connected to other networks through approved security gateways.
- **TEMPEST shielding**: Not required at VS-NfD level (becomes relevant at VS-VERTRAULICH and above).
- **Separate physical hardware**: Not required for VS-NfD. Logical separation (dedicated VLANs, firewall zones, hypervisor isolation) is acceptable, though dedicated hardware simplifies compliance.
- **Type 1 cryptography**: Not required. BSI-approved VS-NfD products (a lower tier than VS-VERTRAULICH approved products) are sufficient.

---

## 4. FLOSS vs. BSI-Approved Products Decision Matrix

This is a critical decision point. The guiding principle is: **use FLOSS wherever the BSI framework does not explicitly require an approved product, and where the FLOSS component can be demonstrably hardened to meet IT-Grundschutz requirements.**

### 4.1 Detailed Layer-by-Layer Analysis

#### Layer: Hypervisor / Compute Virtualization
- **Decision**: FLOSS (KVM/QEMU via OpenStack Nova)
- **Rationale**: KVM is the Linux kernel's built-in hypervisor. It is widely deployed in government environments globally. There is no BSI requirement to use a specific hypervisor product for VS-NfD. The IT-Grundschutz Baustein SYS.1.5 (Virtualisierung) defines requirements that KVM can satisfy through proper configuration (CPU isolation, memory overcommit limits, secure live migration, etc.). SUSE Linux Enterprise Server and Red Hat Enterprise Linux -- both with KVM -- hold Common Criteria certifications that BSI recognizes.
- **Hardening**: Apply BSI SYS.1.5 requirements, disable unnecessary features, enforce sVirt/SELinux mandatory access control on VMs, limit live migration to encrypted channels within the secured zone.

#### Layer: Cloud Platform (IaaS Control Plane)
- **Decision**: FLOSS (OpenStack)
- **Rationale**: OpenStack provides the full IaaS stack. It is used by sovereign cloud initiatives across Europe (e.g., Gaia-X compatible deployments, SUSE OpenStack Cloud, Red Hat OpenStack Platform). There is no BSI-mandated IaaS product for VS-NfD. The control plane must be hardened and restricted to administrative access by cleared personnel.
- **Hardening**: TLS for all API endpoints, Keystone with multi-factor authentication, policy.yaml lockdown, RBAC with least privilege, API rate limiting, separate management network.

#### Layer: Operating System
- **Decision**: FLOSS (Linux), but use an enterprise distribution with security maintenance
- **Rationale**: BSI IT-Grundschutz Baustein SYS.1.3 (Linux-Server) provides explicit requirements for Linux servers. A maintained enterprise distribution (SUSE Linux Enterprise Server, Red Hat Enterprise Linux, or Ubuntu Pro) provides timely security updates and optional Common Criteria certification. Using a community distribution (Debian, AlmaLinux) is technically possible but complicates the audit since you must demonstrate equivalent patch management.
- **Recommendation**: **SUSE Linux Enterprise Server (SLES)** -- German company, Common Criteria certified, strong BSI relationship, available with FIPS-validated crypto modules, excellent OpenStack and Ceph support. Alternatively, **Red Hat Enterprise Linux (RHEL)** with its Common Criteria certification and established government use.
- **Hardening**: CIS benchmark level 2, OpenSCAP with BSI IT-Grundschutz profile, SELinux/AppArmor enforcing mode, AIDE for file integrity monitoring.

#### Layer: VPN / WAN Encryption
- **Decision**: **BSI-approved product REQUIRED**
- **Rationale**: This is non-negotiable. BSI maintains a list of VS-NfD-approved VPN/encryption products. Any VS-NfD data transiting networks outside the physically controlled data center perimeter must be encrypted using one of these products. Standard IPsec or WireGuard implementations, even if technically strong, are not approved and cannot be used.
- **Approved products (examples, verify current BSI list)**:
  - **genua genuscreen / genugate**: German manufacturer, VS-NfD approved firewall/VPN
  - **Rohde & Schwarz SITLine ETH**: VS-NfD approved Ethernet encryptor
  - **secunet SINA**: VS-NfD approved VPN and workstation solution
  - **Lancom Systems LCOS devices**: Select models VS-NfD approved for VPN
  - **NCP Secure VPN**: VS-NfD approved VPN client/gateway
- **Note**: The BSI VS-NfD approval list is updated regularly. Always verify current approvals at [BSI website](https://www.bsi.bund.de) before procurement.

#### Layer: Firewall / Perimeter Security
- **Decision**: **BSI-approved product strongly recommended** for the perimeter; FLOSS acceptable for internal segmentation
- **Rationale**: For the perimeter boundary between the VS-NfD zone and any lower-classification or external network, a BSI-certified or at minimum BSI-qualified firewall is strongly recommended to satisfy IT-Grundschutz NET.3.2 (Firewall). For internal microsegmentation within the VS-NfD zone, OpenStack security groups and OVN ACLs are acceptable.
- **Perimeter**: genua genugate, Rohde & Schwarz, or equivalent BSI-certified product
- **Internal**: OVN/OVS ACLs, OpenStack security groups, iptables/nftables

#### Layer: Hardware Security Module (HSM)
- **Decision**: **BSI-approved HSM REQUIRED**
- **Rationale**: Key material for VS-NfD encryption (TLS certificates for classified API endpoints, disk encryption keys, Barbican key storage) must be protected in hardware security modules that meet BSI requirements. Standard software keystores are insufficient for the key hierarchy root.
- **Approved products (examples)**:
  - **Utimaco SecurityServer**: German manufacturer, Common Criteria certified, BSI recognized
  - **Rohde & Schwarz TrustedObjects**: HSM product line
  - **Bundesdruckerei D-Trust HSM services**: German government-affiliated
- **Integration**: OpenStack Barbican with PKCS#11 backend connecting to the HSM cluster. All TLS certificate private keys stored in HSM. Ceph encryption keys wrapped by HSM-protected master keys.

#### Layer: Storage
- **Decision**: FLOSS (Ceph)
- **Rationale**: Ceph provides unified block (RBD for Cinder), object (RGW for Swift-compatible), and file (CephFS for Manila) storage. There is no BSI-mandated storage product. Encryption at rest is implemented using dm-crypt/LUKS on OSD devices with keys managed through the HSM-backed key management chain.
- **Hardening**: Encrypted OSDs, CephX authentication, network isolation of Ceph cluster traffic on dedicated VLAN, msgr2 protocol with TLS, separate public and cluster networks.

#### Layer: Networking (Internal/SDN)
- **Decision**: FLOSS (OVN/OVS via Neutron)
- **Rationale**: Open Virtual Network (OVN) with Open vSwitch (OVS) provides the software-defined networking for OpenStack Neutron. This handles tenant isolation, security groups, floating IPs, and load balancing (Octavia). There is no BSI requirement for a specific SDN product at VS-NfD level for internal data center networking.
- **Hardening**: Geneve tunnel encryption within the DC, strict security group defaults (deny-all), network isolation between tenants, dedicated provider networks for management and storage traffic.

#### Layer: Authentication Tokens / Smartcards
- **Decision**: **BSI-approved tokens REQUIRED** for personnel authentication
- **Rationale**: Access to VS-NfD systems must use strong multi-factor authentication. BSI requires government-approved smartcards or tokens for this purpose.
- **Products**: Government PKI smartcards (compatible with the Bundesdruckerei/D-Trust PKI infrastructure), or approved FIDO2/smartcard tokens that integrate with the government PKI.
- **Integration**: Keycloak as the internal IdP can broker authentication via smartcard/PKI, but the physical token itself must be BSI-approved.

#### Layer: Monitoring and Logging
- **Decision**: FLOSS (Prometheus, Grafana, Loki, Wazuh)
- **Rationale**: No BSI-mandated monitoring product. However, the monitoring and logging infrastructure must itself be protected at the VS-NfD level and must provide tamper-evident log storage.
- **Hardening**: Log forwarding over TLS, log integrity protection (append-only storage, cryptographic chaining), access to monitoring data restricted to cleared administrators, retention per VSA requirements.

#### Layer: Automation
- **Decision**: FLOSS (Ansible, OpenTofu)
- **Rationale**: Infrastructure as Code tools are not subject to BSI product approval. Ansible and OpenTofu are the recommended automation stack.
- **Hardening**: Ansible Vault for secrets, GitOps workflows with signed commits, AWX/Ansible Automation Platform with RBAC, audit trail for all automation runs.

### 4.2 Summary Decision Matrix

| Component | FLOSS | BSI-Approved | Product Category |
|-----------|-------|-------------|-----------------|
| IaaS Platform | Yes | -- | OpenStack |
| Hypervisor | Yes | -- | KVM/QEMU |
| Operating System | Yes* | -- | SLES or RHEL (enterprise FLOSS with CC cert) |
| Storage | Yes | -- | Ceph |
| SDN / Internal Networking | Yes | -- | OVN/OVS |
| Internal Load Balancer | Yes | -- | Octavia/HAProxy |
| DNS | Yes | -- | Designate/PowerDNS |
| Monitoring | Yes | -- | Prometheus/Grafana/Loki |
| SIEM | Yes | -- | Wazuh |
| Automation | Yes | -- | Ansible, OpenTofu |
| Identity Provider (internal) | Yes | -- | Keycloak |
| Container Registry | Yes | -- | Harbor |
| VPN Gateway | -- | **Required** | genua/secunet/R&S/NCP |
| Perimeter Firewall | -- | **Required** | genua genugate / equivalent |
| HSM | -- | **Required** | Utimaco / R&S |
| Authentication Tokens | -- | **Required** | Government PKI smartcards |
| WAN Encryptor | -- | **Required** | R&S SITLine / equivalent |

*SLES/RHEL are open-source based but commercially supported. Pure community distros (Debian, AlmaLinux) are technically usable but create additional audit burden.

---

## 5. Reference Architecture

### 5.1 High-Level Architecture Overview

```
                    +-----------------------------------------+
                    |         External Connectivity            |
                    |  (Bundesverwaltungsnetz / NdB / Internet)|
                    +-------------------+---------------------+
                                        |
                          +-------------v--------------+
                          |  BSI-Approved VPN Gateway  |
                          |  (genua/secunet/NCP)       |
                          +-------------+--------------+
                                        |
                          +-------------v--------------+
                          |  BSI-Approved Perimeter FW |
                          |  (genua genugate)          |
                          +-------------+--------------+
                                        |
              +-------------------------+-------------------------+
              |                                                   |
    +---------v-----------+                         +-------------v----------+
    |   Management Zone   |                         |    VS-NfD Tenant Zone  |
    |   (VLAN 100-109)    |                         |    (VLAN 200-299)      |
    |                     |                         |                        |
    | - OpenStack APIs    |                         | - Tenant workloads     |
    | - Keystone          |                         | - OVN overlay networks |
    | - Horizon           |                         | - Floating IPs         |
    | - Monitoring        |                         | - LBaaS (Octavia)     |
    | - Ansible/AWX       |                         |                        |
    | - Keycloak          |                         +------------------------+
    | - Log aggregation   |
    +---------+-----------+
              |
    +---------v-----------+
    |   Storage Zone      |
    |   (VLAN 300-309)    |
    |                     |
    | - Ceph MON/MGR/MDS  |
    | - Ceph OSD nodes    |
    | - Ceph RGW          |
    +---------+-----------+
              |
    +---------v-----------+
    |   IPMI/OOB Zone     |
    |   (VLAN 10)         |
    |                     |
    | - BMC/iLO/iDRAC     |
    | - Serial consoles   |
    +---------------------+
```

### 5.2 Network Zone Architecture

The architecture defines four primary security zones, aligned with IT-Grundschutz NET.1.1 (Netzarchitektur und -design):

**Zone 1 -- Perimeter / DMZ**
- BSI-approved VPN termination
- BSI-approved firewall
- Reverse proxy for any externally exposed services (if applicable)
- IDS/IPS

**Zone 2 -- Management Zone**
- OpenStack control plane (API services, schedulers, message queue, database)
- Monitoring and logging infrastructure
- Automation platform (AWX)
- Identity provider (Keycloak)
- Jump hosts / bastion hosts for administrative access
- Restricted to cleared administrators only

**Zone 3 -- Tenant / Workload Zone**
- OpenStack compute nodes running tenant VMs
- OVN overlay networking for tenant isolation
- Tenant-facing provider networks
- This is where VS-NfD workloads execute

**Zone 4 -- Storage Zone**
- Ceph cluster: MON, MGR, OSD, MDS, RGW nodes
- Dedicated storage network (no tenant traffic)
- Encrypted at rest on all OSDs

**Zone 5 -- Out-of-Band Management**
- IPMI/BMC network for hardware management
- Physically isolated (separate switch infrastructure recommended)
- Not routable from any other zone

### 5.3 Availability Architecture

To achieve 99.99% availability (52.6 minutes downtime per year):

**Control Plane HA:**
- 3x controller nodes in an anti-affinity group (separate racks, separate power feeds)
- HAProxy + keepalived for API load balancing
- MariaDB Galera cluster (3 nodes) for OpenStack database
- RabbitMQ cluster (3 nodes, quorum queues) for messaging
- Memcached cluster for token caching

**Compute HA:**
- Minimum 8 compute nodes across 2 availability zones (separate racks/power)
- Nova evacuate for VM recovery on compute node failure
- Masakari for automated instance HA (detects host failure, triggers evacuation)

**Storage HA:**
- Ceph cluster with minimum 3 MON nodes, 3 MGR nodes (colocated with MONs)
- OSD nodes across minimum 3 failure domains (racks)
- Replication factor 3 for VS-NfD data pools
- Erasure coding (4+2) for less critical data (monitoring archives, etc.)

**Network HA:**
- Dual top-of-rack switches per rack (bonded NIC connections from every server)
- Redundant spine switches
- MLAG/VPC for switch-level redundancy
- OVN active-standby controllers on 3 nodes

---

## 6. Compute Architecture

### 6.1 Hardware Specification

**Controller Nodes (3x):**
- 2x Intel Xeon Gold 6400+ series (or current generation)
- 512 GB DDR5 ECC RAM
- 2x 960 GB NVMe SSD (RAID1 for OS)
- 2x 3.84 TB NVMe SSD (for MariaDB, RabbitMQ, and ephemeral)
- 4x 25 GbE SFP28 NICs (2x bonded for tenant/API, 2x bonded for storage/management)
- 1x 1 GbE for IPMI/OOB
- Redundant PSU

**Compute Nodes (8x minimum, expandable):**
- 2x Intel Xeon Gold 6400+ series (or AMD EPYC 9004 series)
- 1 TB DDR5 ECC RAM (allowing for memory overcommit ratio of 1.5:1)
- 2x 960 GB NVMe SSD (RAID1 for OS)
- 2x 3.84 TB NVMe SSD (for local ephemeral storage)
- 4x 25 GbE SFP28 NICs (2x bonded for tenant overlay, 2x bonded for storage)
- 1x 1 GbE for IPMI/OOB
- Redundant PSU
- NUMA topology: pin VMs to NUMA nodes for performance-sensitive workloads

**Ceph Storage Nodes (5x minimum):**
- 2x Intel Xeon Silver 4400+ series (storage-optimized, less CPU needed)
- 256 GB DDR5 ECC RAM
- 2x 480 GB SATA SSD (RAID1 for OS)
- 10x 7.68 TB NVMe SSD (OSD drives -- all-flash for VS-NfD performance requirements)
- 2x 1.6 TB NVMe SSD (WAL/DB devices, partitioned across OSDs)
- 4x 25 GbE SFP28 NICs (2x bonded for Ceph public network, 2x bonded for Ceph cluster network)
- 1x 1 GbE for IPMI/OOB
- Redundant PSU

### 6.2 Hypervisor Configuration

- **SLES 15 SP6** (or current SP) with KVM/QEMU
- SELinux or AppArmor in enforcing mode (AppArmor is SUSE default)
- sVirt for mandatory access control on VM processes
- CPU microcode updates applied at boot
- IOMMU enabled for device isolation
- Hugepages (1 GB) pre-allocated for performance-critical VMs
- CPU pinning available via Nova flavor extra-specs for latency-sensitive workloads
- Live migration encrypted via TLS (QEMU native TLS)
- Nested virtualization disabled unless explicitly required
- Resource overcommit ratios:
  - CPU: 4:1 (adjustable per aggregate)
  - RAM: 1.5:1 (conservative for VS-NfD workloads)
  - Disk: no overcommit on Ceph-backed volumes

### 6.3 Nova Configuration Highlights

```ini
# /etc/nova/nova.conf (key security-relevant settings)

[DEFAULT]
# Disable metadata service on compute nodes where not needed
# Force config drive for metadata delivery (more secure than metadata service)
force_config_drive = true

[libvirt]
# Use QEMU with KVM acceleration
virt_type = kvm
# Enable TLS for live migration
live_migration_with_native_tls = true
# Dedicated migration network
live_migration_inbound_addr = <management-ip>

[filter_scheduler]
# Anti-affinity enforcement
enabled_filters = AvailabilityZoneFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,NUMATopologyFilter,AggregateInstanceExtraSpecsFilter

[pci]
# SR-IOV configuration if needed for specific workloads
# passthrough_whitelist = [{"vendor_id": "8086", "product_id": "..."}]
```

---

## 7. Network Architecture

### 7.1 Physical Network Design

**Spine-Leaf Topology:**

```
         +----------+     +----------+
         | Spine-1  |     | Spine-2  |
         | (MLAG)   +-----+ (MLAG)   |
         +----+-----+     +-----+----+
              |   \         /    |
              |    \       /     |
              |     \     /      |
         +----+---+  +--+-----+ +----+---+
         | Leaf-1 |  | Leaf-2 | | Leaf-3 |  ...
         | Rack-1 |  | Rack-2 | | Rack-3 |
         +--------+  +--------+ +--------+
```

- **Spine switches**: 2x high-port-count switches (e.g., 100 GbE fabric)
- **Leaf switches**: 2x per rack (25 GbE server-facing, 100 GbE uplinks to spine)
- **MLAG/VPC** between leaf pairs for active-active NIC bonding from servers
- **Separate physical switches** for the OOB/IPMI network (1 GbE, no connection to production fabric)

**Switch selection considerations for BSI compliance:**
- Switches themselves do not require BSI VS-NfD approval (they are within the physically secured zone)
- However, firmware must be from a trusted supply chain
- Consider European/German options where available, though Cisco Nexus 9000, Arista, or Dell OS10 switches are widely used in German government environments
- VXLAN-EVPN fabric if scaling beyond a single pair of spines

### 7.2 VLAN and Network Segment Design

| VLAN ID | Name | Purpose | Zone |
|---------|------|---------|------|
| 10 | OOB-MGMT | IPMI/BMC out-of-band management | Zone 5 |
| 100 | OPENSTACK-API | OpenStack API endpoints (internal) | Zone 2 |
| 101 | OPENSTACK-MGMT | Management plane (SSH, Ansible) | Zone 2 |
| 102 | OPENSTACK-DB | Database and message queue | Zone 2 |
| 103 | MONITORING | Prometheus, Loki, Grafana | Zone 2 |
| 104 | IDENTITY | Keycloak, LDAP | Zone 2 |
| 200 | TENANT-PROVIDER | Provider network for tenant VMs | Zone 3 |
| 201-249 | TENANT-VLAN-POOL | Neutron VLAN pool for tenant networks | Zone 3 |
| 300 | CEPH-PUBLIC | Ceph public network (client access) | Zone 4 |
| 301 | CEPH-CLUSTER | Ceph cluster network (replication) | Zone 4 |
| 400 | GENEVE-OVERLAY | Underlay for Geneve tunnels (OVN) | Zone 3 |
| 500 | PERIMETER | Connection to BSI-approved FW/VPN | Zone 1 |

### 7.3 OVN/OVS Configuration

OpenStack Neutron with ML2/OVN:

- **Tunnel type**: Geneve (preferred over VXLAN for OVN, supports larger metadata headers)
- **Distributed routing**: OVN distributed virtual router (DVR) on all compute nodes to minimize east-west traffic hairpinning
- **Security groups**: Implemented via OVN ACLs (stateful firewall rules per port)
- **Default security group policy**: Deny all ingress, allow all egress (tenants can customize, but deny-all ingress is enforced as baseline)
- **Floating IPs**: SNAT/DNAT via OVN gateway nodes (colocated with controllers)
- **DNS integration**: Neutron internal DNS + Designate for tenant DNS management
- **Metadata**: Config drive preferred over metadata service for security

### 7.4 Perimeter Security

```
Internet/External
       |
[BSI-Approved VPN] -- VS-NfD encrypted tunnel to remote sites
       |
[BSI-Approved FW]  -- Stateful inspection, IDS/IPS, application control
       |                Rule: deny all, explicit allow only
       |                Logging: all sessions logged to SIEM
       |
[Internal FW/ACL]  -- Additional filtering between zones
       |                Implemented via dedicated firewall or switch ACLs
       |
   Zone 2/3/4
```

The BSI-approved firewall (e.g., genua genugate) must:
- Perform stateful packet inspection
- Support application-layer filtering
- Provide IDS/IPS functionality or integrate with a separate IDS (Suricata)
- Log all connections to the central SIEM
- Be managed by cleared administrators only
- Have its configuration under change management with approval workflows

---

## 8. Storage Architecture

### 8.1 Ceph Cluster Design

**Cluster sizing (initial):**
- 5 storage nodes x 10 NVMe SSDs x 7.68 TB = 384 TB raw
- Replication factor 3: ~128 TB usable (for replicated pools)
- With erasure coding (8+3): ~280 TB usable (for EC pools)

**Pool design:**

| Pool | Type | Replication | Use |
|------|------|-------------|-----|
| rbd-vsnfd | Replicated | 3x | Cinder volumes for VS-NfD VMs |
| rbd-ephemeral | Replicated | 3x | Nova ephemeral disks |
| rbd-images | Replicated | 3x | Glance images |
| rgw-default | EC 4+2 | -- | Object storage (S3-compatible) |
| cephfs-data | Replicated | 3x | Manila shared filesystems |
| cephfs-metadata | Replicated | 3x | CephFS metadata pool |
| rbd-backup | EC 8+3 | -- | Backup target pool |

**CRUSH map:**
- Failure domain: rack (ensures 3 replicas are on 3 different racks)
- Device class: nvme (all devices)

### 8.2 Encryption at Rest

All Ceph OSD devices are encrypted using dm-crypt/LUKS2:

1. **Master key**: Stored in the BSI-approved HSM (Utimaco or equivalent)
2. **OSD encryption keys**: Generated per OSD, wrapped by the master key, stored in Ceph's key management (backed by Barbican with PKCS#11 HSM backend)
3. **Key rotation**: Supported via LUKS2 key slots -- new key added, old key removed, without re-encrypting the entire OSD
4. **Algorithm**: AES-256-XTS (conforming to BSI TR-02102)

```
HSM Master Key
     |
     v
Barbican (PKCS#11 backend)
     |
     v
dm-crypt/LUKS2 per OSD
     |
     v
Ceph OSD data on NVMe
```

### 8.3 Ceph Security Hardening

- CephX authentication enabled for all client connections
- Separate public and cluster networks (VLAN 300 and 301)
- Msgr2 protocol with TLS encryption for all Ceph daemon communication
- No direct client access to Ceph -- only through OpenStack APIs (Cinder, Glance, Manila, Swift/RGW)
- Ceph dashboard restricted to management zone, with Keycloak SSO and MFA
- RGW configured with STS (Secure Token Service) for temporary credentials
- Bucket policies enforced for object storage access control

---

## 9. Identity, Authentication, and Access Control

### 9.1 Identity Architecture

```
+---------------------------+
| BSI-Approved Smartcard    |  <-- Physical authentication factor
+------------+--------------+
             |
+------------v--------------+
| Keycloak (IdP)            |  <-- Central identity broker
| - Smartcard/PKI auth      |
| - TOTP as backup MFA      |
| - User federation (LDAP)  |
+------------+--------------+
             |
     +-------+-------+
     |               |
+----v----+   +------v------+
| Keystone |   | Other       |
| (OIDC    |   | services    |
|  federation) | (Grafana,   |
|          |   |  AWX, etc.) |
+----------+   +-------------+
```

### 9.2 Authentication Flow

1. User inserts BSI-approved smartcard and enters PIN
2. Keycloak validates the certificate chain against the government PKI (Bundesdruckerei/D-Trust CA)
3. Keycloak issues an OIDC token with claims including user identity, roles, and clearance attributes
4. OpenStack Keystone (configured as an OIDC SP) accepts the token and maps the user to Keystone projects/roles based on group membership
5. All subsequent API calls use scoped Keystone tokens with project-level RBAC

### 9.3 RBAC Model

**Keystone domains:**
- `bundesbehoerde`: Primary domain for the agency
- `service`: Domain for OpenStack service accounts

**Projects (examples):**
- `admin`: Platform administration
- `vsnfd-dept-a`: Department A's VS-NfD workloads
- `vsnfd-dept-b`: Department B's VS-NfD workloads
- `monitoring`: Monitoring and observability infrastructure
- `shared-services`: Shared services (DNS, registry, etc.)

**Roles:**
- `admin`: Full administrative access (cleared senior administrators only)
- `project-admin`: Tenant-level administration (VM lifecycle, network config within project)
- `member`: Standard user (can launch VMs, attach volumes, within project quotas)
- `reader`: Read-only access (for auditors and monitoring)

**Policy enforcement:**
- Custom policy.yaml files for each OpenStack service, restricting sensitive operations
- Service-to-service authentication via dedicated service accounts (no shared passwords)
- Token expiry: 4 hours (short-lived to limit exposure)
- Application credentials for automated workloads (scoped, time-limited where possible)

### 9.4 Privileged Access Management

- All administrative access via jump host / bastion in management zone
- Session recording (using `tlog` or equivalent) for all administrative SSH sessions
- Break-glass procedure for emergency access (documented, audited, requires two-person authorization)
- No direct root login; all access via named user accounts with sudo (logged)
- Separate administrative accounts for different roles (no single account with access to everything)

---

## 10. Encryption and Cryptography

### 10.1 BSI-Compliant Cryptographic Standards

All cryptographic implementations must conform to **BSI TR-02102** (current version). Key requirements:

| Purpose | Algorithm | Key Length | BSI TR-02102 Status |
|---------|-----------|-----------|-------------------|
| Symmetric encryption | AES | 256-bit | Recommended through 2030+ |
| Key exchange | ECDH | P-384 / brainpoolP384r1 | Recommended |
| Digital signatures | ECDSA | P-384 / brainpoolP384r1 | Recommended |
| Digital signatures | RSA | 3072-bit minimum | Acceptable through 2030 |
| Hash function | SHA-384 / SHA-512 | -- | Recommended |
| TLS protocol | TLS 1.2 (minimal) / TLS 1.3 (preferred) | -- | TLS 1.3 recommended |
| Disk encryption | AES-XTS | 256-bit | Recommended |

**BSI brainpool curves**: BSI has historically preferred brainpoolP256r1/P384r1/P512r1 curves (German-developed, no NSA involvement in parameter selection). While NIST P-curves are also accepted by BSI TR-02102, using brainpool curves may be viewed favorably in the audit. OpenSSL supports brainpool curves.

### 10.2 Encryption Matrix

| Data State | Mechanism | Key Management |
|-----------|-----------|---------------|
| Data at rest (Ceph OSD) | dm-crypt/LUKS2 AES-256-XTS | HSM-wrapped keys via Barbican |
| Data at rest (OS disks) | LUKS2 on root/boot (or Clevis+Tang for auto-unlock) | Tang server in management zone, HSM-backed |
| Data in transit (API) | TLS 1.3 with ECDHE+AES-256-GCM | Certificates from internal CA (Keycloak/Smallstep CA), root CA in HSM |
| Data in transit (Ceph) | Msgr2 with TLS | Internal CA certificates |
| Data in transit (WAN) | BSI-approved VPN (IPsec with approved crypto) | VPN appliance internal key management + HSM |
| Data in transit (OVN) | Geneve with IPsec (optional, for high-security tenants) | IPsec managed by OVN, certificates from internal CA |
| Secrets at rest | Barbican + HSM backend | HSM master key |
| Database (MariaDB) | TLS for connections, TDE optional | Internal CA certificates |
| Backup data | AES-256-GCM encrypted backups | Backup encryption keys in Barbican/HSM |

### 10.3 PKI Architecture

```
BSI-Approved HSM (Root of Trust)
     |
     +-- Internal Root CA (offline, in HSM)
          |
          +-- Intermediate CA - Infrastructure
          |    |
          |    +-- OpenStack API TLS certificates
          |    +-- Ceph daemon TLS certificates
          |    +-- Database TLS certificates
          |    +-- Monitoring TLS certificates
          |
          +-- Intermediate CA - Workloads
          |    |
          |    +-- Tenant-facing TLS certificates
          |    +-- Service mesh certificates (if applicable)
          |
          +-- Intermediate CA - Personnel
               |
               +-- User authentication certificates (on smartcards)
               +-- (Bridged to government PKI via cross-certification)
```

Certificate lifecycle managed by **cert-manager** (for Kubernetes workloads if deployed) or **Smallstep CA** (for infrastructure certificates), with private keys generated in and never leaving the HSM.

---

## 11. Logging, Monitoring, and Audit

### 11.1 Monitoring Stack

```
+-------------------+     +-------------------+     +-------------------+
|  Prometheus       |     |  Loki             |     |  Wazuh Manager    |
|  (Metrics)        |     |  (Logs)           |     |  (SIEM/HIDS)      |
+--------+----------+     +--------+----------+     +--------+----------+
         |                         |                          |
         +------------+------------+--------------------------+
                      |
              +-------v-------+
              |   Grafana     |
              |  (Dashboards) |
              +---------------+
```

**Prometheus** (metrics):
- Node exporter on all hosts
- OpenStack exporter for API metrics
- Ceph exporter for storage metrics
- Alertmanager for notification routing
- Thanos for long-term metric storage (on Ceph RGW)
- Retention: 15 days local, 2 years in Thanos/S3

**Loki** (logs):
- Promtail agents on all hosts
- All OpenStack service logs
- All OS syslog/journal logs
- Firewall logs from BSI-approved appliances (via syslog forwarding)
- Retention: 2 years (VS-NfD audit requirements)
- Tamper protection: append-only storage on Ceph, cryptographic log chaining

**Wazuh** (SIEM/HIDS):
- Agent on every host
- File integrity monitoring (FIM) on critical system files
- Rootkit detection
- OpenSCAP integration for continuous compliance checking against BSI IT-Grundschutz
- Active response capability (disabled by default, enable for specific scenarios after risk assessment)
- Correlation rules for security event detection
- Integration with Grafana for unified dashboards

### 11.2 Audit Requirements (IT-Grundschutz OPS.1.1.5)

The following events must be logged with attribution to individual users:

- All authentication attempts (success and failure)
- All authorization decisions (RBAC checks)
- All administrative actions (VM create/delete, network changes, user management)
- All data access events (volume attach/detach, object read/write)
- All security-relevant configuration changes
- All privilege escalation events (sudo usage)
- Firewall rule changes and connection logs
- VPN tunnel establishment and teardown

**Log protection:**
- Logs forwarded in real-time to the centralized log infrastructure
- Local log retention as buffer (7 days)
- Centralized logs stored on Ceph with replication factor 3
- Cryptographic integrity protection (HMAC-based log chaining)
- Access to log data restricted to designated security officers
- Log data classified at the same level as the data it describes (VS-NfD)

### 11.3 Compliance Dashboards

Grafana dashboards providing real-time visibility into:

1. **BSI IT-Grundschutz Compliance Score**: OpenSCAP scan results aggregated across all hosts
2. **C5 Control Status**: Mapping of C5 criteria to technical controls with pass/fail indicators
3. **Security Events**: Wazuh alerts, failed logins, privilege escalation attempts
4. **Infrastructure Health**: Compute, storage, network health and capacity
5. **Certificate Expiry**: Tracking all TLS certificates with expiry warnings (30/14/7 day alerts)
6. **Encryption Status**: Verification that all data-at-rest and data-in-transit encryption is active
7. **Patch Status**: OS and package update status across all nodes

---

## 12. Automation and Infrastructure as Code

### 12.1 Automation Architecture

```
+-------------------+     +-------------------+
|  Git Repository   |     |  AWX / AAP        |
|  (GitLab CE,      |---->|  (Job execution)  |
|   self-hosted)    |     |                   |
+-------------------+     +--------+----------+
                                   |
                    +--------------+--------------+
                    |              |              |
              +-----v----+  +-----v----+  +------v-----+
              | Ansible   |  | OpenTofu |  | Packer     |
              | Playbooks |  | Modules  |  | (Images)   |
              +----------+  +----------+  +------------+
```

### 12.2 Ansible Roles and Playbooks

**Infrastructure deployment roles:**
- `os-hardening`: CIS Level 2 + BSI IT-Grundschutz hardening for SLES
- `openstack-deploy`: Kolla-Ansible based OpenStack deployment
- `ceph-deploy`: cephadm-based Ceph deployment with encryption
- `monitoring-deploy`: Prometheus, Loki, Grafana, Wazuh deployment
- `keycloak-deploy`: Keycloak installation and federation configuration
- `network-config`: Switch configuration (if using Ansible for network devices)
- `compliance-scan`: OpenSCAP BSI IT-Grundschutz profile scanning

**Day-2 operations playbooks:**
- `patch-management`: Rolling OS updates with pre/post checks
- `certificate-rotation`: Automated certificate renewal
- `user-onboarding`: Keycloak user provisioning with role assignment
- `backup-verify`: Backup integrity verification
- `dr-failover`: Disaster recovery failover procedure
- `capacity-report`: Generate capacity utilization reports

**Security automation:**
- All playbooks stored in Git with signed commits
- Ansible Vault for all secrets (vault password stored in HSM-backed secret manager)
- AWX RBAC: operators can run approved playbooks but cannot edit them
- Audit trail: AWX logs all job executions with user attribution

### 12.3 OpenTofu for Infrastructure Definition

OpenTofu (FLOSS fork of Terraform) manages:
- OpenStack project/tenant provisioning
- Quota management
- Network topology (provider networks, routers, security groups)
- Flavor and image management
- DNS zone management (Designate)

State backend: OpenStack Swift (Ceph RGW) with state locking via Consul or PostgreSQL.

### 12.4 Golden Image Pipeline

```
Packer build (SLES base)
     |
     +-- Apply CIS hardening (Ansible provisioner)
     +-- Install Wazuh agent
     +-- Install Prometheus node-exporter
     +-- Install OpenSCAP
     +-- Remove unnecessary packages
     +-- Run OpenSCAP scan (fail build if critical findings)
     |
     v
Upload to Glance (signed image)
     |
     v
Available for tenant VM creation
```

Images rebuilt monthly (or on critical CVE) and old images deprecated after 90 days.

---

## 13. Disaster Recovery and Business Continuity

### 13.1 RTO/RPO Targets

| Component | RPO | RTO | Strategy |
|-----------|-----|-----|----------|
| OpenStack control plane | 1 hour | 4 hours | DB backup + Ansible rebuild |
| VS-NfD tenant VMs | 1 hour | 4 hours | Ceph RBD snapshots + cross-site replication |
| Ceph storage cluster | 0 (synchronous within site) | 2 hours | Triple replication + rebuild |
| Monitoring/logging | 4 hours | 8 hours | Rebuild from Ansible, data loss acceptable for short period |
| Full site loss | 24 hours | 48 hours | DR site with asynchronous Ceph replication |

### 13.2 Backup Strategy

**Backup targets:**
- OpenStack databases: MariaDB logical backup (mysqldump) + Galera SST backup, every 1 hour, retained 30 days
- OpenStack configuration: Stored in Git (Ansible), immutable by design
- Ceph data: RBD snapshots (hourly for VS-NfD pools, daily for others), replicated to DR site
- Keycloak database: PostgreSQL pg_dump, every 1 hour, retained 30 days
- Monitoring data: Thanos compactor for long-term metric retention, Loki compactor for log retention

**Backup encryption:** All backups encrypted with AES-256-GCM, keys in Barbican/HSM.

**Backup verification:** Weekly automated restore test to a sandboxed environment. Monthly manual DR drill.

### 13.3 DR Site Architecture

- Second data center within Germany (minimum 50 km distance for disaster scenarios, maximum 200 km for latency on synchronous replication if desired)
- Ceph RBD mirroring (asynchronous, RPO ~1 hour) to DR site
- Standby OpenStack control plane at DR site (cold standby, activated via Ansible)
- BSI-approved VPN between primary and DR sites for replication traffic
- DR site must meet the same BSI IT-Grundschutz and physical security requirements as the primary

---

## 14. Physical Security and Data Center Requirements

### 14.1 BSI Requirements for VS-NfD Processing Facilities

Per IT-Grundschutz INF.2 (Rechenzentrum) and VSA requirements:

**Access control:**
- Multi-factor physical access control (badge + PIN or biometric)
- Mantrap/airlock entry to the server room
- Visitor escort policy (no uncleared visitors unescorted in VS-NfD areas)
- Access logs retained for minimum 2 years
- 24/7 security monitoring (CCTV with recording, intrusion detection)

**Environmental:**
- Redundant cooling (N+1 minimum)
- Redundant power (UPS + diesel generator, minimum 72 hours fuel)
- Fire detection and suppression (gas-based, e.g., Novec 1230)
- Water leak detection
- Environmental monitoring (temperature, humidity)

**Physical separation:**
- VS-NfD equipment in dedicated cages or rooms (depending on facility classification)
- Cable routing documented and controlled (no shared conduits with external tenants if in a colocation)
- Equipment disposal: BSI-compliant media destruction (DIN 66399 security level per classification)

### 14.2 Data Center Selection Criteria

For a Bundesbehoerde, the data center should be:

1. **Located within Germany** -- non-negotiable for VS-NfD
2. **Operated by a German entity** -- not subject to foreign jurisdiction data access laws
3. **BSI-compliant** -- either BSI-certified or auditable to BSI standards
4. **Ideally a government facility** -- Bundesrechenzentrum, ITZBund facility, or agency-owned
5. **If commercial**: Tier III minimum, ISO 27001 certified, C5 attested, with contractual guarantees for VS-NfD handling

**Recommended approach**: Use an ITZBund (Informationstechnikzentrum Bund) facility or a dedicated government data center. If using a commercial provider, ensure contractual provisions include:
- BSI audit rights
- Personnel security clearance requirements for facility staff
- Data sovereignty guarantees
- Physical isolation of VS-NfD equipment
- Compliance with VSA requirements

---

## 15. BSI IT-Grundschutz Implementation Roadmap

### 15.1 Process Overview

The IT-Grundschutz certification process follows BSI-Standard 200-2:

```
Phase 1: ISMS Establishment (BSI-Standard 200-1)
    |
    v
Phase 2: Structural Analysis (Strukturanalyse)
    |-- Identify all information assets
    |-- Map business processes to IT systems
    |-- Document network topology
    |-- Inventory hardware, software, rooms, personnel
    |
    v
Phase 3: Protection Needs Assessment (Schutzbedarfsfeststellung)
    |-- Classify each asset by confidentiality, integrity, availability
    |-- VS-NfD assets: "hoch" (high) for confidentiality
    |-- Determine protection level per asset
    |
    v
Phase 4: Modelling (Modellierung)
    |-- Map IT-Grundschutz Bausteine to each asset
    |-- Select applicable requirements (Anforderungen) per Baustein
    |-- This produces the complete control set for the environment
    |
    v
Phase 5: IT-Grundschutz Check (IT-Grundschutz-Check)
    |-- Assess implementation status of each requirement
    |-- Gap analysis: identify unimplemented or partially implemented controls
    |
    v
Phase 6: Risk Analysis (BSI-Standard 200-3)
    |-- For any gaps or elevated-risk components
    |-- Determine residual risk and additional measures
    |
    v
Phase 7: Implementation
    |-- Close all gaps
    |-- Document implementation evidence
    |
    v
Phase 8: Audit and Certification
    |-- BSI-licensed auditor conducts the audit
    |-- Findings remediation
    |-- BSI issues certificate (valid 3 years, annual surveillance)
```

### 15.2 Key IT-Grundschutz Bausteine for This Architecture

The following Bausteine from the IT-Grundschutz Compendium (Edition 2025) are most relevant:

**Process Layer (ISMS):**
- ISMS.1: Sicherheitsmanagement
- ORP.1: Organisation
- ORP.2: Personal
- ORP.3: Sensibilisierung und Schulung
- ORP.4: Identitaets- und Berechtigungsmanagement
- CON.1: Kryptokonzept
- CON.3: Datensicherungskonzept
- CON.6: Loeschen und Vernichten
- OPS.1.1.2: Ordnungsgemaesse IT-Administration
- OPS.1.1.3: Patch- und Aenderungsmanagement
- OPS.1.1.5: Protokollierung
- OPS.1.1.6: Software-Tests und -Freigaben
- OPS.1.2.5: Fernwartung
- DER.1: Detektion von sicherheitsrelevanten Ereignissen
- DER.2.1: Behandlung von Sicherheitsvorfaellen
- DER.4: Notfallmanagement

**System Layer:**
- SYS.1.1: Allgemeiner Server
- SYS.1.3: Server unter Linux und Unix
- SYS.1.5: Virtualisierung
- SYS.1.6: Containerisierung (if containers are used)
- SYS.1.8: Speicherloesungen
- SYS.1.9: Terminalserver (if applicable)
- APP.4.4: Kubernetes (if K8s is used for platform services)

**Network Layer:**
- NET.1.1: Netzarchitektur und -design
- NET.1.2: Netzmanagement
- NET.3.1: Router und Switches
- NET.3.2: Firewall
- NET.3.3: VPN
- NET.4.1: TK-Anlagen (if VoIP is in scope)

**Infrastructure Layer:**
- INF.1: Allgemeines Gebaeude
- INF.2: Rechenzentrum sowie Serverraum
- INF.12: Verkabelung

**Application Layer:**
- APP.2.2: MariaDB
- APP.4.3: Relationale Datenbanksysteme
- APP.6: Allgemeine Software

### 15.3 Timeline Estimate

| Phase | Duration | Notes |
|-------|----------|-------|
| ISMS establishment | 2-3 months | Policy framework, roles, responsibilities |
| Structural analysis + protection needs | 2-3 months | Comprehensive inventory and classification |
| Modelling | 1-2 months | Baustein mapping, control selection |
| IT-Grundschutz check (gap analysis) | 2-3 months | Current vs. required state |
| Implementation (gap closure) | 6-12 months | Technical and organizational measures |
| Internal audit | 1-2 months | Pre-audit preparation |
| External audit | 2-3 months | BSI-licensed auditor |
| BSI review and certification | 2-3 months | BSI reviews auditor report |
| **Total** | **18-30 months** | From project start to certification |

**Recommendation**: Begin the ISMS establishment and structural analysis in parallel with the technical platform build. The platform will not be fully operational before the ISMS framework is needed, and early engagement with the BSI methodology ensures the architecture aligns with control requirements from the start.

---

## 16. C5 Attestation Process

### 16.1 C5 vs. IT-Grundschutz

C5 and IT-Grundschutz are complementary, not competing:

- **IT-Grundschutz**: Covers the entire ISMS and all IT infrastructure -- broad and deep
- **C5**: Specifically designed for cloud services -- focuses on cloud-specific risks and controls

For a private cloud operated by a Bundesbehoerde for internal use, **IT-Grundschutz is the primary framework**. However, **C5 attestation is additionally valuable** because:
1. It demonstrates cloud-specific security to stakeholders and oversight bodies
2. If the agency later offers cloud services to other agencies, C5 is expected
3. C5 maps well to international standards (ISO 27017, SOC 2), aiding interoperability assessments
4. BSI increasingly expects C5 for government cloud deployments

### 16.2 C5 Attestation Process

```
Step 1: Scope Definition
    |-- Define the cloud services in scope (IaaS: compute, storage, networking)
    |-- Define the system boundaries
    |-- Identify the applicable C5 criteria
    |
    v
Step 2: Control Implementation
    |-- Implement all applicable C5 criteria (basic + additional for VS-NfD)
    |-- Document the System Description (Systembeschreibung)
    |-- This overlaps significantly with IT-Grundschutz implementation
    |
    v
Step 3: Type 1 Attestation (optional but recommended)
    |-- Engage a Wirtschaftspruefer (auditor) qualified for C5
    |-- Auditor assesses control design at a point in time
    |-- Produces a SOC 2-style report confirming design suitability
    |-- Duration: 2-4 months
    |
    v
Step 4: Operate controls for minimum 6 months
    |-- Demonstrate operating effectiveness
    |-- Collect evidence continuously
    |
    v
Step 5: Type 2 Attestation
    |-- Auditor assesses design AND operating effectiveness
    |-- Review period: minimum 6 months, typically 12 months
    |-- Produces the Type 2 attestation report
    |-- Duration: 3-5 months for the audit itself
    |
    v
Step 6: Ongoing
    |-- Annual Type 2 attestation renewal
    |-- Continuous monitoring and evidence collection
```

### 16.3 C5 Additional Criteria for VS-NfD

C5:2020 defines "additional criteria" (Zusatzkriterien) that apply for higher protection needs. For VS-NfD processing, the following additional criteria are particularly relevant:

- **CKM-04 (additional)**: Use of evaluated/certified cryptographic modules
- **COS-05 (additional)**: Network separation with certified components
- **IDM-08 (additional)**: Strong authentication (smartcard/PKI)
- **PS-08 (additional)**: Enhanced physical security for classified processing
- **OPS-15 (additional)**: Enhanced vulnerability management
- **LOG-04 (additional)**: Extended logging with integrity protection
- **HR-04 (additional)**: Personnel security clearances

### 16.4 Selecting a C5 Auditor

The C5 attestation must be performed by a Wirtschaftspruefer (WP) or Wirtschaftspruefungsgesellschaft (WPG) following IDW PS 951 n.F. (the German adaptation of ISAE 3402). Key auditors with C5 experience include:
- The Big Four (PwC, Deloitte, EY, KPMG) -- all have dedicated German C5 practices
- BDO, Mazars, Grant Thornton -- mid-tier firms with C5 capabilities
- Ensure the specific audit team has demonstrated C5 experience

---

## 17. Supply Chain and Procurement

### 17.1 Hardware Supply Chain

For VS-NfD environments, supply chain integrity is important:

- **Server hardware**: Procure from established vendors with documented supply chains. Dell, HPE, Lenovo, and Fujitsu (Japanese but with significant German operations) are all used in German government. Fujitsu has historically had strong ties to German government IT.
- **Avoid**: Hardware from vendors subject to foreign government-mandated backdoor risks. Consult BSI guidance on specific vendor risks.
- **Firmware integrity**: Verify firmware signatures, use vendor-provided integrity checking tools, consider UEFI Secure Boot with custom keys.
- **Chain of custody**: Document hardware receipt, serial numbers, and placement. Tamper-evident packaging for shipments.

### 17.2 Software Supply Chain

- **Operating system**: SLES or RHEL from official repositories, with GPG signature verification
- **OpenStack**: Packages from the distribution vendor (SUSE OpenStack Cloud or Red Hat OpenStack Platform) or from upstream Kolla images with signature verification
- **Ceph**: Packages from the distribution vendor or official Ceph repositories (GPG signed)
- **All other FLOSS**: From official distribution repositories or verified upstream sources
- **SBOM**: Generate Software Bill of Materials for the entire platform stack
- **Vulnerability scanning**: Automated scanning of all packages against CVE databases (integrated into CI/CD and day-2 operations)

### 17.3 BSI-Approved Product Procurement

BSI-approved products (VPN, firewall, HSM) must be procured through approved channels:

1. Verify the product is on the current BSI approval list
2. Procure from the manufacturer or an authorized reseller
3. Verify serial numbers and firmware versions against BSI-approved configurations
4. Some products require specific firmware versions to maintain approval -- do not update firmware without checking BSI status
5. Maintain support contracts as required by the BSI approval conditions
6. Budget for recurring license/maintenance costs (these are typically not FLOSS)

### 17.4 Procurement Framework

As a Bundesbehoerde, procurement must follow public procurement law (VgV/UVgO). Relevant considerations:

- **EVB-IT contracts**: Use the standard EVB-IT contract templates for IT procurement
- **Rahmenvertraege**: Check existing framework contracts (e.g., through the Beschaffungsamt des BMI) for BSI-approved products -- often more cost-effective and faster
- **FLOSS procurement**: No licensing costs, but budget for support contracts (SUSE, Red Hat) and professional services for deployment
- **Total budget estimate**:

| Category | Estimated Cost (Initial) | Annual Recurring |
|----------|------------------------|-----------------|
| Server hardware (16 nodes) | 800,000 - 1,200,000 EUR | 120,000 EUR (maintenance) |
| Network switches (spine-leaf) | 150,000 - 250,000 EUR | 30,000 EUR (SmartNet/support) |
| BSI-approved VPN (2x HA pair) | 80,000 - 150,000 EUR | 20,000 EUR (maintenance) |
| BSI-approved firewall (2x HA) | 60,000 - 120,000 EUR | 15,000 EUR (maintenance) |
| BSI-approved HSM (2x HA) | 60,000 - 100,000 EUR | 15,000 EUR (maintenance) |
| SLES subscriptions (16 nodes) | -- | 40,000 EUR/year |
| Professional services (deployment) | 200,000 - 400,000 EUR | -- |
| BSI IT-Grundschutz consulting | 150,000 - 300,000 EUR | 50,000 EUR (annual audit support) |
| C5 attestation (auditor fees) | -- | 100,000 - 200,000 EUR/year |
| Data center costs (colocation or internal) | varies | varies |
| **Total (estimate)** | **1,500,000 - 2,520,000 EUR** | **390,000 - 690,000 EUR/year** |

---

## 18. Operational Model and Personnel

### 18.1 Team Structure

| Role | Count | Clearance | Responsibilities |
|------|-------|-----------|-----------------|
| Cloud Platform Lead | 1 | Ue2 | Architecture decisions, BSI coordination, stakeholder management |
| OpenStack Engineer | 2 | Ue2 | OpenStack operations, upgrades, troubleshooting |
| Ceph Storage Engineer | 1 | Ue2 | Ceph operations, capacity management, performance tuning |
| Linux System Engineer | 2 | Ue2 | OS hardening, patching, automation development |
| Network Engineer | 1 | Ue2 | Physical and virtual networking, firewall management |
| Security Officer (ISO) | 1 | Ue2 | ISMS management, IT-Grundschutz compliance, incident response |
| IT Security Officer (IT-SiBe) | 1 | Ue2 | Technical security, vulnerability management, audit support |
| **Total core team** | **9** | | |

**Additional support (can be part-time or shared):**
- Data Protection Officer (DSB) -- required by GDPR, can be shared across the agency
- BSI auditor liaison
- Procurement specialist for IT hardware

### 18.2 Personnel Security

- All team members must hold **Ue2 (erweiterte Sicherheitsueberpruefung)** at minimum
- Security clearance process managed through the agency's Geheimschutzbeauftragte(r)
- Clearance processing time: 3-6 months (plan for this in staffing timelines)
- Foreign nationals: additional restrictions apply; consult with the Geheimschutzbeauftragte(r)
- Contractor/vendor access: vendors performing work on VS-NfD systems must hold appropriate clearances; consider this when selecting professional services partners

### 18.3 On-Call and Incident Response

- 24/7 on-call rotation for platform availability (minimum 3 engineers in rotation)
- All on-call personnel must have Ue2 clearance
- Remote access for on-call: via BSI-approved VPN client (e.g., NCP or secunet SINA Workstation) to jump host
- Incident classification aligned with BSI IT-Grundschutz DER.2.1
- Security incidents involving potential VS-NfD compromise: immediate notification to agency's Geheimschutzbeauftragte(r) and BSI if required

---

## 19. Landing Zone Design

### 19.1 Tenant Onboarding

Each department or workload group receives a standardized "landing zone" in OpenStack:

```
Project: vsnfd-dept-X
|
+-- Network: dept-x-internal (10.X.0.0/16, OVN private network)
|   +-- Subnet: dept-x-app (10.X.1.0/24)
|   +-- Subnet: dept-x-db  (10.X.2.0/24)
|   +-- Subnet: dept-x-mgmt (10.X.3.0/24)
|
+-- Router: dept-x-router
|   +-- External gateway to provider network (NAT)
|   +-- Routes to shared services
|
+-- Security Groups:
|   +-- sg-default (deny all ingress, allow all egress)
|   +-- sg-ssh-from-bastion (allow SSH from bastion only)
|   +-- sg-web (allow 443 from designated sources)
|   +-- sg-db (allow DB port from app subnet only)
|
+-- Quotas:
|   +-- instances: 20
|   +-- cores: 80
|   +-- ram: 160 GB
|   +-- volumes: 50
|   +-- volume_storage: 5 TB
|   +-- floating_ips: 5
|   +-- security_groups: 20
|
+-- Roles:
    +-- dept-x-admin: project admin role
    +-- dept-x-user: member role
    +-- dept-x-reader: reader role
```

### 19.2 Shared Services

Available to all tenants:

- **DNS**: Designate (internal DNS zones per project, forwarding to agency DNS)
- **Load Balancing**: Octavia (project-scoped LBaaS)
- **Object Storage**: Ceph RGW (S3-compatible, per-project buckets with quotas)
- **Shared Filesystem**: Manila with CephFS backend (for workloads needing shared NFS-like storage)
- **Image Registry**: Glance with curated, hardened images
- **Container Registry**: Harbor (if container workloads are planned)
- **Key Management**: Barbican (project-scoped secrets and encryption keys)

---

## 20. Risk Assessment

### 20.1 Key Risks and Mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | BSI certification takes longer than planned | High | Medium | Start ISMS work in parallel with technical build; engage BSI early for informal guidance |
| R2 | Skilled personnel shortage (OpenStack + Ue2 clearance) | High | High | Begin clearance processes early; consider SUSE/Red Hat professional services (with cleared staff) |
| R3 | BSI-approved product compatibility issues with OpenStack | Medium | High | Test integration in lab environment before procurement commitment; select products with documented OpenStack/Linux integration |
| R4 | Ceph performance insufficient for workload demands | Low | Medium | All-flash design mitigates this; performance testing during Phase 2 before production |
| R5 | OpenStack upgrade complexity | Medium | Medium | Use Kolla-Ansible for lifecycle management; test upgrades in staging; maintain N-1 version support |
| R6 | Supply chain compromise of hardware/software | Low | Critical | Verify firmware signatures, maintain SBOM, scan for vulnerabilities, use trusted procurement channels |
| R7 | VS-NfD data breach | Low | Critical | Defense in depth: encryption, access control, monitoring, incident response, regular penetration testing |
| R8 | Single data center failure | Low | Critical | DR site with async replication; tested failover procedures |
| R9 | Vendor lock-in despite FLOSS strategy | Low | Medium | Maintain abstraction layers; document all proprietary integration points; ensure portability |
| R10 | C5 audit findings require architecture changes | Medium | Medium | Engage C5 auditor for pre-assessment (readiness check) before formal attestation |

### 20.2 Risk Treatment

All risks rated "High" impact require a dedicated risk treatment plan documented per BSI-Standard 200-3. Risks R2, R6, and R7 should be prioritized for treatment planning before platform deployment begins.

---

## 21. Implementation Phases

### Phase 0: Foundation (Months 1-3)

- Establish ISMS framework (BSI-Standard 200-1)
- Hire and clear core team (begin Ue2 clearance process immediately -- this is often the longest lead item)
- Procure data center space (if not already available)
- Begin structural analysis (BSI-Standard 200-2)
- Design detailed architecture (refine this document into implementation specifications)
- Procure lab/staging hardware (can be smaller scale)
- Engage BSI-Grundschutz consultant

**Exit criteria**: ISMS policy framework approved, team hiring initiated, data center secured, architecture finalized.

### Phase 1: Lab and Proof of Concept (Months 3-6)

- Deploy OpenStack in lab environment (Kolla-Ansible on minimal hardware)
- Integrate Ceph storage
- Test BSI-approved VPN/firewall integration
- Test HSM integration with Barbican
- Test Keycloak + smartcard authentication
- Validate encryption at rest and in transit
- Performance baseline testing
- Develop Ansible roles for hardening and deployment
- Begin IT-Grundschutz modelling (Baustein mapping)

**Exit criteria**: Working lab environment demonstrating all key integrations, performance baselines documented, Baustein mapping complete.

### Phase 2: Production Build (Months 6-12)

- Procure production hardware
- Deploy production network (spine-leaf, BSI-approved perimeter)
- Deploy production OpenStack (3 controllers, 8+ compute, 5 storage)
- Implement all security controls (encryption, access control, logging, monitoring)
- Harden all systems per IT-Grundschutz requirements
- Deploy monitoring and SIEM
- Implement automation (Ansible, OpenTofu, GitOps workflows)
- IT-Grundschutz check (gap analysis)
- Close identified gaps

**Exit criteria**: Production platform operational, all IT-Grundschutz controls implemented, gap analysis shows no critical gaps.

### Phase 3: Tenant Onboarding and Operations (Months 12-15)

- Deploy landing zones for initial tenants
- Migrate pilot workloads to the platform
- Operational procedures finalized and documented
- Runbooks created for all common operations
- On-call rotation established
- DR site deployment begins
- Begin C5 Type 1 preparation

**Exit criteria**: Pilot workloads running in production, operations team self-sufficient, DR site under construction.

### Phase 4: Certification and Attestation (Months 15-24)

- Internal IT-Grundschutz pre-audit
- Remediate any findings
- External IT-Grundschutz audit by BSI-licensed auditor
- BSI certification submission
- C5 Type 1 attestation (can run in parallel)
- Operate controls for 6+ months
- C5 Type 2 attestation
- DR site operational and tested

**Exit criteria**: BSI IT-Grundschutz certificate issued, C5 Type 2 attestation report delivered.

### Phase 5: Steady State and Continuous Improvement (Month 24+)

- Full tenant onboarding
- Continuous compliance monitoring
- Annual IT-Grundschutz surveillance audit
- Annual C5 Type 2 renewal
- Capacity expansion as needed
- OpenStack lifecycle management (annual upgrades)
- Continuous security improvement

---

## 22. Architectural Decision Records

### ADR-001: OpenStack as IaaS Platform

**Status**: Accepted

**Context**: The agency needs a private cloud IaaS platform for VS-NfD workloads. Options considered: OpenStack, Proxmox VE, VMware vSphere, Kubernetes-only (KubeVirt).

**Decision**: OpenStack with Kolla-Ansible deployment.

**Rationale**:
- Full-featured IaaS with API-driven self-service (Horizon, CLI, SDK)
- Strong alignment with European digital sovereignty initiatives (Gaia-X, SCS)
- FLOSS -- no licensing costs, no vendor lock-in
- Mature ecosystem with enterprise support options (SUSE, Red Hat, Canonical)
- Proven in government and classified environments globally
- Comprehensive networking (Neutron/OVN), storage (Cinder/Ceph), and identity (Keystone) services
- VMware was rejected due to Broadcom acquisition uncertainty, licensing costs, and vendor lock-in
- Proxmox was considered but lacks the API maturity and tenant isolation model needed for multi-department use
- Kubernetes-only was rejected because not all agency workloads are containerized; VMs remain the primary consumption model

**Consequences**: Requires OpenStack expertise on the team. More operational complexity than Proxmox or VMware. Mitigated by Kolla-Ansible for lifecycle management and SUSE/Red Hat professional services.

### ADR-002: Ceph for Unified Storage

**Status**: Accepted

**Context**: The platform needs block, object, and file storage. Options: Ceph, commercial SAN/NAS, MinIO (object only), GlusterFS.

**Decision**: Ceph as the unified storage backend for all OpenStack storage services.

**Rationale**:
- Single storage platform for block (RBD/Cinder), object (RGW/Swift), and file (CephFS/Manila)
- FLOSS, no licensing costs
- Proven at scale in OpenStack deployments
- Encryption at rest via dm-crypt/LUKS2 integration
- Excellent replication and self-healing capabilities
- Strong community and enterprise support (Red Hat, SUSE, Canonical)
- DR replication (RBD mirroring) built in

### ADR-003: BSI-Approved Products at Network Boundary

**Status**: Accepted

**Context**: VS-NfD data must be encrypted in transit with BSI-approved products when leaving the physically controlled zone. The question was whether we could use standard IPsec/WireGuard.

**Decision**: BSI-approved VPN (e.g., genua genuscreen, secunet SINA, or NCP) and BSI-approved firewall (e.g., genua genugate) at the network perimeter.

**Rationale**:
- BSI regulations are non-negotiable for VS-NfD network transit encryption
- Standard IPsec implementations (StrongSwan, etc.) are not on the BSI approval list
- Using non-approved products would invalidate the VS-NfD processing authorization
- The cost is manageable and limited to the perimeter (internal traffic uses standard TLS)

### ADR-004: SLES as Base Operating System

**Status**: Accepted

**Context**: Need a Linux distribution for all platform nodes. Options: SLES, RHEL, Ubuntu, Debian, AlmaLinux.

**Decision**: SUSE Linux Enterprise Server (SLES).

**Rationale**:
- German company (SUSE SE, headquartered in Nuremberg)
- Common Criteria certified
- Strong BSI relationship and understanding of German government requirements
- Excellent OpenStack and Ceph support (SUSE has dedicated cloud products)
- FIPS-validated cryptographic modules available
- Long-term support (LTSS) available
- Aligns with digital sovereignty goals
- RHEL is an acceptable alternative if the agency has existing Red Hat infrastructure

### ADR-005: Keycloak for Identity Brokering

**Status**: Accepted

**Context**: Need centralized identity management that integrates BSI-approved smartcard authentication with OpenStack Keystone and other platform services.

**Decision**: Keycloak as the central identity broker, federating to Keystone via OIDC.

**Rationale**:
- FLOSS (Apache 2.0 license)
- Supports smartcard/X.509 certificate authentication natively
- OIDC and SAML 2.0 support for federation to downstream services
- User federation with LDAP/AD for existing agency directory
- Fine-grained authorization policies
- Well-maintained by Red Hat / community
- Does NOT replace the BSI-approved smartcard/token requirement -- it brokers the authentication

---

## 23. Appendix: BSI IT-Grundschutz Module Mapping

The following table maps key IT-Grundschutz Bausteine to specific components in this architecture. This serves as the starting point for the formal Modellierung phase.

| Baustein | Applies To | Key Requirements | Implementation |
|----------|-----------|-----------------|----------------|
| SYS.1.1 | All servers | Hardening, patching, logging | CIS benchmark, OpenSCAP, Wazuh |
| SYS.1.3 | All Linux hosts | Linux-specific hardening | AppArmor, SLES hardening, auditd |
| SYS.1.5 | Compute nodes (KVM) | Hypervisor security, VM isolation | sVirt, NUMA isolation, encrypted migration |
| SYS.1.8 | Ceph cluster | Storage security, availability | Encryption, CephX, replication factor 3 |
| NET.1.1 | Entire network | Segmentation, zone model | Spine-leaf, VLANs, OVN isolation |
| NET.3.1 | All switches/routers | Switch hardening, access control | 802.1X, ACLs, firmware management |
| NET.3.2 | Perimeter + internal FW | Firewall policies, logging | genua genugate + OVN security groups |
| NET.3.3 | WAN connectivity | VS-NfD approved VPN | genua/secunet/NCP approved VPN |
| INF.2 | Data center | Physical security, environment | Access control, UPS, fire suppression |
| ORP.4 | All systems | Identity and access management | Keycloak + smartcard + Keystone RBAC |
| CON.1 | All encrypted systems | Cryptographic concept | BSI TR-02102 compliance, HSM |
| CON.3 | All data | Backup concept | Ceph snapshots, encrypted backups, DR |
| OPS.1.1.2 | Operations team | Administrative procedures | Documented runbooks, change management |
| OPS.1.1.3 | All systems | Patch management | Automated via Ansible, staged rollout |
| OPS.1.1.5 | All systems | Logging | Loki + Wazuh, tamper-evident, 2yr retention |
| DER.1 | All systems | Security event detection | Wazuh SIEM, Prometheus alerts |
| DER.2.1 | Operations | Incident response | IR plan, notification chain, BSI reporting |
| DER.4 | All systems | Emergency management | DR plan, tested failover, BC procedures |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-20 | Cloud Architecture Team | Initial architecture document |

---

*This document is classified as VS-NfD upon completion with agency-specific details. This template version contains no classified information.*
