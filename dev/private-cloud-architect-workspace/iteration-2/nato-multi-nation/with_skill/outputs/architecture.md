# NATO SECRET Multi-Nation Shared Infrastructure Architecture

## Document Control

| Field | Value |
|-------|-------|
| Classification | NATO SECRET equivalent (architecture document itself to be treated as nationally restricted) |
| Date | 2026-03-20 |
| Status | Draft Architecture |
| Participating Nations | Norway (NOR), Denmark (DNK), Germany (DEU) |
| Hosting Location | LIST-X equivalent facility, Norway |
| Target Classification | NATO SECRET / National equivalents up to SECRET |

---

## 1. Executive Summary

This document defines the architecture for a multinational NATO SECRET-level shared infrastructure platform hosted in a LIST-X equivalent facility in Norway. The platform serves a joint defense project involving Norway, Denmark, and Germany, processing both NATO-classified and nationally-classified data up to the SECRET level.

The core architectural challenge is threefold:

1. **NATO SECRET processing** -- a shared environment where all three nations collaborate on NATO SECRET data under NATO security policy C-M(2002)49.
2. **National enclave isolation** -- each nation requires a segregated enclave for nationally-classified data (Norwegian HEMMELIG, Danish HEMMELIGT, German GEHEIM) that is accessible only to that nation's cleared personnel.
3. **Controlled cross-domain data sharing** -- mechanisms to move data between national enclaves and the shared NATO domain, subject to release authority and sanitization.

The architecture uses a physically segmented, defense-in-depth approach with separate compute and network infrastructure per security domain, hardware-enforced data diodes and cross-domain solutions for inter-domain transfers, and a unified management framework operated by multinational cleared personnel.

---

## 2. Regulatory and Accreditation Framework

### 2.1 Applicable Frameworks

Each participating nation and NATO itself impose requirements that must be satisfied simultaneously.

#### NATO
- **Classification**: NATO SECRET (fourth tier in NATO UNCLASSIFIED / NATO RESTRICTED / NATO CONFIDENTIAL / NATO SECRET / COSMIC TOP SECRET)
- **Governing policy**: C-M(2002)49 (NATO Security Policy), AC/35 series directives
- **Accreditation authority**: NATO Communications and Information Agency (NCIA), in coordination with the host nation's National Security Authority (NSA) and participating nations' NSAs
- **Cryptographic requirements**: NATO-approved cryptographic products only for NATO SECRET data in transit and at rest
- **TEMPEST**: NATO SDIP-27 compliance required (Zone A or Zone B depending on facility construction)

#### Norway (Host Nation)
- **Classification**: HEMMELIG (equivalent to SECRET)
- **Authority**: NSM (Nasjonal Sikkerhetsmyndighet / National Security Authority)
- **Framework**: Norwegian Security Act (Sikkerhetsloven), NSM ICT security guidelines
- **Facility**: LIST-X equivalent accreditation under Norwegian law; NSM conducts physical and IT security inspections
- **Crypto**: NSM-approved cryptographic products for national data
- **Personnel**: Norwegian security clearances issued by NSM; foreign personnel require bilateral security agreements

#### Denmark
- **Classification**: HEMMELIGT (equivalent to SECRET)
- **Authority**: CFCS (Center for Cybersikkerhed)
- **Framework**: Danish Security Act, CFCS guidelines
- **Crypto**: CFCS-approved cryptographic products for Danish national data
- **Key consideration**: Danish data sovereignty requirements may mandate that certain nationally-classified data remains under Danish cryptographic protection even when stored in Norway; bilateral security agreement with Norway governs this

#### Germany
- **Classification**: GEHEIM (equivalent to SECRET)
- **Authority**: BSI (Bundesamt fur Sicherheit in der Informationstechnik) for IT security; BMVg (Federal Ministry of Defence) for military classification
- **Framework**: BSI IT-Grundschutz (comprehensive, thousands of controls), VSA (Verschlusssachenanweisung -- Classified Information Instruction)
- **Crypto**: BSI-approved cryptographic products; VS-NfD and GEHEIM require distinct crypto solutions
- **Key consideration**: IT-Grundschutz is extremely prescriptive; the German enclave must satisfy IT-Grundschutz controls in full, which may exceed NATO or Norwegian baseline requirements. German data sovereignty provisions under the VSA may require German-national-only key management for German GEHEIM data.

### 2.2 Bilateral and Multilateral Security Agreements

Before any technical implementation, the following agreements must be in place:

1. **General Security Agreement (GSA)** between Norway-Denmark, Norway-Germany, and Denmark-Germany (all three pairs likely already exist as NATO allies, but must be verified as current)
2. **Programme Security Instruction (PSI)** specific to this defense project, defining classification levels, handling procedures, and access rules
3. **Communication and Information Systems Security Memorandum of Understanding (CIS SEMU)** covering the shared IT infrastructure
4. **Accreditation agreement** designating the Security Accreditation Authority (SAA) -- typically the host nation NSA (NSM) as lead, with participating nations' NSAs as co-accreditors for their respective national enclaves
5. **TEMPEST agreement** for the shared facility, with NSM as the TEMPEST authority for the Norwegian site

### 2.3 Accreditation Strategy

The platform requires multiple overlapping accreditations:

| Domain | Accreditation Authority | Framework |
|--------|------------------------|-----------|
| NATO SECRET shared domain | NCIA + NSM (host nation lead) | C-M(2002)49, AC/35 |
| Norwegian national enclave | NSM | Sikkerhetsloven, NSM guidelines |
| Danish national enclave | CFCS (with NSM as host oversight) | Danish Security Act, CFCS guidelines |
| German national enclave | BSI/BMVg (with NSM as host oversight) | VSA, IT-Grundschutz |
| Cross-domain solutions | Joint SAA (all four authorities) | Combined assessment |
| Physical facility | NSM | Norwegian LIST-X equivalent standards |

Accreditation follows the universal pattern: Categorize, Select Controls, Implement, Assess, Authorize, Monitor. Design for continuous monitoring (phase 6) from inception.

---

## 3. Physical Architecture

### 3.1 Facility Requirements

The LIST-X equivalent facility in Norway must meet the most stringent requirements across all applicable frameworks:

