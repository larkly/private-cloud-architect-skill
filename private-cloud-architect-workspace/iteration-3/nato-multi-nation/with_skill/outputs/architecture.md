# NATO SECRET Multinational Shared Infrastructure Architecture

## Participating Nations: Norway, Denmark, Germany

### Document Classification: UNCLASSIFIED — Architecture Reference Document

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope and Objectives](#2-scope-and-objectives)
3. [Applicable Frameworks and Authorities](#3-applicable-frameworks-and-authorities)
4. [Classification and Data Handling Model](#4-classification-and-data-handling-model)
5. [Physical Infrastructure and Facility Design](#5-physical-infrastructure-and-facility-design)
6. [Network Architecture](#6-network-architecture)
7. [Compute and Platform Architecture](#7-compute-and-platform-architecture)
8. [Storage Architecture](#8-storage-architecture)
9. [Identity, Access Management, and Personnel](#9-identity-access-management-and-personnel)
10. [Cross-Domain Data Sharing Architecture](#10-cross-domain-data-sharing-architecture)
11. [National Enclave Design](#11-national-enclave-design)
12. [Cryptographic Architecture](#12-cryptographic-architecture)
13. [TEMPEST and Emanation Security](#13-tempest-and-emanation-security)
14. [Monitoring, Audit, and Compliance](#14-monitoring-audit-and-compliance)
15. [Automation and Infrastructure as Code](#15-automation-and-infrastructure-as-code)
16. [Disaster Recovery and Business Continuity](#16-disaster-recovery-and-business-continuity)
17. [Accreditation Strategy](#17-accreditation-strategy)
18. [Supply Chain and Procurement](#18-supply-chain-and-procurement)
19. [Operational Model](#19-operational-model)
20. [Risk Register](#20-risk-register)
21. [Architecture Decision Records](#21-architecture-decision-records)

---

## 1. Executive Summary

This document defines the reference architecture for a NATO SECRET-level shared infrastructure platform hosted in a LIST-X equivalent facility in Norway. The platform supports a multinational defense project involving Norway, Denmark, and Germany, processing both NATO-classified and nationally-classified data with proper separation between national enclaves and a shared NATO SECRET processing domain.

The architecture addresses the fundamental challenge of multinational classified processing: each nation retains sovereign control over its own nationally-classified data while enabling collaborative work on NATO SECRET material through a shared domain. Data flows between national enclaves and the shared domain are mediated through formally accredited cross-domain solutions with content inspection and release authority gates.

Key architectural decisions:

- **Physically separated national enclaves** within a single LIST-X equivalent facility, each governed by the respective national security authority
- **A shared NATO SECRET processing domain** for collaborative multinational work, governed by NATO security policy
- **Cross-domain solutions (CDS)** mediating all data movement between national enclaves and the shared domain
- **National cryptographic sovereignty** maintained — each nation uses its own NSA-approved crypto for national traffic
- **Hardened Kubernetes (RKE2) with KubeVirt** as the converged compute platform, enabling both containerized and VM-based workloads
- **Full air-gap** from any unclassified network; all data ingress/egress through controlled transfer points with data diodes and CDS guards
- **Infrastructure as Code** using Ansible and OpenTofu, with offline execution pipelines

---

## 2. Scope and Objectives

### 2.1 In Scope

- NATO SECRET processing for the multinational defense project
- National SECRET-equivalent processing for each participating nation:
  - Norway: HEMMELIG
  - Denmark: HEMMELIGT
  - Germany: GEHEIM
- Cross-domain data sharing between national enclaves and the NATO SECRET shared domain
- Physical hosting in a Norwegian LIST-X equivalent facility
- Compute, storage, networking, identity management, and security services
- Accreditation coordination across three national authorities and NATO

### 2.2 Out of Scope

- COSMIC TOP SECRET / national TOP SECRET processing
- Unclassified or RESTRICTED-only processing (separate infrastructure)
- Public cloud integration (air-gapped environment)
- Application-level design (this document covers infrastructure only)

### 2.3 Objectives

1. Enable three nations to collaboratively process NATO SECRET data on a shared platform
2. Maintain strict separation between nationally-classified data belonging to each nation
3. Comply with all applicable national security frameworks and NATO security policy
4. Provide controlled, auditable, accreditable cross-domain data sharing
5. Achieve 99.99% availability for the shared NATO SECRET domain
6. Minimize operational complexity through automation while maintaining security posture
7. Design for accreditation from day one — continuous compliance, not bolt-on

---

## 3. Applicable Frameworks and Authorities

### 3.1 NATO

| Document | Applicability |
|---|---|
| C-M(2002)49 — Security within NATO | Primary governing policy for all NATO SECRET data handling |
| AC/35 series — NATO Information Security | Technical security requirements for NATO CIS |
| SDIP-27 (NATO TEMPEST standard) | Emanation security zones for all equipment |
| NCIA accreditation requirements | System accreditation for NATO SECRET processing |

### 3.2 Norway (Host Nation)

| Authority / Framework | Role |
|---|---|
| NSM (Nasjonal Sikkerhetsmyndighet) | National Security Authority — accredits the facility and Norwegian national enclave |
| Sikkerhetsloven (2018) | Legal basis for protective security measures |
| Virksomhetsikkerhetsforskriften | Regulations on enterprise security (physical, personnel, information) |
| Klareringsforskriften | Security clearance regulations |
| Sikkerhetsgraderte anskaffelser | Regulations on classified procurement |
| NSM Grunnprinsipper for IKT-sikkerhet | Primary ICT security baseline (identify, protect, detect, respond) |
| NSM crypto approval list | Mandatory for all classified network encryption |

As the host nation, Norway bears additional responsibility under Sikkerhetsloven for protecting foreign classified information (NATO, Danish, German) handled within Norwegian territory. NSM must approve the facility for handling foreign classified material, and bilateral/multilateral security agreements must be in place.

### 3.3 Denmark

| Authority / Framework | Role |
|---|---|
| CFCS (Center for Cybersikkerhed) | National security authority for cyber — accredits Danish national enclave |
| Danish Security Act (Sikkerhedsloven) | Legal basis for classified information handling |
| CFCS technical guidelines | Security baseline for Danish classified systems |

Denmark must approve the handling of Danish HEMMELIGT material in the Norwegian facility. This requires a bilateral security agreement between Norway and Denmark (likely already exists under NATO frameworks, but must be explicitly verified for this specific use case) and CFCS approval of the Danish enclave's security controls.

### 3.4 Germany

| Authority / Framework | Role |
|---|---|
| BSI (Bundesamt fur Sicherheit in der Informationstechnik) | Federal Office for Information Security — technical security standards |
| BMVg (Bundesministerium der Verteidigung) | Ministry of Defence — for military classified systems |
| BSI IT-Grundschutz | Primary hardening and control baseline (extremely thorough) |
| BSI C5 | Cloud security criteria catalogue |
| VS-NfD / GEHEIM handling regulations | Specific handling requirements for German classified data |
| German Federal Security Clearance Check Act | Personnel clearance framework |

Germany's GEHEIM data processed outside German territory requires explicit approval from the relevant German authority (BMVg for defense projects). BSI IT-Grundschutz Bausteine will be the primary hardening reference for the German enclave — this is non-negotiable from the German perspective and is more prescriptive than NSM Grunnprinsipper or CFCS guidance.

### 3.5 International Context

| Framework | Applicability |
|---|---|
| GDPR / EU data protection | Applies to any personal data processed — all three nations are EU/EEA members |
| NIS2 Directive | Critical infrastructure security obligations (Norway via EEA, Denmark and Germany as EU members) |
| Bilateral security agreements (NO-DK, NO-DE, DK-DE) | Govern handling of each other's nationally-classified information |
| NATO SOFA / technical agreements | May govern personnel and operational arrangements |

---

## 4. Classification and Data Handling Model

### 4.1 Classification Domains

The infrastructure operates four distinct classification domains:

```
+-------------------------------------------------------------------+
|                     LIST-X Equivalent Facility (Norway)            |
|                                                                     |
|  +-------------------+  +-------------------+  +------------------+ |
|  | Norwegian Enclave  |  | Danish Enclave    |  | German Enclave   | |
|  | HEMMELIG           |  | HEMMELIGT         |  | GEHEIM           | |
|  | + NATO SECRET      |  | + NATO SECRET     |  | + NATO SECRET    | |
|  | NSM accredited     |  | CFCS accredited   |  | BSI/BMVg accred. | |
|  +--------+----------+  +--------+----------+  +--------+---------+ |
|           |                       |                       |          |
|           v                       v                       v          |
|  +-------------------------------------------------------------------+
|  |              Cross-Domain Solution (CDS) Layer                    |
|  |     Content inspection, release authority, audit logging          |
|  +-------------------------------------------------------------------+
|           |                       |                       |          |
|           v                       v                       v          |
|  +-------------------------------------------------------------------+
|  |           Shared NATO SECRET Processing Domain                    |
|  |           NCIA accredited, NATO security policy                   |
|  |           Multinational collaborative workspace                   |
|  +-------------------------------------------------------------------+
+-------------------------------------------------------------------+
```

### 4.2 Data Classification Categories

| Domain | Data Types | Governing Authority | Releasability |
|---|---|---|---|
| Norwegian Enclave | Norwegian HEMMELIG, Norwegian-originated NATO SECRET | NSM | NOR eyes only unless explicitly released |
| Danish Enclave | Danish HEMMELIGT, Danish-originated NATO SECRET | CFCS | DNK eyes only unless explicitly released |
| German Enclave | German GEHEIM, German-originated NATO SECRET | BSI/BMVg | DEU eyes only unless explicitly released |
| Shared NATO Domain | NATO SECRET releasable to NO/DK/DE | NCIA (with national delegations) | NO/DK/DE per marking |

### 4.3 Data Flow Rules

**Fundamental principle**: National data does not leave the national enclave unless explicitly released by the originating nation's release authority. NATO SECRET data produced in the shared domain is governed by NATO release policy.

1. **National enclave to shared domain**: Data may only move from a national enclave to the shared NATO domain after formal release by the originating nation's designated release authority. The CDS enforces marking verification — only data marked as releasable to the other participating nations passes through.

2. **Shared domain to national enclave**: NATO SECRET data from the shared domain may flow to a national enclave for national processing, subject to NATO release markings and any national caveats. The CDS verifies that the receiving nation is authorized per the data's releasability markings.

3. **Between national enclaves**: Direct data flow between national enclaves is **not permitted**. All cross-national data sharing goes through the shared NATO SECRET domain. This prevents bilateral data handling complications and ensures all cross-national flows are governed by NATO policy.

4. **Data ingress (from external)**: External classified data enters through controlled transfer points — physical media inspection and data diode ingress points. Each nation has its own dedicated ingress point for its national enclave.

5. **Data egress (to external)**: Classified data export follows the originating nation's downgrading and release procedures. Data must pass through a sanitization review before export. Each nation controls egress from its own enclave.

---

## 5. Physical Infrastructure and Facility Design

### 5.1 LIST-X Equivalent Facility

The facility in Norway must be accredited as a LIST-X equivalent (Norwegian: sikkerhetsgradert objekt) under Sikkerhetsloven and Virksomhetsikkerhetsforskriften, approved by NSM for handling:

- Norwegian HEMMELIG
- Danish HEMMELIGT (under bilateral agreement)
- German GEHEIM (under bilateral agreement)
- NATO SECRET

**Facility requirements**:

- **Perimeter security**: Fenced compound with intrusion detection, CCTV, guard force, vehicle barriers. Minimum two-person integrity for access to classified areas.
- **Access control zones**: Progressively restricted zones — public, controlled, restricted, classified processing areas. Each classification domain occupies a physically separated area within the classified processing zone.
- **Alarm systems**: IDS (intrusion detection system) reporting to a 24/7 security operations center. Tamper detection on all entry points to classified areas.
- **Visitor control**: Escorted access only for uncleared visitors. Foreign nationals require advance approval per bilateral agreements.
- **TEMPEST zoning**: SDIP-27 Zone B minimum for SECRET-level processing areas, with Zone A for any areas processing national TOP SECRET equivalent (out of scope but plan for adjacency).

### 5.2 Physical Separation of Domains

Each domain occupies physically distinct infrastructure:

```
Floor Plan (Conceptual)
+---------------------------------------------------------------+
|  Facility Perimeter                                            |
|  +-----------------------------------------------------------+|
|  | Controlled Zone                                            ||
|  |  +----------+  +----------+  +----------+  +------------+ ||
|  |  | NOR      |  | DNK      |  | DEU      |  | Shared     | ||
|  |  | Enclave  |  | Enclave  |  | Enclave  |  | NATO SEC   | ||
|  |  | Room(s)  |  | Room(s)  |  | Room(s)  |  | Room(s)    | ||
|  |  +----+-----+  +----+-----+  +----+-----+  +-----+------+ ||
|  |       |              |              |               |       ||
|  |  +----+--------------+--------------+---------------+-----+||
|  |  | CDS Room — Cross-Domain Solutions, Data Diodes         |||
|  |  | (Physically secured, dual-person access control)       |||
|  |  +--------------------------------------------------------+||
|  |                                                            ||
|  |  +--------------------------------------------------------+||
|  |  | Shared Services: Power, Cooling, Physical Security Ops  |||
|  |  +--------------------------------------------------------+||
|  +-----------------------------------------------------------+|
+---------------------------------------------------------------+
```

**Physical separation requirements**:

- **Separate racks**: Each domain has dedicated racks. No domain shares a rack with another.
- **Separate cabling**: Red (classified) cables for each domain are physically separated — different cable trays, conduits, or at minimum maintained separation distances per SDIP-27. No mixing of cabling between domains.
- **Separate network equipment**: Each domain has its own switches, routers, and firewalls. No shared network equipment between domains except at the CDS boundary (which is purpose-built for this).
- **Access control per room**: Each enclave room has its own access control tied to the appropriate national clearance. The Norwegian enclave requires Norwegian HEMMELIG clearance. The Danish enclave requires Danish HEMMELIGT clearance or NATO SECRET clearance with Danish approval. The German enclave requires German GEHEIM Sicherheitsuberprufung. The shared domain requires NATO SECRET clearance.
- **Separate power distribution**: Each domain should have dedicated PDUs (power distribution units) from the shared UPS/generator infrastructure. Power is not a signal path concern at SECRET level in most cases, but dedicated PDUs simplify management and physical access control.

### 5.3 Environmental Infrastructure

- **Power**: Redundant utility feeds (A+B), diesel generator with minimum 72-hour fuel supply, UPS (N+1 redundancy per domain)
- **Cooling**: N+1 redundant cooling per server room, monitoring of temperature and humidity per rack
- **Fire suppression**: Clean agent (FM-200 or Novec) fire suppression in all server rooms — water-based systems risk equipment damage and evidence destruction
- **Environmental monitoring**: Temperature, humidity, water leak detection, smoke detection — all feeding into a centralized BMS (building management system) that is itself on a separate, unclassified management network

---

## 6. Network Architecture

### 6.1 Network Design Principles

1. **Physical isolation between domains** — no VLANs as security boundaries between classification domains. Separate physical switches, separate physical cabling.
2. **Spine-leaf within each domain** — each domain gets its own spine-leaf fabric for scalability and predictable latency.
3. **No routing between domains** — domain interconnection is exclusively through the CDS layer, which is not a router. It is a content-inspecting guard with explicit release policies.
4. **Defense in depth within each domain** — microsegmentation, network policies, host-based firewalls, and encrypted inter-node communication even within a single domain.
5. **Management network isolation** — out-of-band management (IPMI/iDRAC/iLO) on a dedicated, physically separate management network per domain. Management networks never cross domain boundaries.

### 6.2 Network Topology per Domain

Each domain (three national enclaves + shared NATO domain = four domains) has an identical network topology pattern:

```
                    +------------------+
                    |   Domain Border  |
                    |   Firewall (HA)  |
                    +--------+---------+
                             |
              +--------------+--------------+
              |                             |
        +-----+------+              +------+-----+
        |  Spine SW 1 |              | Spine SW 2 |
        +-----+------+              +------+-----+
              |    \                /       |
              |     \              /        |
              |      \            /         |
        +-----+------+ +--------+-----+
        |  Leaf SW 1  | |  Leaf SW 2   | ... (N leaf pairs)
        +------+------+ +------+-------+
               |                |
          +----+----+     +----+----+
          | Server  |     | Server  |  ... (compute nodes)
          | Rack 1  |     | Rack 2  |
          +---------+     +---------+
```

**Hardware selection**:

- **Spine switches**: Recommend Cisco Nexus 9300-series or equivalent — well-understood in defense environments, strong VXLAN-EVPN support, Ansible-automatable via cisco.nxos collection. Alternative: Arista 7050X series if Cisco is not mandated.
- **Leaf switches**: Cisco Nexus 93180YC-FX3 or equivalent — 48x 1/10/25G server ports + 6x 40/100G uplinks. One leaf pair per rack or per two racks depending on density.
- **Domain border firewall**: Palo Alto PA-series or Cisco Firepower — stateful inspection, application-level visibility, IPS capability. HA pair per domain. This firewall sits between the domain's internal fabric and the CDS interconnection points. It does NOT connect to the internet or any unclassified network.
- **Management switches**: Separate physical switches for IPMI/BMC traffic, completely isolated from production network.

### 6.3 Inter-Domain Network (CDS Interconnection)

The CDS layer is the only point where traffic can cross domain boundaries. The network design at this boundary:

```
+-------------+     +--------+     +------------------+     +--------+     +---------------+
| NOR Enclave |---->| NOR    |---->|                  |---->| Shared |---->| Shared NATO   |
| Firewall    |     | CDS    |     | CDS Inspection   |     | CDS    |     | Domain FW     |
|             |<----| Guard  |<----| & Release Engine  |<----| Guard  |<----|               |
+-------------+     +--------+     +------------------+     +--------+     +---------------+
```

- Each national enclave connects to the CDS via a dedicated physical interface on the enclave's border firewall
- The CDS equipment sits in its own physically secured room (dual-person access)
- CDS guards perform content inspection, marking verification, and release authority checks
- All CDS transactions are logged with full audit trails, retained for the duration mandated by the strictest applicable national requirement

### 6.4 IP Addressing and DNS

- Each domain has its own independent IP address space (RFC 1918 or NATO-assigned addressing per NCIA guidance)
- No IP address overlap concerns since domains do not route to each other
- Each domain runs its own DNS infrastructure (CoreDNS or BIND on hardened hosts)
- No DNS resolution across domain boundaries — the CDS operates at the application/content level, not at the IP routing level

### 6.5 Network Monitoring

- Each domain runs its own network monitoring stack (covered in Section 14)
- SNMP v3 only (encrypted, authenticated) or gNMI/gRPC telemetry where supported
- Network flow data (NetFlow/sFlow) collected and retained per domain for security forensics
- No network monitoring data crosses domain boundaries

---

## 7. Compute and Platform Architecture

### 7.1 Platform Selection

**Decision: RKE2 (Rancher Kubernetes Engine 2) with KubeVirt**

Rationale:
- **RKE2** is designed for government and security-sensitive environments. It ships with CIS hardening by default, supports FIPS 140-2 cryptographic modules (relevant for NATO interoperability), and uses containerd as its runtime (not Docker). It is the successor to RKE in SUSE's Kubernetes portfolio.
- **KubeVirt** enables running traditional VM workloads alongside containers within Kubernetes, providing a converged platform. This is critical because defense workloads often include legacy applications that cannot be containerized, as well as modern cloud-native services.
- This combination avoids VMware licensing costs and Broadcom acquisition uncertainty while providing a FLOSS-based platform with commercial support available from SUSE/Rancher.
- Alternative considered: OpenShift (Common Criteria certified, strong government track record) — viable but higher licensing cost and more opinionated. If a participating nation mandates OpenShift, it can be deployed within that nation's enclave while RKE2 is used elsewhere.

### 7.2 Cluster Architecture

Each domain runs its own independent Kubernetes cluster. No cluster federation or cross-domain cluster communication.

**Per-domain cluster design**:

```
Control Plane (3 nodes, HA):
  +------------+  +------------+  +------------+
  | CP Node 1  |  | CP Node 2  |  | CP Node 3  |
  | etcd       |  | etcd       |  | etcd       |
  | API server |  | API server |  | API server |
  | Scheduler  |  | Scheduler  |  | Controller |
  | Controller |  | Controller |  | Manager    |
  +------------+  +------------+  +------------+

Worker Nodes (N nodes, scaled per domain workload):
  +------------+  +------------+  +------------+  +---
  | Worker 1   |  | Worker 2   |  | Worker 3   |  | ...
  | Kubelet    |  | Kubelet    |  | Kubelet    |
  | containerd |  | containerd |  | containerd |
  | KubeVirt   |  | KubeVirt   |  | KubeVirt   |
  +------------+  +------------+  +------------+  +---

Infrastructure Nodes (2-3 nodes for platform services):
  +------------------+  +------------------+
  | Infra Node 1     |  | Infra Node 2     |
  | Harbor registry  |  | Harbor registry  |
  | Monitoring stack |  | Monitoring stack |
  | Logging stack    |  | Logging stack    |
  +------------------+  +------------------+
```

**Sizing guidance (per domain)**:

| Component | Minimum | Recommended | Notes |
|---|---|---|---|
| Control plane nodes | 3 | 3 | Dedicated, not schedulable for workloads |
| Worker nodes | 6 | 12+ | Scale based on workload demand |
| Infrastructure nodes | 2 | 3 | Harbor, monitoring, logging, Vault |
| CPU per worker | 64 cores | 128 cores | Xeon Scalable or AMD EPYC |
| RAM per worker | 256 GB | 512 GB | KubeVirt VMs need dedicated RAM |
| Local storage per worker | 2x NVMe 1.6TB | 4x NVMe 3.2TB | For Ceph OSD or local ephemeral |

The shared NATO SECRET domain will be the largest, as it hosts the collaborative workloads. National enclaves may be smaller, depending on the volume of national processing.

### 7.3 Server Hardware

**Recommended platform**: Dell PowerEdge R760 or HPE ProLiant DL380 Gen11 — both widely available through NATO-approved supply chains and well-supported in European defense procurement.

Alternative: Cisco UCS C240 M7 if the organization has existing Cisco UCS investment and Intersight management capability.

**Hardware standardization**: All domains use identical hardware models to simplify sparing, maintenance, and supply chain management. National enclave hardware may be procured through national defense procurement channels if required by national regulations (Germany's GEHEIM procurement rules, for instance, may require BSI-approved supply chain).

### 7.4 Operating System

**Decision: Red Hat Enterprise Linux (RHEL) 9 or SUSE Linux Enterprise Server (SLES) 15 SP5+**

- Both have Common Criteria certification and established presence in European defense
- RHEL aligns with the broader RKE2/Rancher ecosystem (RKE2 officially supports RHEL and SLES)
- Hardened per the respective national baseline:
  - Norwegian enclave: NSM Grunnprinsipper for IKT-sikkerhet
  - Danish enclave: CFCS technical guidelines
  - German enclave: BSI IT-Grundschutz Bausteine (SYS.1.3 Linux Server, APP.6 Container)
  - Shared NATO domain: NATO security requirements + host nation (NSM) baseline as minimum

**Hardening approach**: OpenSCAP with profiles aligned to each national baseline. Automated scanning on build and continuously in production. Golden images built via Packer with hardening baked in.

### 7.5 Container Runtime and Registry

- **Runtime**: containerd (bundled with RKE2)
- **Registry**: Harbor (self-hosted, per domain) — provides vulnerability scanning (Trivy integration), image signing verification (cosign/Notary), RBAC, and audit logging
- **Image provenance**: All container images must be built from known-good base images, scanned for vulnerabilities, signed with the domain's PKI, and have SBOMs generated (Syft). Only signed images are admitted to clusters (enforced by Kyverno admission controller).
- **Offline image pipeline**: Images are built on a designated build system, scanned, signed, and transferred into the air-gapped environment via the controlled transfer point. Within the air-gapped environment, Harbor serves as the single source of truth.

---

## 8. Storage Architecture

### 8.1 Software-Defined Storage: Ceph

**Decision: Ceph deployed via Rook (Rook-Ceph operator within each domain's Kubernetes cluster)**

Rationale:
- Ceph provides unified block (RBD), file (CephFS), and object (RGW) storage
- Rook-Ceph integrates natively with Kubernetes for lifecycle management
- No proprietary SAN/NAS licensing costs
- Battle-tested at scale in government and defense environments
- Each domain runs its own independent Ceph cluster — no storage sharing between domains

**Ceph cluster design (per domain)**:

| Component | Configuration |
|---|---|
| Ceph monitors (MON) | 3 (co-located on control plane or dedicated storage nodes) |
| Ceph managers (MGR) | 2 (active/standby) |
| Ceph OSDs | Minimum 3 nodes, 4+ NVMe SSDs per node |
| Replication | 3x replication for all pools (data durability) |
| Encryption | dm-crypt on all OSD devices (encryption at rest) |
| Network | Dedicated storage VLAN/network (separate from pod network) |

### 8.2 Storage Classes

| Storage Class | Backend | Use Case |
|---|---|---|
| ceph-block-ssd | Ceph RBD (NVMe pool) | Database volumes, high-IOPS workloads, KubeVirt VM disks |
| ceph-filesystem | CephFS | Shared file access (ReadWriteMany), collaborative data stores |
| ceph-object | Ceph RGW | S3-compatible object storage for applications |
| local-nvme | TopoLVM | Ephemeral high-performance local storage (non-replicated) |

### 8.3 Data at Rest Encryption

All storage is encrypted at rest:
- **Ceph OSDs**: dm-crypt/LUKS encryption on every OSD device, keys managed by the domain's key management system
- **etcd**: Kubernetes etcd encryption enabled with AES-CBC or AES-GCM using keys from the domain's Vault instance
- **KubeVirt VM disks**: Inherit encryption from the underlying Ceph RBD volume

Encryption algorithms must be approved by the relevant authority for each domain:
- Norwegian/Danish/Shared NATO: NSM-approved or NATO-approved algorithms
- German: BSI-approved algorithms (BSI TR-02102 series)

### 8.4 Backup

- **Velero** for Kubernetes resource and persistent volume backup
- Ceph RBD snapshots for point-in-time recovery of block volumes
- Backup data stays within the domain — no cross-domain backup
- Backup media (if exported to physical media for offsite storage) follows the classification handling procedures for that domain
- RTO/RPO targets: RPO 1 hour, RTO 4 hours for critical workloads; RPO 24 hours, RTO 24 hours for non-critical

---

## 9. Identity, Access Management, and Personnel

### 9.1 Identity Architecture

Each domain operates its own identity provider. No identity federation across domain boundaries (a user in the Norwegian enclave authenticates to the Norwegian IdP; when they work in the shared NATO domain, they authenticate separately to the NATO domain IdP).

**Per-domain identity stack**:

| Component | Technology | Purpose |
|---|---|---|
| Directory | FreeIPA or Active Directory | User accounts, groups, machine accounts |
| SSO / OAuth2 / OIDC | Keycloak | Application-level SSO, token issuance |
| Kubernetes RBAC | Integrated with OIDC (Keycloak) | Cluster access control |
| PKI | FreeIPA CA or EJBCA | TLS certificates, client authentication |
| MFA | Hardware tokens (YubiKey or national equivalent) | Second factor for all administrative access |

**Preference**: FreeIPA for the Norwegian, Danish, and shared NATO domains (FLOSS, well-supported, Kerberos + LDAP + PKI integrated). The German enclave may require Active Directory if BSI IT-Grundschutz SYS.2.2.3 (Windows client hardening) mandates it for their workstation environment.

### 9.2 Access Control Model

**RBAC + ABAC hybrid**:

- Role-Based Access Control (RBAC) for coarse-grained access: cluster-admin, namespace-admin, developer, viewer
- Attribute-Based Access Control (ABAC) for fine-grained data access: nationality, clearance level, project assignment, need-to-know compartment

**Kubernetes RBAC design**:

```yaml
# Example: Norwegian enclave namespace structure
namespaces:
  - nor-project-alpha          # Norwegian national project work
  - nor-project-beta
  - nor-shared-staging         # Staging area for data release to shared domain
  - nor-system                 # Platform services (monitoring, logging)

# Shared NATO domain namespace structure
namespaces:
  - nato-project-main          # Primary multinational collaborative workspace
  - nato-project-analytics
  - nato-shared-ingest         # Receives data from national enclaves via CDS
  - nato-system
```

### 9.3 Personnel and Clearance Requirements

| Role | Required Clearance | Domain Access |
|---|---|---|
| Norwegian system administrators | Norwegian HEMMELIG + NATO SECRET | Norwegian enclave + Shared NATO domain |
| Danish system administrators | Danish HEMMELIGT + NATO SECRET | Danish enclave + Shared NATO domain |
| German system administrators | German GEHEIM (Ue3) + NATO SECRET | German enclave + Shared NATO domain |
| Shared platform operators | NATO SECRET (any participating nation) | Shared NATO domain only |
| CDS operators | NATO SECRET + host nation approval | CDS room |
| Facility security officers | Norwegian HEMMELIG | Facility-wide physical security |

**Critical constraint**: Personnel from one nation cannot administer another nation's enclave unless explicitly authorized by that nation's security authority. A Norwegian administrator does not have access to the German enclave, even if they hold NATO SECRET clearance, because German GEHEIM requires German-granted clearance or explicit bilateral authorization.

### 9.4 Privileged Access Management

- All administrative access via a Privileged Access Workstation (PAW) within each domain
- Session recording for all administrative sessions (terminal session logging)
- Break-glass procedures for emergency access with post-incident review
- No shared/generic accounts — every action attributable to an individual
- Administrative access requires MFA (hardware token)
- Principle of least privilege enforced: operators get only the permissions needed for their role, scoped to specific namespaces

---

## 10. Cross-Domain Data Sharing Architecture

This is the most architecturally critical component of the entire system. The CDS layer is what makes multinational collaboration possible while maintaining national sovereignty over classified data.

### 10.1 Cross-Domain Solution Design

**Decision: Hardware-enforced CDS guards with content inspection**

The CDS layer consists of:

1. **Data diodes** (hardware one-way): For use cases requiring strictly unidirectional flow (e.g., sensor data ingress from external sources)
2. **CDS guards** (bidirectional, content-inspecting): For managed bidirectional data sharing between national enclaves and the shared NATO domain

**Recommended CDS products** (subject to national authority approval):

- Advenica SecuriCDS (Swedish — strong Nordic defense presence, NATO-approved variants)
- Forcepoint Trusted Gateway System
- Thales Elips (if nationally approved)
- Each participating nation's security authority must approve the specific CDS product for handling their national classified data

### 10.2 CDS Architecture

```
                     National Enclaves                    Shared NATO Domain

  +----------+       +------------------+                 +------------------+
  | NOR      |<----->| CDS Guard        |<--------------->| NATO Shared      |
  | Enclave  |       | (NOR <-> NATO)   |                 | Domain           |
  +----------+       | - Content filter |                 |                  |
                     | - Marking verify |                 |                  |
  +----------+       | - Release auth   |                 |                  |
  | DNK      |<----->| CDS Guard        |<--------------->|                  |
  | Enclave  |       | (DNK <-> NATO)   |                 |                  |
  +----------+       +------------------+                 |                  |
                                                          |                  |
  +----------+       +------------------+                 |                  |
  | DEU      |<----->| CDS Guard        |<--------------->|                  |
  | Enclave  |       | (DEU <-> NATO)   |                 |                  |
  +----------+       +------------------+                 +------------------+
```

**Three separate CDS guard instances** — one per national enclave to shared domain boundary. This ensures:
- National-specific content inspection rules per guard
- National release authority integration per guard
- Independent audit trails per national boundary crossing
- Failure isolation — a CDS failure on one boundary does not affect the others

### 10.3 Content Inspection and Release Policy

Each CDS guard enforces:

1. **Classification marking verification**: Every data object passing through the CDS must carry a valid classification marking. The CDS verifies the marking format and checks it against the transfer policy.

2. **Releasability check**: Data marked "NOR EYES ONLY" is blocked from transfer to the shared domain. Only data explicitly marked as releasable to the participating nations (or NATO-releasable) passes through.

3. **Content type inspection**: The CDS inspects content types and blocks executable code, scripts, and other potentially dangerous payloads unless explicitly whitelisted for the specific data flow.

4. **Malware scanning**: All data passing through the CDS is scanned for malware (recognizing that signatures may be dated in an air-gapped environment — behavioral analysis is preferred).

5. **Size and rate limits**: Data transfers are rate-limited and size-capped to prevent data exfiltration attacks.

6. **Release authority approval**: For sensitive transfers, the CDS can require explicit human approval from the designated release authority before the data passes through. This can be implemented as:
   - Automatic release for pre-approved data types and markings
   - Manual approval queue for transfers that match escalation criteria
   - Dual-person approval for high-sensitivity transfers

### 10.4 Data Transfer Workflow

**Example: Norwegian analyst releases a dataset to the shared NATO domain**

```
1. Analyst prepares dataset in Norwegian enclave
2. Analyst applies NATO SECRET REL NO/DK/DE marking
3. Analyst submits dataset to the NOR release staging namespace
4. Norwegian release authority reviews and approves
5. Dataset is pushed to the CDS guard (NOR <-> NATO)
6. CDS guard verifies:
   a. Classification marking present and valid
   b. Releasability includes DK and DE (matches shared domain policy)
   c. Content type is allowed (e.g., structured data, PDF, not executable)
   d. Malware scan passes
   e. Transfer is logged with full metadata
7. Dataset appears in the NATO shared domain ingest namespace
8. Shared domain data pipeline processes and makes available to authorized users
```

### 10.5 API-Level Integration

For automated data sharing (not just file transfer), the CDS can mediate API calls:

- **REST API proxying**: The CDS can proxy specific, pre-approved API endpoints between domains. The CDS acts as a protocol break — it terminates the connection on one side, inspects the content, and initiates a new connection on the other side.
- **Message queue bridging**: For asynchronous data sharing, a message queue (e.g., Apache Kafka or RabbitMQ) runs on each side of the CDS, with the CDS mediating message transfer between queues.
- **Database replication**: NOT recommended across domain boundaries. Instead, use API-level or message-level data sharing with explicit content inspection.

---

## 11. National Enclave Design

### 11.1 Common Enclave Pattern

Each national enclave follows the same architectural pattern but is independently operated and accredited:

```
+------------------------------------------------------------------+
| National Enclave (NOR / DNK / DEU)                                |
|                                                                    |
|  +------------------+  +------------------+  +------------------+  |
|  | Kubernetes       |  | Storage          |  | Identity         |  |
|  | Cluster (RKE2)   |  | Cluster (Ceph)   |  | (FreeIPA/AD)     |  |
|  | + KubeVirt       |  | via Rook-Ceph    |  | + Keycloak       |  |
|  +------------------+  +------------------+  +------------------+  |
|                                                                    |
|  +------------------+  +------------------+  +------------------+  |
|  | Monitoring       |  | Logging          |  | Secrets          |  |
|  | Prometheus       |  | Loki             |  | HashiCorp Vault  |  |
|  | Grafana          |  | (+ Grafana)      |  | (or Sealed Secs) |  |
|  +------------------+  +------------------+  +------------------+  |
|                                                                    |
|  +------------------+  +------------------+                        |
|  | Harbor Registry  |  | Automation       |                        |
|  | (images, Helm)   |  | AWX / Ansible    |                        |
|  +------------------+  +------------------+                        |
|                                                                    |
|  +--------------------------------------------------------------+  |
|  | CDS Interface (to/from Shared NATO Domain)                    |  |
|  +--------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

### 11.2 Norwegian Enclave Specifics

- **Accreditation authority**: NSM
- **Primary hardening baseline**: NSM Grunnprinsipper for IKT-sikkerhet
- **Crypto**: NSM-approved cryptographic products for any encrypted communication within the enclave
- **Personnel**: Norwegian HEMMELIG clearance (or NATO SECRET with NSM approval for non-Norwegian personnel working on joint tasks)
- **Additional requirements**: NSM may mandate specific logging formats, audit trail retention periods (typically 5-10 years for HEMMELIG), and incident reporting procedures
- **Network monitoring**: Must comply with NSM's detection requirements (category: detect in Grunnprinsipper)

### 11.3 Danish Enclave Specifics

- **Accreditation authority**: CFCS
- **Primary hardening baseline**: CFCS technical security guidelines
- **Crypto**: Danish/NATO-approved cryptographic products
- **Personnel**: Danish HEMMELIGT clearance
- **Additional requirements**: CFCS threat assessment integration — CFCS publishes regular threat assessments that may influence security control tuning. Danish operational requirements may include specific network sensor deployments per CFCS guidance.
- **Key consideration**: Denmark does not have as prescriptive a technical baseline as Germany's IT-Grundschutz or Norway's Grunnprinsipper. CFCS guidance tends to be more principle-based. This means the Danish enclave may adopt NSM Grunnprinsipper or CIS benchmarks as supplementary technical baselines, with CFCS approval.

### 11.4 German Enclave Specifics

- **Accreditation authority**: BSI (for IT security), BMVg (for military authorization)
- **Primary hardening baseline**: BSI IT-Grundschutz Bausteine — this is extremely prescriptive and comprehensive. Key relevant Bausteine:
  - SYS.1.1 General Server
  - SYS.1.3 Linux Server
  - SYS.1.6 Containerization
  - APP.4.4 Kubernetes
  - NET.1.1 Network Architecture and Design
  - INF.2 Data Center
  - OPS.1.1.3 Patch and Change Management
  - CON.1 Crypto Concept
- **Crypto**: BSI-approved cryptographic products per BSI TR-02102 (Kryptographische Verfahren: Empfehlungen und Schlussellangen)
- **Personnel**: Sicherheitsuberprufung Ue3 (extended security check with security investigation) for GEHEIM access
- **Additional requirements**: BSI C5 criteria may apply if the enclave is characterized as a "cloud service" — the C5 catalogue has 17 domains of controls. Germany will likely require the most extensive documentation and compliance evidence among the three nations.
- **Key consideration**: BSI IT-Grundschutz compliance is not optional and is not replaceable by NSM or CFCS baselines. The German enclave must demonstrate IT-Grundschutz compliance as its primary accreditation evidence. This will drive additional documentation effort.

### 11.5 Shared NATO SECRET Domain Specifics

- **Accreditation**: NCIA or designated NATO accreditation authority
- **Governing policy**: C-M(2002)49 and AC/35 series
- **Hardening**: NATO-specific security requirements + host nation (Norway/NSM) baseline as minimum floor
- **Crypto**: NATO-approved cryptographic products
- **Personnel**: NATO SECRET clearance from any participating nation
- **Operational responsibility**: Shared between participating nations per a formal Memorandum of Understanding (MOU) defining responsibilities, cost sharing, and governance
- **Data governance**: All data in the shared domain is NATO SECRET with release markings. National caveats are enforced by the CDS on ingress.

---

## 12. Cryptographic Architecture

### 12.1 Principles

1. All data in transit within a domain is encrypted (mutual TLS between all services)
2. All data at rest is encrypted (LUKS/dm-crypt on all storage devices)
3. Cryptographic algorithms and products must be approved by the governing authority for each domain
4. Key management is per-domain — no shared key material across domain boundaries
5. The CDS terminates encryption on its ingress side and re-encrypts on its egress side — it must inspect plaintext content

### 12.2 Cryptographic Products by Domain

| Domain | Authority | Key Requirements |
|---|---|---|
| Norwegian Enclave | NSM | NSM-approved crypto products, NSM-approved key management |
| Danish Enclave | CFCS | CFCS/NATO-approved crypto |
| German Enclave | BSI | BSI TR-02102 compliant algorithms, BSI-approved products |
| Shared NATO Domain | NCIA | NATO-approved crypto per AC/35 |

### 12.3 TLS and Service Mesh

Within each domain's Kubernetes cluster:

- **cert-manager** with a per-domain CA (issued by the domain's PKI — FreeIPA CA or EJBCA)
- **Cilium** with transparent encryption (WireGuard or IPsec) for pod-to-pod communication
- Alternatively, **Istio** service mesh with mutual TLS for application-level encryption and observability

The choice between Cilium encryption and Istio service mesh depends on whether application-level traffic management (routing, retries, circuit breaking) is needed. For pure encryption, Cilium's transparent encryption is simpler and lower overhead.

### 12.4 Key Management

- **HashiCorp Vault** (per domain instance) for application-level secrets and encryption keys
- Vault auto-unseal using a Hardware Security Module (HSM) if available, or Shamir's Secret Sharing with split keys held by multiple cleared administrators
- Vault audit logging enabled — every secret access logged
- Key rotation policies aligned with national requirements (BSI TR-02102 specifies key lifetime recommendations)

---

## 13. TEMPEST and Emanation Security

### 13.1 SDIP-27 Zoning

All processing areas must meet NATO SDIP-27 emanation security requirements:

| Zone | Description | Application |
|---|---|---|
| Zone A | No emanation security requirement | Not applicable — all areas process classified data |
| Zone B | 20 dB attenuation at 1 GHz | Minimum for NATO SECRET processing rooms |
| Zone C | Equipment-level TEMPEST protection | Alternative if room-level shielding is impractical |

**Recommendation**: Zone B shielded rooms for all four domains. This means:
- Shielded room construction (Faraday cage) for each server room
- Filtered power entry points
- Fiber optic (non-conductive) cabling for inter-room connections where possible
- Waveguide air vents
- Shielded doors with proper contact/seal

### 13.2 Red/Black Separation

- Classified (red) and unclassified (black) signals must never share conductors, conduits, or cable trays
- Minimum separation distances per national TEMPEST guidance (NSM as host nation sets the minimum; if German or Danish requirements are stricter, apply the stricter standard to that nation's enclave)
- All KVM equipment (if used) must be TEMPEST-rated for the classification level
- Printers, if present, must be TEMPEST-rated or located within the shielded area

### 13.3 TEMPEST Inspection

- Initial TEMPEST inspection by NSM (or NSM-delegated body) for the facility
- Each national enclave may require TEMPEST inspection by its own national authority or acceptance of the host nation's inspection
- Periodic re-inspection as required (typically annually or upon significant equipment changes)

---

## 14. Monitoring, Audit, and Compliance

### 14.1 Monitoring Stack (Per Domain)

```
+------------------------------------------------------------------+
| Monitoring Architecture (replicated per domain)                    |
|                                                                    |
|  +------------------+     +------------------+                     |
|  | Prometheus       |---->| Alertmanager     |---> Alert routing   |
|  | (metrics)        |     |                  |     (within domain) |
|  +------------------+     +------------------+                     |
|                                                                    |
|  +------------------+     +------------------+                     |
|  | Promtail/Fluentd |---->| Loki             |                     |
|  | (log shipping)   |     | (log aggregation)|                     |
|  +------------------+     +------------------+                     |
|                                                                    |
|  +------------------+                                              |
|  | Grafana          |---> Dashboards for metrics, logs, alerts     |
|  +------------------+                                              |
|                                                                    |
|  +------------------+     +------------------+                     |
|  | Falco            |---->| Security event   |                     |
|  | (runtime sec.)   |     | pipeline         |                     |
|  +------------------+     +------------------+                     |
|                                                                    |
|  +------------------+                                              |
|  | OpenSCAP         |---> Continuous compliance scanning           |
|  | (compliance)     |                                              |
|  +------------------+                                              |
+------------------------------------------------------------------+
```

### 14.2 Audit Requirements

Classified systems require comprehensive audit trails. Design for the strictest requirement across all participating nations:

| Audit Requirement | Implementation |
|---|---|
| All authentication events | FreeIPA/AD audit logs + Keycloak event logs |
| All authorization decisions | Kubernetes audit logs (RequestReceived, ResponseComplete) |
| All administrative actions | Terminal session recording, kubectl audit logs |
| All data access | Application-level audit logging, Ceph access logs |
| All CDS transfers | CDS guard audit logs (source, destination, content hash, decision, approver) |
| All network connections | Firewall logs, Cilium flow logs |
| All system changes | Configuration management audit (Ansible/OpenTofu state) |

**Retention**: Minimum 5 years for SECRET-level systems (verify against each national requirement — BSI IT-Grundschutz OPS.1.1.5 may specify longer). Logs stored within the domain on tamper-evident storage (append-only, integrity-verified).

**Log integrity**: All audit logs must be integrity-protected. Options:
- Append-only filesystem (immutable storage)
- Cryptographic chaining (each log entry includes a hash of the previous entry)
- Write-once storage media for archival copies

### 14.3 Continuous Compliance

- **OpenSCAP** scans running on all nodes on a recurring schedule (daily minimum)
- Compliance profiles per domain:
  - Norwegian: Custom profile aligned with NSM Grunnprinsipper
  - Danish: Custom profile aligned with CFCS guidelines
  - German: BSI IT-Grundschutz profile (extremely detailed)
  - Shared NATO: NATO security requirements profile
- **Kyverno** policies enforcing Kubernetes-level compliance:
  - No privileged containers
  - No containers running as root (unless explicitly exempted)
  - All images from Harbor (no external registries)
  - All images signed
  - Resource limits on all pods
  - Network policies on all namespaces (default deny)
- Compliance dashboard in Grafana showing current posture per domain
- Automated alerting on compliance drift

---

## 15. Automation and Infrastructure as Code

### 15.1 Automation Strategy

All infrastructure is defined as code, stored in Git (per-domain Git repositories — no cross-domain Git access), and applied through automated pipelines.

**Toolchain**:

| Tool | Purpose | Notes |
|---|---|---|
| Ansible | OS configuration, hardening, day-2 operations | Offline execution with bundled collections |
| OpenTofu | Infrastructure provisioning (VMs via KubeVirt, storage, network objects) | State stored within domain |
| Helm | Kubernetes application packaging | Charts stored in Harbor within each domain |
| Kustomize | Kubernetes manifest customization | Used alongside or instead of Helm for simpler deployments |
| ArgoCD | GitOps continuous delivery | Per-domain instance, watches domain-local Git repo |
| Packer | Golden image building | OS images built with hardening baked in |

### 15.2 Offline Automation Pipeline

Since all domains are air-gapped, the automation pipeline must work entirely offline:

```
External (unclassified) build environment:
  1. Pull upstream packages, container images, Ansible collections, Helm charts
  2. Scan for vulnerabilities (Trivy, Grype)
  3. Build golden OS images (Packer)
  4. Build container images from verified sources
  5. Generate SBOMs (Syft)
  6. Sign all artifacts
  7. Package onto transfer media (encrypted, checksummed)

Controlled transfer point:
  8. Verify checksums and signatures
  9. Malware scan transfer media
  10. Transfer through data diode or manual import with chain-of-custody

Air-gapped domain:
  11. Import into domain's Harbor registry (images, Helm charts)
  12. Import into domain's package repository (RPMs/DEBs)
  13. Import into domain's Ansible collection mirror
  14. ArgoCD / Ansible detects new artifacts and applies updates per GitOps workflow
```

### 15.3 GitOps Workflow

Each domain runs its own Gitea or GitLab instance (FLOSS) for Git hosting:

```
Developer/Operator --> Git commit --> Gitea/GitLab --> ArgoCD detects change
                                                            |
                                                            v
                                                     ArgoCD syncs to
                                                     Kubernetes cluster
                                                            |
                                                            v
                                                     Deployment updated,
                                                     health checks pass,
                                                     compliance verified
```

- All changes require merge request with peer review (minimum two-person integrity for classified system changes)
- ArgoCD runs in "auto-sync with manual approval" mode for production namespaces
- Drift detection alerts on any manual changes to cluster state

---

## 16. Disaster Recovery and Business Continuity

### 16.1 Recovery Objectives

| Domain | RPO | RTO | Notes |
|---|---|---|---|
| Shared NATO Domain | 1 hour | 4 hours | Primary collaborative platform — highest priority |
| National Enclaves | 4 hours | 8 hours | Can operate semi-independently if shared domain is down |
| CDS Layer | N/A | 2 hours | If CDS is down, national enclaves and shared domain continue independently but cross-domain sharing stops |

### 16.2 Backup Strategy

- **Velero** for Kubernetes resource backup (per domain)
- **Ceph RBD snapshots** for persistent volume point-in-time recovery
- **Ceph pool-level** replication within the domain (3x replication provides resilience against drive and node failures)
- **Offline backup copies**: Periodic backup to encrypted removable media, stored in a secure vault within the facility (same classification level, separate physical location within the facility if possible)
- **etcd backup**: Automated etcd snapshots every 30 minutes, retained for 30 days

### 16.3 Failure Scenarios

| Scenario | Impact | Recovery |
|---|---|---|
| Single node failure | Kubernetes reschedules pods; Ceph rebuilds | Automatic — no intervention needed |
| Rack failure | Loss of nodes in one rack; Ceph data available from other racks | Automatic reschedule; replace failed hardware |
| Domain network failure | One domain offline | Other domains continue; CDS queues transfers |
| CDS failure | No cross-domain sharing | National enclaves and shared domain continue independently; CDS repaired/replaced |
| Facility-wide power loss | All domains offline | Generator auto-start; UPS bridges gap; graceful shutdown if generator fails |
| Facility compromise (physical) | Security incident | Incident response per national security protocols; potential system wipe and rebuild from backup |

### 16.4 Multi-Site Considerations

This architecture describes a single-site deployment. If multi-site DR is required:
- A secondary LIST-X equivalent facility (in Norway or another participating nation) would host a cold/warm standby
- Ceph RBD mirroring could replicate storage to the secondary site over an encrypted classified WAN link
- The secondary site would need the same accreditation as the primary
- Failover procedures would need to be agreed upon in the multinational MOU

---

## 17. Accreditation Strategy

### 17.1 Accreditation Approach

This system requires accreditation from four authorities:

1. **NSM** (Norway) — for the facility, the Norwegian national enclave, and host-nation oversight of the entire installation
2. **CFCS** (Denmark) — for the Danish national enclave and acceptance that Danish HEMMELIGT data is adequately protected in a Norwegian facility
3. **BSI/BMVg** (Germany) — for the German national enclave and acceptance that German GEHEIM data is adequately protected in a Norwegian facility
4. **NCIA** (or designated NATO authority) — for the shared NATO SECRET domain

### 17.2 Accreditation Coordination

A **Security Accreditation Board (SAB)** should be established with representatives from all four accrediting authorities. The SAB:
- Agrees on common security requirements where national frameworks overlap
- Identifies gaps where one nation's requirements exceed others
- Resolves conflicts between national requirements
- Coordinates accreditation timelines
- Manages the accreditation documentation package

### 17.3 Documentation Requirements

| Document | Purpose | Audience |
|---|---|---|
| System Security Plan (SSP) | Comprehensive description of security controls | All accrediting authorities |
| Security Operating Procedures (SyOPs) | Operational security procedures for daily operations | Operators, auditors |
| Risk Assessment | Threat analysis, vulnerability assessment, residual risk | SAB, national authorities |
| Interconnection Security Agreement (ISA) | Defines security requirements for each domain interconnection (CDS boundaries) | All authorities |
| TEMPEST Assessment | Emanation security evaluation | NSM + national authorities |
| Physical Security Plan | Facility security measures | NSM + national authorities |
| Cryptographic Security Plan | Crypto products, key management, algorithms | Each national authority for their domain |
| Incident Response Plan | Procedures for security incidents | All authorities |
| Configuration Management Plan | How changes are controlled and audited | All authorities |
| Continuous Monitoring Plan | Ongoing compliance and security monitoring approach | All authorities |

### 17.4 Accreditation-Driven Design

The architecture has been designed with accreditation in mind from the start:

- **Physical separation** between domains simplifies the accreditation boundary — each authority accredits their domain independently, plus the CDS boundary
- **Identical architecture pattern** across domains means accreditation evidence is reusable (with national customization)
- **Continuous compliance scanning** (OpenSCAP, Kyverno) provides automated evidence collection
- **Comprehensive audit logging** satisfies the audit requirements of all four authorities
- **Per-domain identity management** avoids the complexity of cross-domain identity federation, which would massively complicate accreditation

### 17.5 Accreditation Timeline (Estimated)

| Phase | Duration | Activities |
|---|---|---|
| Pre-accreditation planning | 3 months | Establish SAB, agree on documentation requirements, identify bilateral agreement gaps |
| Documentation development | 6 months | Write SSP, SyOPs, risk assessment, all supporting documents (in parallel with infrastructure build) |
| Infrastructure build | 6 months | Deploy hardware, configure platforms, implement security controls |
| Testing and assessment | 3 months | Penetration testing, compliance verification, TEMPEST inspection, CDS testing |
| Authority review | 3-6 months | Each authority reviews documentation and assessment results |
| Accreditation decision | 1-2 months | SAB and individual authorities issue accreditation |
| **Total estimated** | **18-24 months** | From project start to operational accreditation |

This timeline is aggressive. German BSI accreditation for GEHEIM systems can take 12+ months for the review phase alone. Early engagement with BSI is critical.

---

## 18. Supply Chain and Procurement

### 18.1 Principles

- All hardware and software must be procured through trusted supply chains
- Tamper-evident packaging for all hardware deliveries
- Chain-of-custody documentation from manufacturer to facility
- Firmware verification (checksums, signatures) before deployment
- SBOM (Software Bill of Materials) for all software components
- Prefer products from NATO member nations or allied countries

### 18.2 Hardware Procurement

| Component | Procurement Approach |
|---|---|
| Servers | Through national defense procurement frameworks (FLO/IFO in Norway, BAAINBw in Germany, FMI in Denmark) or direct from manufacturer with security requirements in contract |
| Network switches | Cisco via authorized defense reseller with TAA compliance |
| CDS equipment | Through CDS vendor's government sales channel with national authority approval |
| Storage (NVMe drives) | Through server vendor or defense-approved distributor |
| HSMs | From national-authority-approved HSM vendor |
| TEMPEST equipment | Through TEMPEST-certified suppliers |

### 18.3 Software and FLOSS

| Component | License | Supply Chain |
|---|---|---|
| RHEL / SLES | Commercial | Vendor subscription, offline repo mirror |
| RKE2 | Apache 2.0 (FLOSS) | SUSE/Rancher, verified from GitHub releases |
| Ceph (via Rook) | LGPL (FLOSS) | Verified from official releases |
| Harbor | Apache 2.0 (FLOSS) | Verified from official releases |
| Prometheus/Grafana/Loki | Apache 2.0 / AGPL (FLOSS) | Verified from official releases |
| Cilium | Apache 2.0 (FLOSS) | Verified from official releases |
| HashiCorp Vault | BSL (source-available, not FLOSS) | Evaluate OpenBao (FLOSS fork) as alternative |
| Ansible | GPL (FLOSS) | Verified from official releases |
| OpenTofu | MPL 2.0 (FLOSS) | Verified from official releases |

**Note on Vault licensing**: HashiCorp Vault moved to Business Source License (BSL), which is not FLOSS. **OpenBao** is the FLOSS fork (MPL 2.0). Evaluate OpenBao for this deployment. If Vault enterprise features are needed (HSM auto-unseal, namespaces), the BSL licensing terms must be reviewed for this use case.

---

## 19. Operational Model

### 19.1 Governance Structure

```
+---------------------------------------------------------------------+
|                    Multinational Project Board                        |
|              (Strategic decisions, budget, scope)                     |
+----+----------------------------+----------------------------+-------+
     |                            |                            |
+----v--------+          +--------v--------+          +--------v------+
| NOR National|          | DNK National    |          | DEU National  |
| Delegation  |          | Delegation      |          | Delegation    |
+----+--------+          +--------+--------+          +--------+------+
     |                            |                            |
     |     +----------------------+----------------------+     |
     |     |                                             |     |
     v     v                                             v     v
+----+-----+----+                                  +-----+-----+----+
| Security      |                                  | Technical      |
| Accreditation |                                  | Operations     |
| Board (SAB)   |                                  | Board          |
+---------------+                                  +----------------+
```

### 19.2 Operational Responsibilities

| Responsibility | Assigned To |
|---|---|
| Facility physical security | Host nation (Norway) facility security team |
| Norwegian enclave operations | Norwegian national delegation (cleared NOR personnel) |
| Danish enclave operations | Danish national delegation (cleared DNK personnel) |
| German enclave operations | German national delegation (cleared DEU personnel) |
| Shared NATO domain operations | Multinational operations team (cleared personnel from any participating nation) |
| CDS operations | Dedicated CDS team (host nation personnel with SAB oversight) |
| Incident response coordination | SAB with national CERT coordination (NSM NorCERT, CFCS, BSI CERT-Bund) |

### 19.3 Operating Procedures

- **Change management**: All changes to classified systems require change request, risk assessment, peer review, and approval. Changes to CDS require SAB approval. Changes within a national enclave require that nation's authority approval.
- **Patch management**: Monthly patch cycle for non-critical updates. Emergency patching for critical vulnerabilities with expedited approval. All patches enter through the controlled transfer point and are scanned/verified before deployment.
- **Incident response**: Tiered incident response — domain-level first responders escalate to facility-level coordination, which escalates to national CERTs as required. Security incidents affecting classified data trigger mandatory reporting to the relevant national authority.
- **On-call**: Each domain has an on-call rotation of cleared personnel. Minimum two-person response for any classified system incident.

### 19.4 Training

- All operators must be trained on:
  - The specific platform stack (RKE2, Ceph, Cilium, ArgoCD, Ansible)
  - Security procedures for classified systems
  - Incident response procedures
  - CDS operating procedures (for CDS operators)
- Training environment: A separate, unclassified lab environment replicating the architecture for training and testing. This avoids risking the classified environment for training purposes.

---

## 20. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R01 | German BSI accreditation delays due to IT-Grundschutz documentation volume | High | High | Engage BSI early (pre-design phase), dedicate German-speaking documentation resources, use BSI's own templates |
| R02 | CDS product not approved by all three national authorities | Medium | Critical | Identify CDS products already multi-nationally approved in NATO context. Engage all three authorities early on CDS selection. Have backup CDS product identified. |
| R03 | Clearance processing delays for multinational personnel | High | Medium | Start clearance requests 12+ months before operational need. Prioritize host-nation (Norwegian) personnel for initial operations. |
| R04 | Bilateral security agreement gaps for handling foreign classified data in Norway | Medium | Critical | Verify existing bilateral agreements cover this specific use case. Engage foreign ministries / defense ministries early. |
| R05 | TEMPEST non-compliance discovered during inspection | Low | High | Commission TEMPEST assessment during facility design phase, not after build. Use known-good shielded room vendors with defense track record. |
| R06 | Supply chain compromise of hardware or software | Low | Critical | Procure through defense supply chains, verify firmware, inspect hardware on delivery, maintain chain-of-custody documentation. |
| R07 | Key personnel attrition (cleared staff leaving) | Medium | High | Cross-train minimum three persons per domain, document procedures thoroughly, maintain training pipeline. |
| R08 | Divergent national requirements creating irreconcilable design conflicts | Medium | Medium | SAB resolves conflicts early. Design each enclave to meet its national requirements independently — the common architecture pattern reduces conflict surface. |
| R09 | Air-gapped patching lag creates vulnerability window | Medium | Medium | Establish monthly patch import cycle, prioritize critical security patches for expedited import, compensate with strong network segmentation and runtime security (Falco). |
| R10 | Scope creep to include additional nations or higher classification levels | Medium | High | Firm scope control through Project Board. Architecture is extensible (add another enclave) but each addition requires new bilateral agreements and accreditation. |

---

## 21. Architecture Decision Records

### ADR-001: Kubernetes Distribution Selection

**Status**: Accepted

**Context**: Need a Kubernetes distribution suitable for NATO SECRET processing across four domains (three national enclaves + shared NATO domain).

**Decision**: RKE2 (Rancher Kubernetes Engine 2)

**Rationale**:
- CIS-hardened by default (CIS Kubernetes Benchmark)
- FIPS 140-2 compliant mode available
- Uses containerd (not Docker) — smaller attack surface
- SUSE commercial support available for European defense customers
- Successfully deployed in US DoD environments (strong precedent)
- FLOSS (Apache 2.0 license) — no vendor lock-in

**Alternatives considered**:
- OpenShift: Common Criteria certified, but higher licensing cost and more opinionated
- Talos Linux: Immutable and API-only (excellent security properties), but smaller ecosystem and less defense sector precedent
- k3s: Too lightweight for this scale; lacks some enterprise features

### ADR-002: Physical vs. Logical Separation Between Domains

**Status**: Accepted

**Context**: Four classification domains must be separated. Options range from physical separation (separate hardware) to logical separation (VLANs, namespaces, network policies).

**Decision**: Full physical separation — separate servers, switches, cabling, and racks per domain.

**Rationale**:
- NATO SECRET processing with multiple national classifications requires the highest assurance of separation
- No national authority will accept VLAN-based separation between their national enclave and another nation's enclave at SECRET level
- Physical separation simplifies accreditation — each domain has a clean boundary
- Cost increase is justified by risk reduction and accreditation feasibility
- Logical separation (multi-tenancy within a single cluster) is acceptable WITHIN a domain (e.g., namespace separation within the Norwegian enclave) but not BETWEEN domains

### ADR-003: Cross-Domain Data Sharing Mechanism

**Status**: Accepted

**Context**: Data must flow between national enclaves and the shared NATO SECRET domain in a controlled, auditable manner.

**Decision**: Hardware CDS guards per national enclave boundary, with content inspection and release authority integration.

**Rationale**:
- Software-only solutions (API gateways, message brokers) do not provide the assurance level required for cross-classification-domain transfer
- Hardware CDS guards provide physical enforcement of data flow policies
- One CDS guard per national boundary provides isolation and independent auditability
- Content inspection ensures only properly marked, releasable data crosses boundaries

### ADR-004: Storage Platform Selection

**Status**: Accepted

**Context**: Need unified storage (block, file, object) across all domains.

**Decision**: Ceph deployed via Rook-Ceph operator within each domain's Kubernetes cluster.

**Rationale**:
- Unified storage backend (block, file, object) reduces operational complexity
- FLOSS — no licensing costs, no vendor lock-in
- Proven at scale in government and defense environments
- Native Kubernetes integration via Rook
- Self-healing and auto-rebalancing
- Alternative considered: Proprietary SAN/NAS — higher cost, licensing complexity, vendor dependency

### ADR-005: Network Fabric Design

**Status**: Accepted

**Context**: Each domain needs a network fabric. Options include Cisco ACI, traditional spine-leaf, and software-defined approaches.

**Decision**: Traditional spine-leaf with Cisco Nexus 9000 series, automated via Ansible (cisco.nxos collection). No ACI.

**Rationale**:
- ACI adds complexity (APIC controller, policy model) that is not justified for four relatively contained domains
- Traditional spine-leaf with VXLAN-EVPN (if needed) is simpler, well-understood, and automatable
- Cisco Nexus is widely deployed in defense environments with established support contracts
- Ansible automation provides IaC for network configuration without ACI's abstraction layer
- If a domain grows to a scale where ACI's policy model adds value, it can be introduced per-domain without affecting others

### ADR-006: FLOSS-First with Pragmatic Proprietary

**Status**: Accepted

**Context**: Balancing FLOSS principles with defense sector requirements for supported, certified products.

**Decision**: FLOSS by default; proprietary only where FLOSS does not meet a specific accreditation, certification, or support requirement.

**Rationale**:
- FLOSS reduces licensing costs, avoids vendor lock-in, and enables full code auditability (critical for classified environments)
- Proprietary exceptions: RHEL/SLES (OS support contracts required by most national authorities), Cisco Nexus (defense sector standard for networking), CDS hardware (no FLOSS alternative exists)
- Each proprietary choice is documented with justification

---

## Appendix A: Technology Stack Summary

| Layer | Technology | License |
|---|---|---|
| Operating System | RHEL 9 or SLES 15 SP5+ | Commercial |
| Kubernetes | RKE2 | Apache 2.0 (FLOSS) |
| Container Runtime | containerd | Apache 2.0 (FLOSS) |
| VM Workloads | KubeVirt | Apache 2.0 (FLOSS) |
| CNI / Network Policy | Cilium | Apache 2.0 (FLOSS) |
| Storage | Ceph (via Rook) | LGPL (FLOSS) |
| Container Registry | Harbor | Apache 2.0 (FLOSS) |
| Identity | FreeIPA / Keycloak | GPL / Apache 2.0 (FLOSS) |
| Secrets Management | OpenBao or HashiCorp Vault | MPL 2.0 (FLOSS) / BSL |
| Monitoring | Prometheus + Grafana + Alertmanager | Apache 2.0 / AGPL (FLOSS) |
| Logging | Loki + Promtail | AGPL (FLOSS) |
| Runtime Security | Falco + Tetragon | Apache 2.0 (FLOSS) |
| Policy Enforcement | Kyverno | Apache 2.0 (FLOSS) |
| Compliance Scanning | OpenSCAP | LGPL (FLOSS) |
| GitOps | ArgoCD | Apache 2.0 (FLOSS) |
| Git Hosting | Gitea or GitLab CE | MIT / MIT (FLOSS) |
| Automation | Ansible + OpenTofu | GPL / MPL 2.0 (FLOSS) |
| Image Building | Packer | BSL (source-available) |
| Network Switches | Cisco Nexus 9000 | Proprietary |
| Domain Firewalls | Palo Alto or Cisco Firepower | Proprietary |
| Cross-Domain Solution | Advenica SecuriCDS or equivalent | Proprietary |

## Appendix B: Network Port and Protocol Matrix

| Source | Destination | Protocol | Port | Purpose |
|---|---|---|---|---|
| Worker nodes | Control plane | TCP | 6443 | Kubernetes API |
| Worker nodes | Worker nodes | UDP | 51820 | Cilium WireGuard (pod-to-pod encryption) |
| All nodes | Ceph MON | TCP | 6789, 3300 | Ceph monitor |
| All nodes | Ceph OSD | TCP | 6800-7300 | Ceph OSD communication |
| All nodes | DNS | TCP/UDP | 53 | CoreDNS |
| All nodes | FreeIPA | TCP | 389, 636, 88, 464 | LDAP, LDAPS, Kerberos |
| Operators | API server | TCP | 6443 | kubectl access (via PAW) |
| Operators | Grafana | TCP | 443 | Monitoring dashboards (via PAW) |
| Prometheus | All nodes | TCP | 9090+ | Metrics scraping |
| Promtail | Loki | TCP | 3100 | Log shipping |
| ArgoCD | Gitea/GitLab | TCP | 443 | Git repository sync |
| Harbor | Internal only | TCP | 443 | Container image pull/push |
| Enclave FW | CDS Guard | TCP | Varies | Cross-domain data transfer |

## Appendix C: Compliance Mapping

| Requirement | NSM Grunnprinsipper | CFCS Guidelines | BSI IT-Grundschutz | NATO AC/35 |
|---|---|---|---|---|
| Access control | Beskytte 2.1-2.4 | Section 3 | ORP.4, SYS.1.1.M4 | AC/35-D/2004 |
| Encryption | Beskytte 3.1-3.3 | Section 4 | CON.1, BSI TR-02102 | AC/35-D/2001 |
| Audit logging | Oppdage 1.1-1.4 | Section 6 | OPS.1.1.5 | AC/35-D/2004 |
| Network security | Beskytte 1.1-1.7 | Section 5 | NET.1.1, NET.3.2 | AC/35-D/2004 |
| Incident response | Handtere 1.1-1.4 | Section 7 | DER.2.1 | AC/35-D/2004 |
| Physical security | Beskytte 4.1-4.3 | Section 2 | INF.1, INF.2 | C-M(2002)49 Ch. IV |
| Personnel security | Identifisere 2.1-2.3 | Section 1 | ORP.1 | C-M(2002)49 Ch. III |
| Configuration mgmt | Beskytte 5.1-5.3 | Section 4 | OPS.1.1.3 | AC/35-D/2004 |

---

*This architecture document should be reviewed by the Security Accreditation Board and updated as national requirements are clarified during the pre-accreditation planning phase. It represents a reference architecture — specific implementation details will be refined during detailed design.*