- **Physical perimeter**: Controlled access area with 24/7 guard force, intrusion detection, CCTV with classified-rated recording and retention
- **Access control**: Multi-factor authentication (badge + biometric), mantrap entry, visitor escort requirements
- **Zoning**: The facility must be internally divided into zones corresponding to each security domain:
  - Zone N-NATO: NATO SECRET shared processing area
  - Zone N-NOR: Norwegian national enclave
  - Zone N-DNK: Danish national enclave
  - Zone N-DEU: German national enclave
  - Zone MGMT: Management and operations area (cleared to NATO SECRET minimum)
  - Zone XD: Cross-domain solution room (highest physical security, dual-person access)
- **TEMPEST**: NATO SDIP-27 Zone A or Zone B. Given that this is a purpose-built classified facility, Zone B (within a controlled area with defined boundary) is the minimum. Each zone must meet TEMPEST requirements independently. Cable routing must enforce red/black separation -- classified (red) and unclassified (black) signals must never share conduits.
- **Power**: Redundant power feeds, UPS (N+1 minimum), diesel generator with 72-hour fuel autonomy. Each security domain should have independent power distribution units (PDUs) to prevent side-channel leakage across domains.
- **Fire suppression**: Gas-based (FM-200 or Novec 1230) to protect equipment without water damage. Independent zones per security domain.
- **Environmental**: Precision cooling per zone, independent HVAC to prevent cross-domain acoustic or emanation leakage through shared ductwork.

### 3.2 Rack Layout and Physical Separation

Each security domain occupies physically separate racks, ideally in separate caged areas within the facility:

```
+------------------------------------------------------------------+
|                     LIST-X FACILITY (Norway)                      |
|                                                                   |
|  +------------+  +------------+  +------------+  +------------+   |
|  |  NATO SEC  |  |  NOR HEMM  |  |  DNK HEMM  |  |  DEU GEH   |  |
|  |  (Shared)  |  |  (Enclave) |  |  (Enclave) |  |  (Enclave) |  |
|  |            |  |            |  |            |  |            |   |
|  | Racks 1-8  |  | Racks 9-12|  | Racks13-16 |  | Racks17-20 |  |
|  +------+-----+  +------+-----+  +------+-----+  +------+-----+  |
|         |               |               |               |         |
|         +-------+-------+-------+-------+-------+-------+         |
|                 |               |               |                 |
|           +-----+-----+  +-----+-----+  +-----+-----+           |
|           |  XD: CDS   |  |  XD: CDS   |  |  XD: CDS   |        |
|           | NATO<->NOR |  | NATO<->DNK |  | NATO<->DEU |         |
|           +-----------+  +-----------+  +-----------+            |
|                                                                   |
|  +------------------------------------------------------------+  |
|  |              MANAGEMENT / OPS ZONE                          |  |
|  |  (KVM consoles, monitoring terminals, ops workstations)     |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

**Critical physical constraints:**
- Minimum 1 meter separation between racks of different security domains, or use of shielded cabinets
- Separate cable trays per domain, color-coded (e.g., RED for NATO SECRET, BLUE for NOR, GREEN for DNK, YELLOW for DEU)
- No shared patch panels between domains
- Cross-domain solution equipment in a dedicated locked cage with dual-person integrity (two-person rule for physical access)

---

## 4. Network Architecture

### 4.1 Network Segmentation Model

Each security domain has a **physically separate network fabric**. There are no shared switches, routers, or cables between domains. VLANs and firewalls alone do not constitute domain separation at the SECRET level -- physical air gaps are mandatory.

```
                    DOMAIN NETWORK TOPOLOGY

  +------------------+     +------------------+
  |  NATO SECRET     |     |  NOR HEMMELIG    |
  |  Network Fabric  |     |  Network Fabric  |
  |                  |     |                  |
  |  Spine-Leaf      |     |  Spine-Leaf      |
  |  (Dedicated HW)  |     |  (Dedicated HW)  |
  +--------+---------+     +--------+---------+
           |                         |
    [Data Diode/CDS]          [Data Diode/CDS]
           |                         |
  +--------+---------+     +--------+---------+
  |  DNK HEMMELIGT   |     |  DEU GEHEIM      |
  |  Network Fabric  |     |  Network Fabric  |
  |                  |     |                  |
  |  Spine-Leaf      |     |  Spine-Leaf      |
  |  (Dedicated HW)  |     |  (Dedicated HW)  |
  +------------------+     +------------------+

  NOTE: Cross-domain connections are NOT direct network
  links. They pass through hardware-enforced CDS/data
  diodes in the XD zone. Each domain fabric is fully
  independent.
```

### 4.2 Per-Domain Network Design

Each domain's network fabric follows a spine-leaf architecture for predictable latency and scalability.

#### Hardware Selection

For a classified environment of this scale, Cisco Nexus 9000 series switches are recommended for each domain's fabric:

- **Spine**: 2x Cisco Nexus 9332D-GX2B per domain (providing redundancy)
- **Leaf**: 2-4x Cisco Nexus 93180YC-FX3 per domain (depending on rack count)
- **Management**: Dedicated out-of-band management switches per domain (Cisco Nexus 3048 or equivalent)

**Alternative (FLOSS-friendly)**: For nations or procurement frameworks that prefer open networking, Edgecore or Dell OS10-based switches running SONiC (Software for Open Networking in the Cloud) could be evaluated. However, in classified environments, vendor support and supply chain assurance typically favor Cisco, Juniper, or nationally-approved vendors. The supply chain integrity requirements at SECRET level may restrict open networking hardware options.

#### Network Design Per Domain

Each domain implements:

- **VXLAN-EVPN fabric** for flexible L2/L3 segmentation within the domain
- **BGP underlay** between spine and leaf (eBGP or iBGP depending on scale)
- **EVPN overlay** for tenant/workload isolation within each domain
- **Dedicated management network** (out-of-band) for IPMI/iLO/iDRAC/CIMC
- **Storage network** (dedicated VLAN or separate physical NIC for Ceph traffic)
- **Internal segmentation**: Even within a single classified domain, workloads should be micro-segmented using network policies (Cilium or Calico if running Kubernetes) or ACI contracts (if using ACI)

#### IP Addressing

Each domain uses non-overlapping RFC 1918 address space to simplify cross-domain routing rules in the CDS:

| Domain | Underlay | Overlay (Workloads) | Management | Storage |
|--------|----------|---------------------|------------|---------|
| NATO SECRET | 10.10.0.0/16 | 10.11.0.0/16 | 10.12.0.0/24 | 10.13.0.0/24 |
| NOR HEMMELIG | 10.20.0.0/16 | 10.21.0.0/16 | 10.22.0.0/24 | 10.23.0.0/24 |
| DNK HEMMELIGT | 10.30.0.0/16 | 10.31.0.0/16 | 10.32.0.0/24 | 10.33.0.0/24 |
| DEU GEHEIM | 10.40.0.0/16 | 10.41.0.0/16 | 10.42.0.0/24 | 10.43.0.0/24 |

### 4.3 External Connectivity

#### NATO Wide Area Network

The NATO SECRET domain connects to NATO's classified wide area network (likely via a NATO NCIA-provided circuit) for collaboration with other NATO sites:

- **Circuit**: Encrypted link using NATO-approved Type 1 crypto (e.g., NATO CERTUS or equivalent bulk encryptors)
- **Termination**: Dedicated router/firewall in the NATO SECRET zone
- **Firewall**: Stateful inspection firewall with NATO-approved ruleset; all traffic logged and auditable
- **DNS**: Internal DNS for the NATO domain; no external DNS resolution

#### National Reachback

Each national enclave may require connectivity back to national classified networks:

- **Norway**: NSM-approved encrypted link to Norwegian HEMMELIG network
- **Denmark**: CFCS-approved encrypted link to Danish HEMMELIGT network
- **Germany**: BSI-approved encrypted link to German GEHEIM network (likely using BSI-approved VPN devices, e.g., SINA or equivalent)

Each national reachback link terminates in the respective national enclave only. There is no cross-connection between national reachback links and the NATO shared domain except through the CDS.

### 4.4 DNS and Time Synchronization

- **DNS**: Each domain runs its own internal DNS infrastructure (CoreDNS or BIND, deployed as part of the platform). No DNS queries cross domain boundaries.
- **NTP**: Each domain has its own NTP source. For TEMPEST and emanation security, GPS-disciplined NTP appliances are preferred (no internet NTP). All domains must be synchronized to a common time reference for audit log correlation -- a shared GPS receiver with physically separate NTP distribution to each domain achieves this without creating a network bridge.

---

## 5. Compute Architecture

### 5.1 Hardware Selection

Each security domain has its own dedicated compute nodes. No hardware is shared across domains.

#### Recommended Server Platform

For a classified environment, server selection must consider supply chain integrity, firmware security, and national approval:

- **Primary recommendation**: HPE ProLiant DL360 Gen11 or Dell PowerEdge R660 -- both offer comprehensive security features (Silicon Root of Trust / iDRAC Secure Boot), wide NATO/defense sector usage, and national supply chain availability
- **Alternative**: Cisco UCS C240 M7 if Cisco is the preferred compute vendor for Intersight-based management
- **TEMPEST consideration**: Depending on the SDIP-27 zone assessment, TEMPEST-rated server variants may be required. Vendors like Elbit Systems, General Dynamics, or L3Harris provide TEMPEST-rated enclosures or modified COTS servers

#### Sizing Per Domain

| Domain | Compute Nodes | vCPUs per Node | RAM per Node | Purpose |
|--------|---------------|----------------|--------------|---------|
| NATO SECRET (shared) | 8 | 64 (2x 32-core Xeon) | 512 GB | Primary workload processing |
| NOR HEMMELIG | 4 | 64 | 512 GB | Norwegian national workloads |
| DNK HEMMELIGT | 4 | 64 | 512 GB | Danish national workloads |
| DEU GEHEIM | 4 | 64 | 512 GB | German national workloads |
| CDS / Management | 4 | 32 | 256 GB | Cross-domain, monitoring, ops |

Total: ~24 servers (scalable by adding nodes per domain).

#### Firmware and Supply Chain

- All servers must be procured through a verified supply chain with tamper-evident packaging
- Firmware must be verified against vendor-published checksums before deployment
- UEFI Secure Boot enabled on all nodes
- TPM 2.0 modules installed and used for measured boot and disk encryption key sealing
- BMC/IPMI access restricted to the domain-specific management network; default credentials changed; IPMI-over-LAN encrypted

### 5.2 Hypervisor / Platform Strategy

Given the multi-nation classified environment, the platform must balance security, manageability, and compliance across different national frameworks.

#### Recommended: Kubernetes with KubeVirt on hardened Linux

**Rationale:**
- A Kubernetes-native platform allows running both containerized workloads and traditional VMs (via KubeVirt) on a single infrastructure
- Kubernetes provides declarative, auditable infrastructure management aligned with GitOps
- Hardened Kubernetes distributions (RKE2, Talos Linux) have FIPS-validated configurations and are gaining traction in defense environments
- KubeVirt enables legacy VM workloads alongside modern container workloads

**Distribution selection:**
- **RKE2** (Rancher Government): FIPS-validated, DISA STIG-hardened, CIS benchmark compliant. Strong fit for NATO/defense environments.
- **Talos Linux**: Immutable OS with API-only management (no SSH, no shell). Extremely hardened by design. Excellent for classified environments where attack surface minimization is paramount.

**Recommendation**: Use **Talos Linux** as the base OS and Kubernetes distribution for all domains. Its immutable, API-only design radically reduces attack surface and aligns with the principle that classified infrastructure should be minimal and auditable. For national enclaves with specific requirements (e.g., Germany's IT-Grundschutz may require specific OS certifications), **RKE2 on RHEL 9** (Common Criteria certified, FIPS-validated) is the fallback.

#### Alternative: OpenStack

If the project requires traditional IaaS with full VM self-service (and the team has OpenStack expertise), OpenStack deployed via Kolla-Ansible on hardened RHEL/Rocky Linux is a viable alternative. OpenStack provides Keystone (identity), Nova (compute), Neutron (networking), and Cinder (storage) with mature multi-tenancy. However, the operational overhead of OpenStack is significantly higher than Kubernetes+KubeVirt, and the classified defense community is increasingly standardizing on Kubernetes.

### 5.3 Operating System Hardening

Regardless of platform choice:

- **Base OS**: RHEL 9 (Common Criteria certified, FIPS-validated) or Talos Linux (immutable)
- **Hardening**: CIS Benchmark Level 2 at minimum; apply national hardening guides where they exceed CIS
  - Norway: NSM baseline security guidelines
  - Denmark: CFCS hardening recommendations
  - Germany: BSI IT-Grundschutz SYS modules (extremely detailed)
- **Scanning**: OpenSCAP with DISA STIG profiles for continuous compliance validation; automated scanning on every boot and on a daily schedule
- **Patching**: Disconnected patching via local repositories (Pulp or Spacewalk/Uyuni). All patches verified by GPG signature before application. Patch windows coordinated across nations per the PSI.

---

## 6. Storage Architecture

### 6.1 Storage Strategy

Each security domain has its own independent storage cluster. No storage is shared across domains.

#### Recommended: Ceph (deployed via Rook on Kubernetes)

**Rationale:**
- Ceph provides unified block (RBD), object (RGW), and file (CephFS) storage
- Rook deploys and manages Ceph as a Kubernetes-native operator
- No licensing costs (FLOSS)
- Self-healing, distributed, and scalable
- Widely deployed in defense and government environments
- Encryption at rest via dm-crypt/LUKS on OSDs with keys sealed to TPM

#### Storage Sizing Per Domain

| Domain | OSD Nodes | Drives per Node | Drive Type | Raw Capacity | Usable (3x replication) |
|--------|-----------|-----------------|------------|--------------|------------------------|
| NATO SECRET | 4 | 8x 3.84TB NVMe | NVMe SSD | 122 TB | ~40 TB |
| NOR HEMMELIG | 3 | 6x 3.84TB NVMe | NVMe SSD | 69 TB | ~23 TB |
| DNK HEMMELIGT | 3 | 6x 3.84TB NVMe | NVMe SSD | 69 TB | ~23 TB |
| DEU GEHEIM | 3 | 6x 3.84TB NVMe | NVMe SSD | 69 TB | ~23 TB |

**Note**: OSD nodes can be the same physical servers as compute nodes (hyperconverged) or dedicated storage nodes. For classified environments, hyperconverged simplifies the physical footprint and reduces the number of devices requiring accreditation.

#### Encryption at Rest

- All Ceph OSDs encrypted with dm-crypt/LUKS
- Encryption keys sealed to TPM 2.0 (keys never leave the TPM in plaintext)
- Key management: Each domain uses its own key hierarchy. National enclaves use nationally-approved key management:
  - NATO domain: NATO-approved KMS
  - NOR enclave: NSM-approved key management
  - DNK enclave: CFCS-approved key management
  - DEU enclave: BSI-approved key management (likely integrated with SINA or VS-NfD-approved HSMs)

### 6.2 Backup Architecture

- **Intra-domain backup**: Velero for Kubernetes resource backup; Ceph RBD snapshots for persistent volumes
- **Backup storage**: Separate backup pool within each domain's Ceph cluster, or dedicated NFS/S3 backup target
- **Backup encryption**: All backups encrypted with domain-specific keys
- **Cross-domain backup**: Backups never cross domain boundaries. Each domain's data is backed up within its own domain.
- **Offsite backup**: If required by the PSI, encrypted backup media can be physically transported to a secondary LIST-X facility. This requires sanitization review and chain-of-custody documentation.

---

## 7. Security Domain Architecture

### 7.1 Domain Model

The architecture defines five security domains:

```
+-------------------------------------------------------------------+
|                                                                   |
|  +-----------------------------------------------------------+   |
|  |                    NATO SECRET DOMAIN                       |   |
|  |                                                             |   |
|  |  - Shared compute, storage, network                        |   |
|  |  - NATO SECRET data processing                             |   |
|  |  - Multi-nation personnel access (NOR, DNK, DEU)           |   |
|  |  - NATO-approved crypto for data at rest and in transit     |   |
|  |  - Connected to NATO WAN                                   |   |
|  +----------------------------+--------------------------------+   |
|                               |                                   |
|              +----------------+------------------+                |
|              |                |                  |                |
|         [CDS/Guard]     [CDS/Guard]        [CDS/Guard]           |
|              |                |                  |                |
|  +-----------+--+ +----------+---+ +------------+---+            |
|  | NOR HEMMELIG | | DNK HEMMELIGT| | DEU GEHEIM     |            |
|  |              | |              | |                |            |
|  | - NOR only   | | - DNK only   | | - DEU only     |            |
|  | - NOR crypto | | - DNK crypto | | - DEU/BSI      |            |
|  | - NSM accred | | - CFCS accred| |   crypto       |            |
|  | - NOR WAN    | | - DNK WAN    | | - BSI accred   |            |
|  |   reachback  | |   reachback  | | - DEU WAN      |            |
|  +--------------+ +--------------+ |   reachback    |            |
|                                    +----------------+            |
|                                                                   |
|  +-----------------------------------------------------------+   |
|  |                  MANAGEMENT DOMAIN                          |   |
|  |  - Infrastructure monitoring (read-only views per domain)  |   |
|  |  - Physical facility systems (HVAC, power, access control) |   |
|  |  - NOT connected to any classified domain's data plane     |   |
|  +-----------------------------------------------------------+   |
+-------------------------------------------------------------------+
```

### 7.2 Access Control Model

#### Personnel Clearances

| Domain | Required Clearance | Nations Allowed |
|--------|-------------------|-----------------|
| NATO SECRET shared | NATO SECRET (any participating nation) | NOR, DNK, DEU |
| NOR HEMMELIG enclave | Norwegian HEMMELIG + project need-to-know | NOR only |
| DNK HEMMELIGT enclave | Danish HEMMELIGT + project need-to-know | DNK only |
| DEU GEHEIM enclave | German GEHEIM + project need-to-know | DEU only |
| Management domain | NATO SECRET minimum | Designated ops personnel (any nation) |
| CDS room | NATO SECRET + specific CDS operator role | Designated personnel (dual-person) |

#### Identity and Authentication

Each domain runs its own identity infrastructure. There is no federated identity across domains (by design -- federation would create a cross-domain dependency that undermines isolation).

- **NATO SECRET domain**: FreeIPA or Red Hat IdM (RHEL-based) as the identity provider. LDAP/Kerberos for authentication. Multi-factor authentication mandatory (smart card / CAC + PIN).
- **National enclaves**: Each national enclave runs its own FreeIPA/IdM instance. Users are provisioned independently in each domain they are authorized to access.
- **Kubernetes RBAC**: Kubernetes OIDC integration with the domain's IdP. Namespace-level RBAC with least privilege. Service accounts with short-lived tokens.
- **Workstation access**: TEMPEST-rated KVM switches allow operators to access multiple domains from a single physical desk, with hard-switched (not software-switched) selection between domains. Each domain has its own keyboard, mouse, and display signal path.

### 7.3 Cryptographic Architecture

| Domain | Data at Rest | Data in Transit | Key Management | Approved By |
|--------|-------------|-----------------|----------------|-------------|
| NATO SECRET | AES-256 (NATO-approved implementation) | TLS 1.3 with NATO-approved cipher suites; Type 1 for WAN | NATO-approved KMS / HSM | NCIA |
| NOR HEMMELIG | AES-256 (NSM-approved) | TLS 1.3 with NSM-approved config | NSM-approved KMS | NSM |
| DNK HEMMELIGT | AES-256 (CFCS-approved) | TLS 1.3 with CFCS-approved config | CFCS-approved KMS | CFCS |
| DEU GEHEIM | AES-256 (BSI-approved) | TLS 1.3 with BSI-approved config | BSI-approved HSM (e.g., Utimaco or Rohde & Schwarz) | BSI |

**Critical note**: "AES-256" is necessary but not sufficient. Each national authority approves specific *implementations* (specific hardware or software modules, specific firmware versions). The crypto products must be on each authority's approved list. For Germany at GEHEIM level, this likely requires BSI-evaluated hardware security modules.

---

## 8. Cross-Domain Solutions (CDS)

### 8.1 Overview

Cross-domain data sharing is the most architecturally sensitive component. The CDS enables controlled, auditable transfer of data between:

- NATO SECRET <-> NOR HEMMELIG
- NATO SECRET <-> DNK HEMMELIGT
- NATO SECRET <-> DEU GEHEIM

**Direct transfers between national enclaves (e.g., NOR <-> DEU) are not permitted.** All inter-nation sharing goes through the NATO SECRET shared domain, where the data is reclassified as NATO SECRET and made available to all participating nations. This is a deliberate architectural constraint that enforces the NATO sharing model.

### 8.2 CDS Architecture

Each domain pair (NATO <-> National) has a dedicated cross-domain solution:

```
National Enclave          CDS Equipment              NATO SECRET Domain
  (e.g., NOR)                                        (Shared)

+-------------+    +-----------------------------+    +-------------+
|             |    |  +-------+    +-----------+ |    |             |
| Source      +--->|  | Guard |    | Content   | +--->| Destination |
| System      |    |  | (HW)  +--->| Inspector | |    | System      |
|             |    |  +-------+    +-----------+ |    |             |
+-------------+    |                             |    +-------------+
                   |  +-------+    +-----------+ |
+-------------+    |  | Guard |    | Content   | |    +-------------+
|             |<---+  | (HW)  |<---+ Inspector | |<---+             |
| Destination |    |  +-------+    +-----------+ |    | Source      |
| System      |    |                             |    | System      |
+-------------+    |  Audit + Policy Engine      |    +-------------+
                   |  +-------------------------+|
                   |  | Transfer Log | Approval ||
                   |  +-------------------------+|
                   +-----------------------------+
```

#### CDS Product Selection

For NATO SECRET level cross-domain solutions, the following products are suitable:

- **Advenica SecuriCDS** (Swedish, widely used in Nordic defense): Hardware-enforced content inspection with configurable data filters. Strong fit given the Nordic context.
- **Thales Elips / Nexium Guard**: European-origin CDS used in NATO and EU environments.
- **SEALANT** (if available through NCIA): NATO-standardized cross-domain solution.

Each CDS implements:

1. **Hardware-enforced directionality**: Data can only flow in the configured direction at any given time (or uses separate hardware paths for each direction)
2. **Content inspection**: Deep content inspection against approved data formats (e.g., only XML, JSON, PDF in approved schemas; strip embedded objects, macros, executables)
3. **Data format validation**: Only pre-approved data types and schemas pass through
4. **Dirty word search**: Scanning for classification markings that indicate data above the receiving domain's clearance
5. **Malware scanning**: Multiple AV engines in sequence
6. **Approval workflow**: Human-in-the-loop approval for transfers (configurable per data type -- automated for routine, manual for novel)
7. **Complete audit trail**: Every transfer attempt (successful or rejected) is logged with timestamp, source, destination, file hash, approver, and disposition

### 8.3 Data Transfer Policies

| Transfer Direction | Policy | Approval |
|-------------------|--------|----------|
| National -> NATO SECRET | National release authority must approve. Data re-marked as NATO SECRET upon transfer. | National release officer + CDS operator |
| NATO SECRET -> National | Data marked for release to specific nation. Must comply with originator's release policy. | NATO release authority + receiving nation's CDS operator |
| National -> National | NOT PERMITTED directly. Must go National -> NATO (with release) -> NATO -> Other National (with release) | Two separate approvals |
| Any -> Lower classification | Downgrade/sanitization review required per originator's procedures | Originator + sanitization reviewer |

### 8.4 Data Diode Option

For scenarios where data flow must be strictly one-way (e.g., sensor data from a national system feeding into the NATO shared environment, with no possibility of reverse flow):

- **Hardware data diodes**: Owl Cyber Defense DualDiode, Advenica SecuriCDS Data Diode, Fox-IT DataDiode
- Data diodes provide hardware-enforced one-way transfer with no possibility of reverse channel
- Suitable for: log aggregation from national to NATO, sensor data ingest, intelligence feed distribution

---

## 9. Platform Services Architecture

### 9.1 Kubernetes Cluster Topology

Each security domain runs an independent Kubernetes cluster:

```
Per Domain Kubernetes Architecture:

  +--------------------------------------------------+
  |  Kubernetes Cluster (e.g., NATO SECRET)           |
  |                                                    |
  |  Control Plane (3 nodes, HA):                     |
  |  +--------+  +--------+  +--------+               |
  |  | CP-01  |  | CP-02  |  | CP-03  |               |
  |  | etcd   |  | etcd   |  | etcd   |               |
  |  | API    |  | API    |  | API    |               |
  |  | sched  |  | ctrl   |  |        |               |
  |  +--------+  +--------+  +--------+               |
  |                                                    |
  |  Worker Nodes:                                     |
  |  +--------+  +--------+  +--------+  +--------+   |
  |  | WRK-01 |  | WRK-02 |  | WRK-03 |  | WRK-04 |  |
  |  +--------+  +--------+  +--------+  +--------+   |
  |                                                    |
  |  Storage (Rook-Ceph):                              |
  |  Integrated on worker nodes (hyperconverged)       |
  |  or dedicated storage nodes                        |
  |                                                    |
  |  Networking: Cilium (eBPF-based CNI)               |
  |  Ingress: Ingress-NGINX or Envoy Gateway           |
  |  Service Mesh: Cilium (no sidecar overhead)        |
  |  Load Balancer: MetalLB (L2 or BGP mode)           |
  +--------------------------------------------------+
```

#### Cluster Configuration Per Domain

| Domain | Control Plane Nodes | Worker Nodes | CNI | Storage |
|--------|-------------------|--------------|-----|---------|
| NATO SECRET | 3 | 5-8 | Cilium | Rook-Ceph |
| NOR HEMMELIG | 3 | 3-4 | Cilium | Rook-Ceph |
| DNK HEMMELIGT | 3 | 3-4 | Cilium | Rook-Ceph |
| DEU GEHEIM | 3 | 3-4 | Cilium | Rook-Ceph |

### 9.2 Container Image Supply Chain

In an air-gapped classified environment, container image management is critical:

```
External (Unclassified)              Classified Environment

+------------------+               +------------------------+
| Public Registries|               | Air-gapped Registry    |
| (Docker Hub,     |  Sneakernet   | (Harbor)               |
|  Red Hat, GHCR)  +--- Media ---->|                        |
+------------------+  Verified     | Per-domain instance:   |
                      + Scanned    | - harbor.nato.secret   |
+------------------+               | - harbor.nor.hemmelig  |
| Vendor Registries|               | - harbor.dnk.hemmeligt |
| (Rancher, SUSE,  |               | - harbor.deu.geheim    |
|  Bitnami)        |               +------------------------+
+------------------+
```

**Image ingestion process:**

1. Images pulled on an unclassified network onto verified, encrypted media
2. Images scanned with Trivy/Grype for vulnerabilities on the unclassified side
3. SBOMs generated with Syft and reviewed
4. Images signed with cosign using the project's signing key
5. Media transferred through the facility's media ingestion process (sanitization, virus scan, approval)
6. Images loaded into each domain's Harbor registry
7. Kubernetes admission controllers (Kyverno or OPA Gatekeeper) enforce: only images from the domain's Harbor, only signed images, only scanned images with no critical CVEs

### 9.3 GitOps and Deployment

Each domain uses ArgoCD for GitOps-based deployment:

- **Git server**: Gitea (FLOSS) instance per domain, hosting infrastructure and application manifests
- **ArgoCD**: Per-domain ArgoCD instance watching the domain's Gitea
- **Workflow**: All changes go through Git. Merge requests require multi-person review (aligned with two-person integrity requirements at SECRET level).
- **Drift detection**: ArgoCD continuously reconciles desired state in Git with actual cluster state; any drift triggers an alert and auto-remediation

### 9.4 Monitoring and Observability

Each domain has an independent monitoring stack:

- **Metrics**: Prometheus + Thanos (for long-term retention within the domain)
- **Logs**: Loki (or ELK if full-text search is required)
- **Alerting**: Alertmanager with routing to the operations team
- **Dashboards**: Grafana
- **Audit logs**: All Kubernetes audit logs, OS-level auditd logs, and application logs shipped to the domain's log aggregation
- **Retention**: Classified audit logs retained per the PSI (typically 7-10 years for SECRET level)
- **SIEM integration**: Wazuh for security event correlation within each domain

**Cross-domain monitoring view**: The management domain can receive one-way, data-diode-fed health metrics from each classified domain (stripped of classified content -- only infrastructure health telemetry like CPU utilization, disk usage, node status). This provides a unified operational view without creating a data exfiltration path.

### 9.5 Secret Management

- **HashiCorp Vault** (or OpenBao, the FLOSS fork) per domain for application-level secrets
- Vault unsealing tied to the domain's HSM or TPM-sealed keys
- Kubernetes External Secrets Operator to sync Vault secrets into Kubernetes secrets
- Vault audit logging enabled; all secret access logged and attributable

---

## 10. Operational Model

### 10.1 Staffing and Clearances

| Role | Clearance Required | Nations | Count (Estimated) |
|------|-------------------|---------|-------------------|
| Infrastructure Lead | NATO SECRET + all national | NOR (host) | 1 |
| Platform Engineers | NATO SECRET | Mixed (NOR, DNK, DEU) | 4-6 |
| NOR Enclave Admin | NOR HEMMELIG | NOR | 1-2 |
| DNK Enclave Admin | DNK HEMMELIGT | DNK | 1-2 |
| DEU Enclave Admin | DEU GEHEIM | DEU | 1-2 |
| CDS Operators | NATO SECRET + CDS training | Mixed (dual-person) | 2-4 |
| Security Officer | NATO SECRET + national | NOR (host nation ISSO) | 1 |
| National Security Officers | National SECRET equiv. | 1 per nation | 3 |

**Operational constraint**: On-call rotations must account for clearance requirements. An uncleared person cannot respond to a classified system alert. This means 24/7 operations require either a larger cleared team or acceptance of delayed response during off-hours.

### 10.2 Change Management

All changes to the classified infrastructure follow a formal change management process:

1. **Change request** submitted in the domain's Gitea (as a merge request)
2. **Peer review** by a second cleared engineer (two-person integrity)
3. **Security review** by the ISSO for changes affecting security controls
4. **National authority notification** for changes to national enclaves (may require re-accreditation assessment)
5. **Approval** by the change advisory board (CAB) -- composed of representatives from all three nations for changes to the NATO shared domain
6. **Implementation** via GitOps (ArgoCD applies the merged change)
7. **Verification** via automated tests and manual validation
8. **Documentation** updated in the domain's configuration management database

### 10.3 Incident Response

- **Incident classification**: Security incidents are classified per NATO and national procedures simultaneously
- **Reporting chain**: Host nation NSM is the primary incident response coordinator; NCIA notified for NATO domain incidents; national NSAs notified for respective enclave incidents
- **Forensics**: Each domain maintains forensic imaging capability; evidence handling follows the most stringent national procedure among participants
- **Containment**: Domain isolation capability -- any domain can be network-isolated within minutes via physical switch disconnect (not just firewall rules)

### 10.4 Patching and Lifecycle

- **Patch cadence**: Monthly for routine patches; emergency patches within 72 hours for critical vulnerabilities
- **Patch source**: Disconnected repositories (Pulp) updated via sneakernet from unclassified sources
- **Patch testing**: Tested in a non-production namespace within each domain before production rollout
- **OS lifecycle**: RHEL 9 (support through 2032); Talos Linux follows Kubernetes release cadence
- **Kubernetes lifecycle**: Follow upstream support policy (N-2 minor versions); upgrade annually at minimum
- **Hardware lifecycle**: 5-year refresh cycle aligned with vendor warranty; firmware updates via verified media

---

## 11. Disaster Recovery and Business Continuity

### 11.1 Recovery Objectives

| Metric | NATO SECRET Domain | National Enclaves |
|--------|-------------------|-------------------|
| RTO (Recovery Time Objective) | 4 hours | 8 hours |
| RPO (Recovery Point Objective) | 1 hour | 4 hours |

### 11.2 DR Strategy

- **Primary**: Ceph replication (3x) within each domain provides resilience against individual disk and node failures
- **Backup**: Velero snapshots + Ceph RBD snapshots to backup storage pool within each domain
- **Offsite**: Encrypted backup media physically transported to a secondary LIST-X facility (location TBD -- possibly a secondary Norwegian facility or a NATO facility in another member state)
- **DR testing**: Quarterly DR drills per the PSI; each domain tested independently
- **Runbooks**: Automated runbooks in Gitea; tested during DR drills

### 11.3 High Availability Design

- All Kubernetes control planes are 3-node HA (etcd quorum requires 2 of 3)
- Ceph survives loss of any single OSD node (3x replication)
- Network fabric survives loss of any single spine or leaf switch (redundant paths)
- Power: UPS + generator; survives single power feed loss
- No single points of failure within any domain

---

## 12. Compliance Automation

### 12.1 Continuous Compliance Monitoring

Each domain implements automated compliance scanning aligned with its governing framework:

| Domain | Compliance Framework | Scanning Tool | Schedule |
|--------|---------------------|---------------|----------|
| NATO SECRET | C-M(2002)49 controls + NIST 800-53 (NATO alignment) | OpenSCAP + custom policies | Daily |
| NOR HEMMELIG | NSM ICT security baseline | OpenSCAP + NSM profiles | Daily |
| DNK HEMMELIGT | CFCS guidelines | OpenSCAP + CFCS profiles | Daily |
| DEU GEHEIM | BSI IT-Grundschutz | OpenSCAP + IT-Grundschutz profiles (note: IT-Grundschutz has thousands of controls; automated coverage is partial, manual assessment supplements) | Daily + quarterly manual |

### 12.2 Policy-as-Code

- **OPA/Gatekeeper or Kyverno** on each cluster enforcing:
  - Pod security standards (restricted profile)
  - No privileged containers
  - No host network/PID/IPC namespace sharing
  - Resource limits mandatory
  - Image provenance (signed + scanned)
  - No latest tags
  - Namespace-level network policies mandatory
- **Ansible compliance roles** for OS-level hardening verification
- **Git-tracked policy**: All compliance policies stored in Git, changes tracked and auditable

### 12.3 Audit Trail

- All Kubernetes API server audit logs captured (RequestResponse level for write operations, Metadata level for read operations)
- OS auditd capturing all privileged operations, file access to classified data, authentication events
- Network flow logs from Cilium Hubble
- CDS transfer logs (see section 8)
- All logs shipped to domain-local Loki/ELK with immutable storage (write-once, no delete capability for retention period)
- Log integrity: Logs cryptographically chained (hash chain) to detect tampering

---

## 13. Network Security Controls

### 13.1 Default-Deny Everywhere

- **Network level**: All inter-VLAN traffic denied by default; explicit allow rules per approved data flows
- **Kubernetes level**: Default-deny NetworkPolicy in every namespace; explicit allow policies per workload
- **Cilium eBPF**: L3/L4/L7 policy enforcement with identity-based policies (not just IP-based)
- **Host level**: iptables/nftables on every node with default-deny and explicit service rules

### 13.2 Internal Segmentation

Within each Kubernetes cluster, workloads are segmented by:

- **Namespaces**: One namespace per application or functional group
- **Network policies**: Per-namespace Cilium policies allowing only required communication
- **Service mesh**: Mutual TLS between all services (Cilium's built-in mTLS or Istio)
- **RBAC**: Per-namespace RBAC with least-privilege service accounts

### 13.3 Intrusion Detection

- **Falco**: Runtime security monitoring on all nodes (syscall-based behavioral detection)
- **Tetragon**: eBPF-based security observability (process execution, file access, network connections)
- **Suricata**: Network-based IDS on each domain's network fabric (monitoring SPAN/mirror ports)
- **Wazuh**: Host-based intrusion detection and compliance monitoring

---

## 14. Implementation Roadmap

### Phase 0: Agreements and Accreditation Planning (Months 1-3)

- Finalize bilateral/multilateral security agreements (GSA, PSI, CIS SEMU)
- Designate SAA structure and accreditation leads
- Complete threat and risk assessment
- Define Security Target documentation for each domain
- Procure CDS equipment (long lead time -- order early)
- Begin physical facility preparation

### Phase 1: Physical Infrastructure (Months 3-6)

- Complete facility zoning and TEMPEST preparation
- Install rack infrastructure with physical separation per domain
- Install power distribution and cooling per zone
- Deploy network fabric per domain (Cisco Nexus spine-leaf)
- Install and verify CDS hardware in XD zone
- Complete physical security accreditation with NSM

### Phase 2: Platform Deployment -- NATO SECRET Domain (Months 6-9)

- Deploy Talos Linux / RKE2 on NATO SECRET compute nodes
- Bootstrap Kubernetes cluster (3 CP + workers)
- Deploy Rook-Ceph storage
- Deploy Cilium CNI with default-deny policies
- Deploy Harbor registry, load initial container images
- Deploy core services: FreeIPA, Gitea, ArgoCD, Vault
- Deploy monitoring stack: Prometheus, Grafana, Loki, Alertmanager
- Deploy security tooling: Falco, Tetragon, Kyverno, OpenSCAP
- Begin NATO SECRET domain accreditation assessment

### Phase 3: National Enclave Deployment (Months 9-12)

- Deploy NOR, DNK, DEU enclaves in parallel (same stack as Phase 2, per-nation configuration)
- Deploy national identity providers (FreeIPA per enclave)
- Configure national cryptographic controls per enclave
- Deploy national monitoring and compliance tooling
- Begin national enclave accreditation assessments (in parallel with respective NSAs)

### Phase 4: Cross-Domain Integration (Months 12-14)

- Commission CDS equipment between NATO <-> each national enclave
- Configure and test data transfer policies
- Implement approval workflows
- Test data transfer scenarios (all direction combinations)
- Accreditation of CDS by joint SAA

### Phase 5: Accreditation and Initial Operations (Months 14-18)

- Complete all accreditation assessments
- Address findings from security assessors
- Obtain Authority to Operate (ATO) for each domain
- Transition from build team to operations team
- Conduct initial DR drill
- Begin operational workload deployment

### Phase 6: Continuous Operations (Month 18+)

- Continuous monitoring and compliance scanning
- Monthly patching cycles
- Quarterly DR drills
- Annual re-accreditation review
- Capacity planning and hardware refresh planning

---

## 15. Risk Register

| ID | Risk | Impact | Likelihood | Mitigation |
|----|------|--------|------------|------------|
| R1 | Accreditation delays from any single nation block entire project | High | Medium | Parallel accreditation tracks; early engagement with all NSAs; host nation NSM as coordinator |
| R2 | German IT-Grundschutz requirements exceed baseline architecture | Medium | High | Engage BSI early; design German enclave with IT-Grundschutz from day one; accept that German enclave may have additional controls |
| R3 | CDS procurement delays (long lead times for evaluated products) | High | Medium | Order CDS equipment in Phase 0; have backup vendor identified |
| R4 | TEMPEST requirements exceed budget | Medium | Medium | Early TEMPEST survey; design for Zone B minimum; Zone A only where assessment mandates |
| R5 | Cleared personnel shortage | High | Medium | Cross-train personnel; establish relationships with national cleared contractor pools; design for automation to reduce staffing needs |
| R6 | National crypto product incompatibilities | Medium | Low | Each domain uses its own crypto stack; no requirement for interoperability at crypto level (CDS handles domain transitions) |
| R7 | Supply chain compromise of hardware/software | High | Low | Verified procurement channels; firmware attestation; SBOM verification; tamper-evident shipping |
| R8 | Classification spillage between domains | Critical | Low | Physical separation; CDS with content inspection; comprehensive training; incident response procedures |

---

## 16. Architectural Decision Records (ADRs)

### ADR-001: Physical Separation Over Logical Separation

**Decision**: Each security domain uses physically separate network fabric, compute, and storage.

**Rationale**: At NATO SECRET level, VLANs and firewalls do not constitute sufficient separation. All participating national frameworks (NSM, CFCS, BSI) and NATO policy require physical isolation between security domains at the SECRET level. The marginal hardware cost is justified by the accreditation simplicity and reduced risk of classification spillage.

### ADR-002: Kubernetes + KubeVirt Over OpenStack

**Decision**: Use Kubernetes (Talos Linux / RKE2) with KubeVirt for VM workloads rather than OpenStack for IaaS.

**Rationale**: The defense community is standardizing on Kubernetes-based platforms (RKE2 Government, Platform One Big Bang, OpenShift for Government). Kubernetes provides a more declarative, GitOps-friendly operational model. KubeVirt enables legacy VM workloads without maintaining a separate hypervisor platform. OpenStack's operational complexity is not justified for this deployment scale (~24 servers).

### ADR-003: Cilium as CNI Over Calico

**Decision**: Use Cilium (eBPF-based) as the CNI plugin across all domains.

**Rationale**: Cilium provides L3/L4/L7 network policy enforcement, built-in mutual TLS (service mesh without sidecars), Hubble for network observability, and superior performance via eBPF. Tetragon (Cilium's runtime security component) provides kernel-level security monitoring. The eBPF-based approach reduces attack surface compared to iptables-based CNIs.

### ADR-004: Ceph Via Rook Over Dedicated SAN

**Decision**: Use Rook-Ceph for storage across all domains rather than dedicated SAN/NAS.

**Rationale**: Ceph provides unified block/object/file storage, is FLOSS (no licensing costs), integrates natively with Kubernetes via Rook, and is self-healing. Hyperconverged Ceph on compute nodes reduces rack footprint and hardware count, simplifying physical security accreditation. Dedicated SAN would add physical infrastructure requiring separate accreditation and management.

### ADR-005: No Direct National-to-National Cross-Domain Path

**Decision**: All data sharing between national enclaves must transit through the NATO SECRET shared domain.

**Rationale**: The NATO sharing model requires that multinational data be under NATO classification. Direct national-to-national transfers would require bilateral release authority assessment for each pair (3 pairs for 3 nations), tripling the CDS complexity. Routing through the NATO domain centralizes release authority and ensures all shared data is properly marked and tracked under NATO procedures.

### ADR-006: Harbor for Container Registry Over Quay

**Decision**: Use Harbor as the container registry in all domains.

**Rationale**: Harbor is CNCF graduated, supports image signing (cosign/Notary), vulnerability scanning (Trivy integration), replication policies, and RBAC. It has a lighter operational footprint than Quay in disconnected environments and does not require a Red Hat subscription.

---

## 17. Summary

This architecture provides a NATO SECRET-level multinational infrastructure with:

- **Four physically isolated security domains**: one shared NATO SECRET domain and three national enclaves (NOR, DNK, DEU)
- **Hardware-enforced cross-domain solutions** enabling controlled, auditable data sharing between domains
- **Per-domain Kubernetes clusters** on hardened, immutable operating systems (Talos Linux / RKE2) with Ceph storage, Cilium networking, and comprehensive security tooling
- **Independent cryptographic controls** per domain, using nationally-approved products
- **Continuous compliance automation** aligned with NATO, Norwegian (NSM), Danish (CFCS), and German (BSI IT-Grundschutz) frameworks
- **GitOps operational model** with full audit trails, two-person integrity, and change management
- **No shared infrastructure between domains** at the compute, storage, or network layer -- only CDS-mediated data transfers

The estimated timeline is 18 months from agreement signature to initial operational capability, with the critical path running through security agreements (Phase 0), CDS procurement (long lead time), and parallel accreditation by four authorities (NCIA, NSM, CFCS, BSI).
