# NATO SECRET Multinational Shared Infrastructure Architecture

## Multinational Defense Project: Norway, Denmark, Germany
### Classification: NATO SECRET / RELEASABLE TO NOR, DNK, DEU

---

**Document Control**

| Field | Value |
|---|---|
| Classification | NATO SECRET |
| Version | 1.0 |
| Date | 2026-03-20 |
| Status | Draft Architecture |
| Participating Nations | Norway (NOR), Denmark (DNK), Germany (DEU) |
| Hosting Nation | Norway |
| Facility Standard | LIST-X Equivalent (Norwegian NSM-approved) |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Policy Framework](#2-regulatory-and-policy-framework)
3. [National Security Authority Requirements](#3-national-security-authority-requirements)
4. [Facility and Physical Security Architecture](#4-facility-and-physical-security-architecture)
5. [Network Architecture](#5-network-architecture)
6. [Security Domain Architecture](#6-security-domain-architecture)
7. [Cross-Domain Solution Architecture](#7-cross-domain-solution-architecture)
8. [Compute and Platform Architecture](#8-compute-and-platform-architecture)
9. [Identity, Access Management, and Personnel Security](#9-identity-access-management-and-personnel-security)
10. [Data Governance and Information Handling](#10-data-governance-and-information-handling)
11. [Cryptographic Architecture](#11-cryptographic-architecture)
12. [Monitoring, Audit, and SIEM](#12-monitoring-audit-and-siem)
13. [Incident Response and TEMPEST](#13-incident-response-and-tempest)
14. [Accreditation Strategy](#14-accreditation-strategy)
15. [Operational Governance](#15-operational-governance)
16. [Risk Register](#16-risk-register)
17. [Reference Architecture Diagrams](#17-reference-architecture-diagrams)

---

## 1. Executive Summary

This document defines the reference architecture for a NATO SECRET-level shared infrastructure platform hosting a multinational defense project involving Norway, Denmark, and Germany. The platform is physically hosted in a LIST-X equivalent facility in Norway, operated under Norwegian national security regulations with bilateral and NATO oversight.

The architecture must satisfy several simultaneous constraints:

- **NATO Security Policy (C-M(2002)49)** requirements for handling NATO SECRET information
- **Norwegian** national security requirements under Sikkerhetsloven (Security Act) and NSM (Nasjonal sikkerhetsmyndighet) regulations
- **Danish** national security requirements under CFCS (Center for Cybersikkerhed) and FE (Forsvarets Efterretningstjeneste) guidelines
- **German** national security requirements under BSI (Bundesamt fur Sicherheit in der Informationstechnik) and BMVg (Bundesministerium der Verteidigung) regulations
- Proper separation between NATO-classified and nationally-classified data from each nation
- Controlled cross-domain information sharing where authorized

The architecture employs a **multi-enclave model** with a shared NATO SECRET common processing environment, nation-specific enclaves for nationally-classified data, and accredited cross-domain solutions (CDS) to enable controlled data flow between domains.

---

## 2. Regulatory and Policy Framework

### 2.1 NATO-Level Regulations

| Document | Applicability |
|---|---|
| C-M(2002)49 - Security Within NATO | Primary NATO security policy governing all classified information handling |
| AC/35-D/2004 - Primary Directive on INFOSEC | Technical security requirements for CIS handling NATO classified information |
| AC/35-D/2005 - INFOSEC Management Directive | Management framework for INFOSEC within NATO |
| AC/322-D/0048 - NATO CIS Security Accreditation | Accreditation framework for NATO CIS systems |
| STANAG 4774 / 4778 | Confidentiality labeling and metadata binding for NATO information |
| NIST/NATO Crypto requirements | Cryptographic standards for NATO SECRET communications |

### 2.2 Norwegian National Framework

| Regulation | Applicability |
|---|---|
| Sikkerhetsloven (Security Act, LOV-2018-06-01-24) | Primary legislation governing national security in Norway |
| Forskrift om virksomheters arbeid med forebyggende sikkerhet (Virksomhetsikkerhetsforskriften) | Enterprise security regulation |
| Forskrift om informasjonssikkerhet (Information Security Regulation) | Information security requirements for classified information |
| NSM Grunndokument for sikkerhetsgodkjenning av informasjonssystemer | System accreditation baseline |
| NSM Kravdokumenter for LIST-X / Leverandorklarering | Facility clearance requirements for LIST-X equivalent |
| Norwegian Crypto Policy (NSM) | National cryptographic requirements |

### 2.3 Danish National Framework

| Regulation | Applicability |
|---|---|
| Sikkerhedscirkulaeret (Security Circular) | Primary regulation for handling classified information |
| CFCS Technical Requirements for SECRET systems | Technical baseline for classified CIS |
| FE/CFCS Accreditation Standards | System accreditation framework |
| Danish Industrial Security (PET / FE joint guidance) | Industrial security for defense contractors |
| Danish Crypto Policy | National cryptographic requirements |

### 2.4 German National Framework

| Regulation | Applicability |
|---|---|
| Geheimschutzordnung (Classified Information Regulation) | Federal classified information handling rules |
| VS-Anweisung (VSA) | Administrative regulation for classified material |
| BSI TR-02102 (Cryptographic Mechanisms) | Cryptographic technical requirements |
| BSI Grundschutz (IT-Grundschutz) | IT baseline protection standards |
| BMVg InSan / IT-Sicherheitskonzept | Military IT security concept requirements |
| SuG (Sicherheitsuberpruefungsgesetz) | Personnel security vetting legislation |

### 2.5 Bilateral and Multilateral Agreements Required

Before this infrastructure can operate, the following agreements must be in place:

1. **Memorandum of Understanding (MOU)** between NOR, DNK, DEU defense ministries covering the project scope, data sharing, cost sharing, and liability
2. **Technical Arrangement (TA)** specifying the technical implementation details, accreditation responsibilities, and operational procedures
3. **Security Agreement (SOFA/Security Annex)** covering personnel, facility, and information security obligations
4. **Bilateral INFOSEC agreements** (or reaffirmation of existing ones) between each pair of nations for the exchange of nationally classified information
5. **NATO Communication and Information Agency (NCIA) coordination** for NATO SECRET network connectivity and accreditation
6. **Cross-Domain Transfer Policy (CDTP)** approved by all three NSAs and the NATO Security Accreditation Authority

---

## 3. National Security Authority Requirements

### 3.1 Accreditation Authority Mapping

Each security domain within the infrastructure requires accreditation from the relevant authority:

| Domain | Classification | Accreditation Authority (SAA) | Supporting Authority |
|---|---|---|---|
| NATO SECRET Common Domain | NATO SECRET | NCIA SAB (NATO Security Accreditation Board) via Norwegian DAA | NSM (NOR) as Host Nation SAA |
| Norwegian National Enclave | NOR HEMMELIG (SECRET) | NSM (Nasjonal sikkerhetsmyndighet) | Norwegian MOD |
| Danish National Enclave | DNK HEMMELIGT (SECRET) | CFCS / FE | Danish MOD |
| German National Enclave | DEU GEHEIM (SECRET) | BSI (in coordination with BMVg) | German MOD |
| Cross-Domain Solutions | Multiple | Joint accreditation by all involved SAAs | Requires consensus approval |

### 3.2 Norwegian Requirements (Host Nation)

As the host nation, Norway bears primary responsibility for:

- **Physical security** of the facility under NSM LIST-X requirements
- **Host Nation SAA** role for NATO SECRET systems (delegated from NCIA SAB)
- **TEMPEST** inspection and certification of the facility
- **Personnel security** clearance verification for all personnel with physical access
- **Incident response** coordination as first responder

Key Norwegian-specific technical requirements:
- Systems handling NOR HEMMELIG must comply with NSM's "Grunndokument for sikkerhetsgodkjenning"
- Encryption for NOR national classified data must use NSM-approved Type 1 crypto (e.g., Thales/Kongsberg products approved by NSM)
- Audit logs for NOR national systems must be accessible to NSM inspectors
- NOR national enclave must be physically or logically separable for NSM inspection without exposing other nations' data

### 3.3 Danish Requirements

Denmark participates as a contributing nation with data sovereign over DNK HEMMELIGT material:

- CFCS must approve the security concept (sikkerhedskoncept) for the Danish national enclave
- Danish national enclave must be operated by DNK-cleared personnel or under DNK-approved automated controls
- CFCS requires a dedicated security assessment of the cross-domain solution before DNK HEMMELIGT data can traverse it
- Danish personnel security clearances must be verified through the PET/FE channel
- Audit logs for DNK national systems must be exportable to CFCS in agreed format

### 3.4 German Requirements

Germany has historically stringent requirements for classified information systems:

- BSI must conduct or approve the IT-Sicherheitskonzept (IT security concept) for the German national enclave
- German national enclave must meet BSI IT-Grundschutz at the "hoch" (high) protection level minimum
- VS-NfD and above material requires BSI-certified encryption (Type 1 or BSI-approved)
- German law requires that DEU GEHEIM material remains under German sovereign control even when hosted abroad -- this necessitates specific contractual and technical arrangements
- BMVg requires a dedicated German Security Officer (Sicherheitsbevollmachtigter) with access to the German enclave
- Germany may require a dedicated hardware partition (not just logical separation) for DEU GEHEIM depending on BSI risk assessment outcome
- Audit logs and forensic capability for the German enclave must be available to BSI/BMVg inspectors

---

## 4. Facility and Physical Security Architecture

### 4.1 Facility Classification

The hosting facility must be accredited as a LIST-X equivalent under Norwegian NSM regulations. For multinational SECRET processing, the facility requirements exceed a standard LIST-X:

**Facility Security Level: NATO SECRET / National SECRET (NOR, DNK, DEU)**

### 4.2 Physical Security Zones

The facility employs a defense-in-depth physical security model with concentric security zones:

```
+------------------------------------------------------------------+
|  ZONE 0: PUBLIC AREA (Reception, Parking)                        |
|  - No classified processing                                      |
|  - Visitor management, badge issuance                            |
|  +------------------------------------------------------------+  |
|  |  ZONE 1: CONTROLLED AREA (Administrative offices)          |  |
|  |  - Unclassified / NATO RESTRICTED processing only           |  |
|  |  - Access control: badge + PIN                              |  |
|  |  +------------------------------------------------------+  |  |
|  |  |  ZONE 2: RESTRICTED AREA (Operations floor)          |  |  |
|  |  |  - NATO SECRET processing authorized                  |  |  |
|  |  |  - Access control: badge + biometric + PIN            |  |  |
|  |  |  - TEMPEST Zone 1 minimum                             |  |  |
|  |  |  +------------------------------------------------+  |  |  |
|  |  |  |  ZONE 3: HIGH-SECURITY AREA (Server rooms)     |  |  |  |
|  |  |  |  - All SECRET enclaves hosted here              |  |  |  |
|  |  |  |  - Access control: dual-person + biometric      |  |  |  |
|  |  |  |  - TEMPEST Zone 0                               |  |  |  |
|  |  |  |  - 24/7 CCTV with 90-day retention              |  |  |  |
|  |  |  |  - IDS (Intrusion Detection System)             |  |  |  |
|  |  |  |  +------------------------------------------+   |  |  |  |
|  |  |  |  | ZONE 3a: National Cages (per-nation)     |   |  |  |  |
|  |  |  |  | - Physical cage/partition per nation      |   |  |  |  |
|  |  |  |  | - Nation-specific access control          |   |  |  |  |
|  |  |  |  | - Tamper-evident seals                    |   |  |  |  |
|  |  |  |  +------------------------------------------+   |  |  |  |
|  |  |  +------------------------------------------------+  |  |  |
|  |  +------------------------------------------------------+  |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
```

### 4.3 National Cages (Zone 3a)

Within the high-security server room area, each nation receives a physically separated cage/enclosure:

| Cage | Contents | Access |
|---|---|---|
| **NOR National Cage** | Norwegian national enclave hardware, NOR crypto devices | NOR-cleared personnel only (dual-person) |
| **DNK National Cage** | Danish national enclave hardware, DNK crypto devices | DNK-cleared personnel only (dual-person) |
| **DEU National Cage** | German national enclave hardware, DEU crypto devices | DEU-cleared personnel only (dual-person) |
| **NATO Common Cage** | NATO SECRET common platform, CDS hardware, shared infrastructure | Multinational cleared personnel (NATO SECRET + host nation) |

Each national cage features:
- Independent lock systems (keys held by national security officer)
- Tamper-evident seals inspected at each access
- Dedicated power feeds with UPS (to prevent cross-cage power analysis)
- Dedicated HVAC or shared HVAC with TEMPEST-compliant ducting
- Independent fire suppression zones
- Cage-specific CCTV with footage accessible to the respective nation

### 4.4 TEMPEST Requirements

| Zone | TEMPEST Standard | Requirement |
|---|---|---|
| Zone 3 (Server rooms) | NATO SDIP-27 Level A (AMSG 720B) | Full TEMPEST protection, all equipment certified |
| Zone 2 (Operations floor) | NATO SDIP-27 Level B (AMSG 788A) | Zoned TEMPEST, workstations certified |
| National Cages | SDIP-27 Level A + national addenda | Each nation may impose additional TEMPEST requirements |
| Cable infrastructure | SDIP-29 | All cabling meets RED/BLACK separation requirements |

---

## 5. Network Architecture

### 5.1 Network Domain Overview

The infrastructure operates multiple physically and logically separated networks:

```
                        +-----------------------+
                        |   NATO WAN (NGCS/     |
                        |   BICES equivalent)   |
                        +-----------+-----------+
                                    |
                          [NATO Crypto - Type 1]
                                    |
        +---------------------------+---------------------------+
        |                                                       |
+-------+--------+                                    +---------+-------+
| NATO SECRET    |                                    | NATO Management |
| Common Domain  +--[CDS]--+--[CDS]--+--[CDS]--+    | Network (OOB)   |
| (Mission LAN)  |         |         |         |    +-----------------+
+----------------+         |         |         |
                           |         |         |
                 +---------+-+ +-----+-----+ +-+-----------+
                 | NOR       | | DNK       | | DEU         |
                 | National  | | National  | | National    |
                 | Enclave   | | Enclave   | | Enclave     |
                 | (HEMMELIG)| | (HEMMELIGT| | (GEHEIM)    |
                 +-----+-----+ +-----+-----+ +------+------+
                       |              |               |
                 [NOR Crypto]   [DNK Crypto]    [DEU Crypto]
                       |              |               |
                 +-----+-----+ +-----+-----+ +------+------+
                 | NOR Natl   | | DNK Natl  | | DEU Natl    |
                 | WAN (FISBasis| | WAN (FIIN | | WAN (BWI/   |
                 | /NorMilNet)| | equiv.)   | | FuInfoSysLw)|
                 +-----------+ +-----------+ +-------------+
```

### 5.2 Network Segmentation Rules

**Principle: Air-gap or accredited CDS between every classification domain.**

| Source | Destination | Connection Type | Mechanism |
|---|---|---|---|
| NATO SECRET Common | NOR National | CDS (accredited) | Hardware-enforced, one-way or controlled bidirectional |
| NATO SECRET Common | DNK National | CDS (accredited) | Hardware-enforced, one-way or controlled bidirectional |
| NATO SECRET Common | DEU National | CDS (accredited) | Hardware-enforced, one-way or controlled bidirectional |
| NOR National | DNK National | **Prohibited (direct)** | Must traverse via NATO common with dual CDS |
| NOR National | DEU National | **Prohibited (direct)** | Must traverse via NATO common with dual CDS |
| DNK National | DEU National | **Prohibited (direct)** | Must traverse via NATO common with dual CDS |
| Any SECRET domain | Management Network | Out-of-band only | Physically separate management plane |
| Any domain | Internet/UNCLASS | **Prohibited** | Air-gapped from all classified domains |

### 5.3 NATO SECRET Common Domain Network

The NATO SECRET common domain serves as the primary collaboration environment:

- **Classification:** NATO SECRET REL NOR/DNK/DEU
- **Purpose:** Shared mission processing, collaboration tools, shared data stores
- **WAN Connectivity:** Connected to NATO General Communications System (NGCS) or BICES network via NATO-approved Type 1 encryption
- **Internal Network:**
  - Core switches: Redundant pair, NATO SECRET accredited
  - VLAN segmentation for functional separation (compute, storage, management, DMZ)
  - 802.1X port-based access control
  - All traffic encrypted in transit (TLS 1.3 minimum, NATO-approved cipher suites)
  - Network IDS/IPS at all enclave boundaries

### 5.4 National Enclave Networks

Each national enclave operates as an independent security domain:

**NOR National Enclave:**
- Classification: NOR HEMMELIG
- WAN: Connected to Norwegian military networks (FISBasis/NorMilNet) via NSM-approved Type 1 crypto
- Internal: Independent switching infrastructure within NOR cage
- Firewall: Norwegian-accredited firewall between NOR enclave and CDS

**DNK National Enclave:**
- Classification: DNK HEMMELIGT
- WAN: Connected to Danish military networks (FIIN or equivalent) via CFCS-approved Type 1 crypto
- Internal: Independent switching infrastructure within DNK cage
- Firewall: Danish-accredited firewall between DNK enclave and CDS

**DEU National Enclave:**
- Classification: DEU GEHEIM
- WAN: Connected to German military networks (BWI/FuInfoSysLw) via BSI-approved Type 1 crypto (e.g., SINA or Elcrodat)
- Internal: Independent switching infrastructure within DEU cage
- Firewall: German-accredited (BSI-certified) firewall between DEU enclave and CDS

### 5.5 Management Network

A dedicated, physically separate out-of-band (OOB) management network:

- **Physically separate** cabling and switching from all operational networks
- Used for: hardware management (IPMI/iLO/iDRAC), infrastructure monitoring, configuration management
- Classification: treated at the highest classification level present in the facility
- Access: restricted to accredited system administrators with appropriate clearances
- No routing or connectivity to any operational domain

---

## 6. Security Domain Architecture

### 6.1 Domain Model

The infrastructure implements a **Multi-Domain Security Architecture (MDSA)** with the following domains:

```
+-------------------------------------------------------------------+
|                    SECURITY DOMAIN MODEL                          |
+-------------------------------------------------------------------+
|                                                                   |
|  +---------------------+  +-----------------------+               |
|  | NATO SECRET COMMON  |  | MANAGEMENT DOMAIN     |               |
|  | REL NOR/DNK/DEU     |  | (Out-of-Band)         |               |
|  |                     |  |                       |               |
|  | - Shared compute    |  | - Infrastructure mgmt |               |
|  | - Collaboration     |  | - Monitoring          |               |
|  | - Mission apps      |  | - Patching            |               |
|  +----------+----------+  +-----------+-----------+               |
|             |                         |                           |
|     [CDS Gateway Cluster]    [OOB Physical Separation]            |
|      |          |       |                                         |
|  +---+---+ +---+---+ +-+-----+                                   |
|  | NOR   | | DNK   | | DEU   |                                   |
|  | HEMM. | | HEMM. | | GEH.  |                                   |
|  |       | |       | |       |                                   |
|  | Natl  | | Natl  | | Natl  |                                   |
|  | Apps  | | Apps  | | Apps  |                                   |
|  | Natl  | | Natl  | | Natl  |                                   |
|  | Data  | | Data  | | Data  |                                   |
|  +-------+ +-------+ +-------+                                   |
+-------------------------------------------------------------------+
```

### 6.2 Data Classification and Labeling

All data within the infrastructure must be classified and labeled according to STANAG 4774 (Confidentiality Metadata Label Syntax) and STANAG 4778 (Metadata Binding Mechanism):

| Label | Marking | Handling |
|---|---|---|
| `NATO SECRET` | `NS` | Processable in NATO SECRET Common Domain |
| `NATO SECRET REL NOR/DNK/DEU` | `NS REL NOR/DNK/DEU` | Processable in NATO Common; releasable subset |
| `NOR HEMMELIG` | National caveat | NOR enclave only; release requires NOR SAA approval |
| `DNK HEMMELIGT` | National caveat | DNK enclave only; release requires DNK SAA approval |
| `DEU GEHEIM` | National caveat | DEU enclave only; release requires DEU SAA approval |
| `NATO SECRET // NOR HEMMELIG` | Dual marking | Requires access to both domains; stored in NOR enclave |
| `NATO SECRET // DEU GEHEIM` | Dual marking | Requires access to both domains; stored in DEU enclave |

### 6.3 Sensitivity Compartments

Beyond classification levels, the architecture supports compartmented handling:

- **Project compartment:** Data specific to the multinational project receives a project-specific compartment marking (e.g., `PROJECT NORTHERN SHIELD`)
- **National compartments:** Each nation may define sub-compartments within their enclave
- **Need-to-know enforcement:** Mandatory access controls (MAC) enforce compartment boundaries; discretionary access controls (DAC) enforce need-to-know within a compartment

---

## 7. Cross-Domain Solution Architecture

### 7.1 CDS Requirement

Cross-domain solutions are the most critical and scrutinized components of this architecture. Every data transfer between security domains must transit an accredited CDS.

### 7.2 CDS Deployment Model

Three CDS instances are deployed, one per national enclave boundary:

```
+----------------+     +-------------------+     +------------------+
| NOR National   |<--->| CDS-NOR           |<--->| NATO SECRET      |
| Enclave        |     | (Bidirectional,   |     | Common Domain    |
|                |     |  content-filtered)|     |                  |
+----------------+     +-------------------+     +--------+---------+
                                                          |
+----------------+     +-------------------+              |
| DNK National   |<--->| CDS-DNK           |<------------>|
| Enclave        |     | (Bidirectional,   |              |
|                |     |  content-filtered)|              |
+----------------+     +-------------------+              |
                                                          |
+----------------+     +-------------------+              |
| DEU National   |<--->| CDS-DEU           |<------------>|
| Enclave        |     | (Bidirectional,   |
|                |     |  content-filtered)|
+----------------+     +-------------------+
```

### 7.3 CDS Technical Requirements

Each CDS must meet the following requirements:

**Hardware Architecture:**
- Dedicated, tamper-evident hardware (not virtualized)
- Physically located in the NATO Common cage (accessible to all nations' inspectors)
- Redundant pair for availability (active/standby)
- Hardware security module (HSM) integrated for cryptographic operations and data integrity verification

**Functional Requirements:**
- **Content inspection:** Deep content inspection of all transiting data for classification markings, keywords, patterns
- **Data type enforcement:** Only approved data types/formats may transit (e.g., approved XML schemas, specific file formats)
- **Dirty word checking:** Configurable keyword lists per nation to prevent unauthorized release of national information
- **Malware scanning:** Multi-engine antivirus/anti-malware scanning at the CDS boundary
- **Digital signature verification:** All data transiting the CDS must be digitally signed by an authorized originator
- **Audit logging:** Complete audit trail of every object transiting the CDS, with tamper-proof log storage
- **Manual release option:** For certain data categories, the CDS supports a "human review and release" workflow where a cleared reviewer must approve transfer
- **Rate limiting and anomaly detection:** Prevent bulk data exfiltration

**Policy Enforcement:**
- Each nation maintains a **Transfer Policy** that is enforced at their respective CDS
- Transfer policies are configured jointly but each nation holds veto authority over their CDS configuration
- Policy changes require formal change management with multi-national approval

### 7.4 CDS Product Candidates

The following CDS products are commonly used in NATO environments at SECRET level:

| Product | Vendor | Notes |
|---|---|---|
| HIGHWALL | Forcepoint (Everfox) | Widely deployed in NATO/Five Eyes; accredited to SECRET in multiple nations |
| Nexor Sentinel | Nexor | UK-origin; accredited in NATO environments; strong XML filtering |
| ISSE Guard | BAE Systems | US-origin; widely used in NATO; content-based filtering |
| Deep Secure (Forcepoint) | Forcepoint | Content threat removal; transform-based approach |

**Recommendation:** Deploy Forcepoint HIGHWALL or Nexor Sentinel due to existing NATO SECRET accreditations and familiarity within European defense establishments. Final selection subject to accreditation feasibility assessment by all three NSAs.

### 7.5 Cross-Domain Data Flows

Authorized data flows between domains:

| Flow ID | Source Domain | Destination Domain | Data Type | Direction | Review |
|---|---|---|---|---|---|
| F-001 | NATO Common | NOR Enclave | Mission tasking, shared products | NATO -> NOR | Automated + policy |
| F-002 | NOR Enclave | NATO Common | NOR-released intelligence, reports | NOR -> NATO | Manual release or automated with NOR policy |
| F-003 | NATO Common | DNK Enclave | Mission tasking, shared products | NATO -> DNK | Automated + policy |
| F-004 | DNK Enclave | NATO Common | DNK-released intelligence, reports | DNK -> NATO | Manual release or automated with DNK policy |
| F-005 | NATO Common | DEU Enclave | Mission tasking, shared products | NATO -> DEU | Automated + policy |
| F-006 | DEU Enclave | NATO Common | DEU-released intelligence, reports | DEU -> NATO | Manual release or automated with DEU policy |
| F-007 | NOR -> DNK (via NATO) | Dual CDS transit | Bilaterally agreed data | Bidirectional | Dual manual release |
| F-008 | NOR -> DEU (via NATO) | Dual CDS transit | Bilaterally agreed data | Bidirectional | Dual manual release |
| F-009 | DNK -> DEU (via NATO) | Dual CDS transit | Bilaterally agreed data | Bidirectional | Dual manual release |

---

## 8. Compute and Platform Architecture

### 8.1 Platform Strategy

The infrastructure uses a **private cloud** model built on hardened virtualization technology. Public cloud services are not authorized for NATO SECRET processing.

### 8.2 Hypervisor and Virtualization

**Common Criteria-evaluated hypervisor** is required. Options include:

| Platform | Certification | Suitability |
|---|---|---|
| VMware vSphere (hardened) | CC EAL4+ | Widely deployed in NATO SECRET environments; strong ecosystem |
| Nutanix AHV (hardened) | CC EAL2+ | Hyperconverged; simplifies deployment; newer to classified environments |
| KVM (hardened, e.g., Red Hat) | Varies by configuration | Open source base; requires extensive hardening; some national preferences |

**Recommendation:** VMware vSphere 8.x in hardened configuration, leveraging existing NATO SECRET accreditation precedents.

### 8.3 Hardware Architecture

Each domain operates on dedicated physical hardware:

**NATO SECRET Common Domain:**
- 6-8 x rack-mount compute nodes (e.g., Dell PowerEdge or HPE ProLiant, NATO-approved supply chain)
- 2 x SAN/NAS storage arrays (encrypted at rest, FIPS 140-3 Level 2 minimum)
- 2 x core switches + 2 x access switches
- 2 x firewalls (next-generation, NATO-accredited)
- 2 x CDS pairs (one per national enclave boundary, plus one spare)
- 2 x load balancers (if web services required)
- HSM cluster for key management

**Per National Enclave (NOR, DNK, DEU each):**
- 2-4 x rack-mount compute nodes
- 1 x storage array (encrypted at rest, national crypto requirements)
- 2 x switches
- 1 x firewall (nationally accredited)
- National crypto devices for WAN connectivity
- National HSM or key management device

### 8.4 Server Hardening

All servers must be hardened to the following standards:

- **Operating System:** Red Hat Enterprise Linux 8/9 (DISA STIG applied) or Windows Server 2022 (DISA STIG applied), depending on workload requirements
- **CIS Benchmarks:** Applied as baseline, with deviations documented and risk-accepted
- **DISA STIG / NATO STIG equivalent:** All applicable STIGs applied
- **Unnecessary services disabled:** Minimal installation; no unnecessary packages
- **Local firewalls enabled:** Host-based firewall on every system
- **Endpoint protection:** Approved anti-malware on all systems
- **Secure boot:** UEFI Secure Boot enabled; measured boot where supported
- **Disk encryption:** Full disk encryption with national or NATO-approved algorithms
- **USB/removable media:** Disabled at BIOS and OS level; exceptions require formal approval

### 8.5 Storage Architecture

```
+--------------------------------------------------+
|           STORAGE ARCHITECTURE                    |
+--------------------------------------------------+
|                                                   |
|  NATO SECRET Common Storage                       |
|  +---------------------------------------------+ |
|  | Primary SAN (iSCSI/FC)                       | |
|  | - VM datastores (VMFS/NFS)                   | |
|  | - Database storage (Oracle/PostgreSQL)        | |
|  | - File shares (SMB/NFS - NATO SECRET)        | |
|  | - Encryption: AES-256 at rest                | |
|  | - Replication to secondary SAN               | |
|  +---------------------------------------------+ |
|                                                   |
|  NOR Enclave Storage    DNK Enclave Storage       |
|  +------------------+  +------------------+       |
|  | NOR SAN          |  | DNK SAN          |       |
|  | - NOR national   |  | - DNK national   |       |
|  |   data only      |  |   data only      |       |
|  | - NSM-approved   |  | - CFCS-approved  |       |
|  |   encryption     |  |   encryption     |       |
|  +------------------+  +------------------+       |
|                                                   |
|  DEU Enclave Storage                              |
|  +------------------+                             |
|  | DEU SAN          |                             |
|  | - DEU national   |                             |
|  |   data only      |                             |
|  | - BSI-approved   |                             |
|  |   encryption     |                             |
|  +------------------+                             |
+--------------------------------------------------+
```

All storage must implement:
- Encryption at rest (AES-256 minimum; algorithm must be approved by relevant NSA)
- Access controls aligned with the domain security policy
- Snapshot and backup capability (backups stored at same or higher classification)
- Secure deletion capability (cryptographic erasure or degaussing per NATO/national standards)
- No shared storage across security domains

---

## 9. Identity, Access Management, and Personnel Security

### 9.1 Personnel Security Clearance Requirements

| Access Level | Required Clearance | Verification |
|---|---|---|
| Facility access (Zone 0-1) | Uncleared (escorted) or RESTRICTED | Host nation (NSM) verification |
| Zone 2 (Operations) | NATO SECRET | National vetting agency confirmation via NSM |
| NATO Common Domain | NATO SECRET + project briefing | NATO SECRET clearance from home nation, verified through NPSC |
| NOR National Enclave | NOR HEMMELIG + NOR national briefing | NSM clearance verification |
| DNK National Enclave | DNK HEMMELIGT + DNK national briefing | PET/FE clearance verification |
| DEU National Enclave | DEU GEHEIM + DEU national briefing | BMVg/MAD clearance verification |
| CDS Administration | NATO SECRET + all relevant national clearances | Multi-national verification |
| System Administration | NATO SECRET + relevant national clearances for domains administered | Verified by respective national security officer |

### 9.2 Identity Management Architecture

```
+---------------------------------------------------+
|           IDENTITY ARCHITECTURE                    |
+---------------------------------------------------+
|                                                    |
|  +----------------------------------------------+ |
|  | Central Identity Store (NATO Common Domain)   | |
|  | - Active Directory / LDAP                     | |
|  | - PKI integration (NATO/national certificates)| |
|  | - Multi-factor authentication required         | |
|  | - Smart card / CAC / national equivalent       | |
|  +---+------------------------------------------+ |
|      |                                             |
|      | Trust (one-way, filtered)                    |
|      |                                             |
|  +---+---+ +--------+ +--------+                  |
|  | NOR   | | DNK    | | DEU    |                  |
|  | AD/   | | AD/    | | AD/    |                  |
|  | LDAP  | | LDAP   | | LDAP   |                  |
|  | (Natl)| | (Natl) | | (Natl) |                  |
|  +-------+ +--------+ +--------+                  |
|                                                    |
+---------------------------------------------------+
```

**Key design decisions:**
- Each domain maintains its own identity store (Active Directory or LDAP)
- NATO Common Domain has a dedicated AD forest
- National enclaves have independent AD forests
- **No cross-forest trusts between national enclaves** (prevents lateral identity compromise)
- Limited, one-way filtered trusts from national enclaves to NATO Common (if required for SSO)
- Alternative: federated identity using SAML/OIDC with CDS-mediated token exchange (preferred for reduced trust surface)

### 9.3 Authentication Requirements

| Domain | Primary Authentication | MFA Requirement |
|---|---|---|
| NATO Common | National smart card / PKI certificate | Smart card + PIN (something you have + something you know) |
| NOR Enclave | Norwegian national PKI card (Buypass/Commfides or military equivalent) | Card + PIN |
| DNK Enclave | Danish national PKI / military ID card | Card + PIN |
| DEU Enclave | German Truppenausweis / BSI-approved smart card | Card + PIN |
| Administrative access | PKI + additional factor | Smart card + PIN + approval workflow |

### 9.4 Authorization Model

- **Role-Based Access Control (RBAC)** for functional role assignments
- **Mandatory Access Control (MAC)** for classification enforcement
  - Bell-LaPadula model: no read-up, no write-down enforced at the OS and application level
- **Attribute-Based Access Control (ABAC)** for fine-grained data access decisions incorporating:
  - User clearance level
  - User nationality
  - User project role
  - Data classification
  - Data releasability markings
  - Data compartment markings
  - Time-based constraints (if applicable)

### 9.5 Privileged Access Management (PAM)

- Dedicated PAM solution (e.g., CyberArk, BeyondTrust) deployed per domain
- Privileged credentials vaulted and rotated automatically
- Session recording for all administrative access
- Just-in-time (JIT) access provisioning for administrative tasks
- Break-glass procedures documented and tested
- Multi-person authorization for critical operations (e.g., CDS configuration changes, crypto key operations)

---

## 10. Data Governance and Information Handling

### 10.1 Data Sovereignty Principles

1. **National data remains under national control.** Each nation retains sovereign authority over its nationally-classified data, even when hosted in the Norwegian facility.
2. **No nation may access another nation's nationally-classified data** without explicit bilateral authorization.
3. **Data released to the NATO Common Domain** is subject to NATO handling rules and the agreed Transfer Policy.
4. **Classification authority rests with the originator.** Only the originating nation may declassify or change releasability of its data.
5. **Right to audit.** Each nation has the right to audit the handling of its data within the infrastructure.

### 10.2 Data Lifecycle Management

| Phase | Requirements |
|---|---|
| **Creation** | Data must be classified and labeled at creation (STANAG 4774/4778 metadata) |
| **Storage** | Encrypted at rest; stored only in authorized domain; access logged |
| **Processing** | Processed only in authorized domain; no caching in unauthorized domains |
| **Sharing** | Via accredited CDS only; subject to Transfer Policy and release authority |
| **Archival** | Retained per NATO/national retention policies; encrypted archive |
| **Destruction** | Cryptographic erasure or physical destruction per NATO CM(2002)49 Annex V / national equivalents |

### 10.3 Data Loss Prevention (DLP)

- DLP agents deployed on all endpoints and servers within each domain
- DLP policies tuned per domain:
  - NATO Common: prevent exfiltration of NATO SECRET data to lower domains
  - National enclaves: prevent exfiltration of national data beyond authorized channels
- DLP integration with CDS for content-aware filtering
- USB/removable media blocked; exceptions require formal approval and cryptographic protection

### 10.4 Backup and Recovery

- **Per-domain backup:** Each domain has independent backup infrastructure; no cross-domain backup
- **Backup encryption:** All backups encrypted with domain-specific keys (key escrow per national/NATO requirements)
- **Backup storage:** On-site secure storage (same facility, same classification zone) + potential off-site at national facilities (transported via approved courier)
- **Recovery testing:** Quarterly recovery tests per domain
- **Backup retention:** Per NATO/national records management policies (typically 5-7 years for operational data)

---

## 11. Cryptographic Architecture

### 11.1 Cryptographic Requirements by Domain

| Domain | Network Encryption | Data at Rest | Key Management | Approved Products |
|---|---|---|---|---|
| NATO SECRET Common | NATO Type 1 (WAN); TLS 1.3 (LAN) | AES-256 (FIPS 140-3) | NATO KMS / HSM | NATO-approved crypto catalog |
| NOR National | NSM Type 1 (WAN); TLS 1.3 (LAN) | AES-256 (NSM-approved) | NSM KMS / HSM | NSM-approved crypto list |
| DNK National | CFCS Type 1 (WAN); TLS 1.3 (LAN) | AES-256 (CFCS-approved) | CFCS KMS / HSM | CFCS-approved crypto list |
| DEU National | BSI Type 1 (WAN, e.g., SINA); TLS 1.3 (LAN) | AES-256 (BSI-approved) | BSI KMS / HSM | BSI VS-Produktkatalog |

### 11.2 Key Management

- **Per-domain key management:** Each domain operates its own key management infrastructure; keys never shared across domains
- **HSM-backed:** All cryptographic keys stored in FIPS 140-3 Level 3 (minimum) or national equivalent HSMs
- **Key lifecycle:** Generation, distribution, activation, rotation, deactivation, destruction -- all logged and auditable
- **Key escrow:** Per national/NATO requirements; escrow keys held by respective NSA
- **Crypto-period:** Per NATO/national crypto policies (typically 1-2 years for symmetric keys, longer for asymmetric with re-keying)

### 11.3 PKI Architecture

- **NATO Common Domain:** Uses NATO PKI or project-specific PKI subordinate to NATO PKI
- **National Enclaves:** Use national military PKI (NOR: Norwegian military PKI, DNK: Danish military PKI, DEU: German Bundeswehr PKI)
- **Cross-domain certificate trust:** NOT established directly; CDS handles protocol break and re-encapsulation
- **Certificate validation:** OCSP/CRL checking within each domain; CDS does not forward certificate validation across domains

---

## 12. Monitoring, Audit, and SIEM

### 12.1 SIEM Architecture

Each security domain operates an independent SIEM instance:

```
+--------------------------------------------------+
|              SIEM ARCHITECTURE                    |
+--------------------------------------------------+
|                                                   |
|  +---------------------------------------------+ |
|  | NATO Common SIEM                             | |
|  | - Collects: NATO Common Domain logs          | |
|  | - Collects: CDS logs (all three CDS)         | |
|  | - Collects: Shared infrastructure logs       | |
|  | - Accessible to: multinational SOC team      | |
|  +---------------------------------------------+ |
|                                                   |
|  +-------------+ +-------------+ +-------------+ |
|  | NOR SIEM    | | DNK SIEM    | | DEU SIEM    | |
|  | NOR enclave | | DNK enclave | | DEU enclave | |
|  | logs only   | | logs only   | | logs only   | |
|  | NOR SOC     | | DNK SOC     | | DEU SOC     | |
|  +-------------+ +-------------+ +-------------+ |
|                                                   |
+--------------------------------------------------+
```

### 12.2 Log Sources

All of the following must be logged and forwarded to the respective domain SIEM:

- Operating system security events (authentication, authorization, privilege escalation)
- Application-level audit logs
- Network device logs (firewalls, switches, routers, IDS/IPS)
- CDS transit logs (every object that passes or is rejected)
- Hypervisor security events
- Storage access logs
- Physical access logs (badge reader events)
- CCTV metadata (access to footage requires physical security officer)
- Cryptographic device logs
- PAM session logs

### 12.3 Audit Requirements

| Requirement | Standard | Implementation |
|---|---|---|
| Log integrity | Tamper-proof logging | Write-once storage or cryptographic chaining (blockchain-style log chains) |
| Log retention | Minimum 2 years (NATO); national requirements may be longer | Archived to encrypted offline storage after 90 days online |
| Log review | Daily automated review + weekly human review | SIEM correlation rules + SOC analyst review |
| Cross-domain correlation | Limited to CDS transit logs visible to all parties | NATO Common SIEM correlates CDS logs from all three boundaries |
| National audit access | Each nation can audit their own enclave + shared infrastructure | Dedicated audit accounts per nation; read-only SIEM access |
| NATO audit access | NATO NCIA/SAB may audit NATO Common Domain | NCIA audit team access provisions |

### 12.4 Security Operations Center (SOC)

A multinational SOC operates for the NATO Common Domain:

- **Staffing:** Representatives from all three nations during operational hours
- **Location:** Zone 2 (Operations floor) with dedicated SOC room
- **Responsibilities:** Monitor NATO Common Domain, CDS alerts, coordinate with national SOC elements
- **Escalation:** National enclave incidents escalated to respective national SOC/CERT (NorCERT, CFCS CERT, CERT-Bw)

---

## 13. Incident Response and TEMPEST

### 13.1 Incident Response Framework

```
+---------------------------------------------------+
|         INCIDENT RESPONSE STRUCTURE                |
+---------------------------------------------------+
|                                                    |
|  +----------------------------------------------+ |
|  | Multinational Incident Response Team (MIRT)   | |
|  | - Lead: Host Nation (NOR) Security Officer     | |
|  | - Members: NOR/DNK/DEU security representatives| |
|  | - Scope: NATO Common Domain + CDS incidents    | |
|  +----------------------------------------------+ |
|                                                    |
|  +------+ +------+ +------+                        |
|  | NOR  | | DNK  | | DEU  |                        |
|  | IRT  | | IRT  | | IRT  |                        |
|  | Natl | | Natl | | Natl |                        |
|  +------+ +------+ +------+                        |
|                                                    |
|  Escalation Path:                                  |
|  Local IRT -> MIRT -> National CERT -> NATO NCIRC  |
+---------------------------------------------------+
```

### 13.2 Incident Categories and Response

| Category | Description | Lead | Notification |
|---|---|---|---|
| **CAT 1: Compromise** | Confirmed compromise of classified information | MIRT + affected national IRT | Immediate: all NSAs + NATO Office of Security |
| **CAT 2: Cyber intrusion** | Unauthorized access to classified system | MIRT (NATO Common) or national IRT (enclave) | Within 1 hour: all participating nations + NCIRC |
| **CAT 3: Malware** | Malware detected on classified system | Domain-specific IRT | Within 4 hours: relevant SAA |
| **CAT 4: Policy violation** | Security policy violation (no compromise) | Local security officer | Within 24 hours: relevant SAA |
| **CAT 5: Anomaly** | Suspicious activity requiring investigation | SOC team | Logged; escalated if confirmed |

### 13.3 TEMPEST Incident Procedures

- Any suspected TEMPEST compromise triggers immediate notification to NSM (as host nation) and to the affected national SAA
- TEMPEST testing conducted annually by NSM-accredited TEMPEST testing facility
- National enclaves subject to additional TEMPEST verification by their respective NSAs
- All new equipment installations require TEMPEST assessment before operational use

### 13.4 Forensic Capability

- Forensic toolkit maintained per domain (cannot cross domain boundaries)
- Forensic workstations: air-gapped, dedicated, stored in secure locker when not in use
- Forensic procedures documented per NATO NCIRC guidance and national equivalents
- Chain of custody procedures aligned with both military and potential legal proceedings requirements
- Each nation has the right to participate in forensic investigation of incidents affecting their data

---

## 14. Accreditation Strategy

### 14.1 Accreditation Approach

The infrastructure requires a **multi-authority accreditation** due to the presence of both NATO and multiple national security domains. The accreditation approach follows a phased strategy:

### 14.2 Accreditation Phases

**Phase 1: Facility Accreditation**
- NSM accredits the physical facility to LIST-X equivalent standard
- National inspections by DNK (CFCS/FE) and DEU (BSI/BMVg) to verify facility meets their national requirements
- TEMPEST certification by NSM-accredited TEMPEST authority
- Timeline: 6-12 months

**Phase 2: NATO Common Domain Accreditation**
- Norwegian DAA (delegated from NCIA SAB) accredits the NATO SECRET Common Domain
- Security accreditation documentation per AC/322-D/0048:
  - Security Concept of Operations (SecConOps)
  - System Security Policy (SSP)
  - Security Risk Assessment (SRA)
  - Security Accreditation Plan (SAP)
  - System Security Accreditation Statement (SSAS)
- NATO Cyber Security Centre (NCSC) may conduct vulnerability assessment
- Timeline: 6-9 months (parallel with Phase 1 where possible)

**Phase 3: National Enclave Accreditation (parallel tracks)**

| Enclave | Accreditation Authority | Documentation | Timeline |
|---|---|---|---|
| NOR | NSM | Sikkerhetsgodkjenning documentation per NSM Grunndokument | 4-6 months |
| DNK | CFCS | Sikkerhedskoncept + CFCS technical assessment | 6-9 months |
| DEU | BSI (coord. BMVg) | IT-Sicherheitskonzept + BSI Grundschutz assessment | 9-12 months |

**Phase 4: Cross-Domain Solution Accreditation**
- Joint accreditation by all four authorities (NCIA SAB + NSM + CFCS + BSI)
- Each CDS instance accredited individually
- Transfer Policy agreed and signed by all participating NSAs
- CDS testing: functional testing, penetration testing, content filtering validation
- Timeline: 6-12 months (may be the longest lead item)

**Phase 5: Integrated System Accreditation**
- End-to-end accreditation of the complete multinational system
- Joint accreditation statement signed by all authorities
- Operational readiness review
- Timeline: 3-6 months after all component accreditations

### 14.3 Accreditation Documentation Tree

```
Multinational System Accreditation
|
+-- Facility Security Accreditation (NSM)
|   +-- Physical Security Assessment
|   +-- TEMPEST Certificate
|   +-- Environmental Security Assessment
|
+-- NATO SECRET Common Domain (NCIA SAB / NOR DAA)
|   +-- SecConOps
|   +-- System Security Policy
|   +-- Security Risk Assessment
|   +-- Residual Risk Statement
|   +-- Vulnerability Assessment Report
|
+-- NOR National Enclave (NSM)
|   +-- Sikkerhetsdokumentasjon
|   +-- Risikovurdering
|   +-- Sikkerhetsgodkjenning
|
+-- DNK National Enclave (CFCS)
|   +-- Sikkerhedskoncept
|   +-- Risikovurdering
|   +-- Akkrediteringserklaring
|
+-- DEU National Enclave (BSI)
|   +-- IT-Sicherheitskonzept
|   +-- Risikoanalyse (BSI 200-3)
|   +-- Freigabeerklarung
|
+-- CDS-NOR (Joint: NCIA SAB + NSM)
+-- CDS-DNK (Joint: NCIA SAB + CFCS)
+-- CDS-DEU (Joint: NCIA SAB + BSI)
|   +-- Transfer Policy Document
|   +-- Content Filtering Rules
|   +-- Penetration Test Report
|   +-- Functional Test Report
|
+-- Integrated System Accreditation Statement (Joint all authorities)
```

### 14.4 Accreditation Timeline (Estimated)

```
Month:  0  3  6  9  12  15  18  21  24
        |--|--|--|--|---|---|---|---|---|
Phase 1 [Facility===============]
Phase 2    [NATO Common=========]
Phase 3a   [NOR Enclave====]
Phase 3b   [DNK Enclave=========]
Phase 3c   [DEU Enclave=============]
Phase 4          [CDS Accreditation==========]
Phase 5                              [Integrated====]
IOC                                              *IOC*
```

**Estimated time to Initial Operating Capability (IOC): 21-27 months**

---

## 15. Operational Governance

### 15.1 Governance Structure

```
+---------------------------------------------------+
|            GOVERNANCE STRUCTURE                     |
+---------------------------------------------------+
|                                                    |
|  +----------------------------------------------+ |
|  | Multinational Project Steering Committee      | |
|  | - NOR/DNK/DEU MOD representatives             | |
|  | - Meets: Quarterly                            | |
|  | - Decides: strategic direction, budget, scope | |
|  +----------------------------------------------+ |
|                                                    |
|  +----------------------------------------------+ |
|  | Security Accreditation Board (SAB)            | |
|  | - Representatives from all four SAAs          | |
|  | - Meets: Monthly during accreditation;        | |
|  |   quarterly during operations                 | |
|  | - Decides: accreditation status, risk         | |
|  |   acceptance, policy changes                  | |
|  +----------------------------------------------+ |
|                                                    |
|  +----------------------------------------------+ |
|  | Configuration Control Board (CCB)             | |
|  | - Technical leads from all three nations      | |
|  | - Meets: Biweekly                             | |
|  | - Decides: change requests, patching,         | |
|  |   configuration changes                       | |
|  +----------------------------------------------+ |
|                                                    |
|  +----------------------------------------------+ |
|  | Operational Management Team (OMT)             | |
|  | - Day-to-day operations management            | |
|  | - Led by host nation (NOR)                    | |
|  | - Includes national system administrators     | |
|  +----------------------------------------------+ |
|                                                    |
+---------------------------------------------------+
```

### 15.2 Roles and Responsibilities

| Role | Nationality | Responsibilities |
|---|---|---|
| **Infrastructure Security Officer (InfraSecO)** | NOR (host nation) | Overall facility security; liaison with NSM; physical security; TEMPEST |
| **NATO Common Domain ISSM** | Multinational (rotating or agreed) | Information system security management for NATO Common Domain |
| **NOR National Security Officer** | NOR | NOR enclave security; NOR clearance verification; NOR data release authority |
| **DNK National Security Officer** | DNK | DNK enclave security; DNK clearance verification; DNK data release authority |
| **DEU National Security Officer** | DEU | DEU enclave security; DEU clearance verification; DEU data release authority |
| **CDS Administrator** | Multinational (dual-person from different nations) | CDS configuration, monitoring; requires multi-national oversight |
| **System Administrators (per domain)** | Nationality matching domain | OS, network, application administration within their domain |
| **SOC Analysts** | Multinational | Security monitoring and incident detection |
| **Crypto Custodian** | Per domain (national for national domains) | Cryptographic key management, crypto device handling |

### 15.3 Change Management

All changes to the infrastructure must follow a formal change management process:

1. **Change Request (CR)** submitted to CCB
2. **Security Impact Assessment** conducted by ISSM and affected national security officers
3. **CCB Review and Approval** (consensus required for shared infrastructure; national authority for national enclaves)
4. **SAB Notification** for security-relevant changes; SAB approval required for changes affecting accreditation boundary
5. **Implementation** during approved maintenance window
6. **Verification** post-implementation testing and security validation
7. **Documentation Update** all affected accreditation documentation updated

### 15.4 Patch Management

- **Critical security patches:** Applied within 72 hours after testing in staging environment (per domain)
- **Standard patches:** Monthly patch cycle, tested in staging, applied during maintenance window
- **CDS patches:** Require vendor support and SAB approval before deployment; extended testing cycle
- **National enclave patches:** Subject to national approval process in addition to CCB
- **Zero-day response:** Emergency patching procedure with expedited CCB approval; post-facto SAB notification

---

## 16. Risk Register

| Risk ID | Risk | Likelihood | Impact | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| R-001 | Accreditation delays due to multi-authority coordination | High | High | Early engagement with all SAAs; phased accreditation; pre-accreditation workshops | Medium |
| R-002 | German BSI requires hardware separation beyond logical separation | Medium | High | Design for physical cage separation from outset; maintain hardware partitioning option for DEU | Low |
| R-003 | CDS product not accredited by all three national authorities | Medium | Critical | Select CDS with existing NATO SECRET accreditation; engage all SAAs in product selection | Medium |
| R-004 | Personnel clearance verification delays for multinational team | High | Medium | Initiate clearance processes 12+ months before IOC; use existing bilateral clearance recognition | Medium |
| R-005 | National crypto interoperability issues | Medium | Medium | Use NATO-standard crypto for common domain; national crypto only for national domains | Low |
| R-006 | Divergent national security requirements create design conflicts | Medium | High | Establish joint security requirements document early; identify conflicts through SAB process | Medium |
| R-007 | Supply chain compromise of hardware | Low | Critical | Procure through NATO/national approved supply chains; hardware inspection on receipt; TPMS/secure boot | Low |
| R-008 | Insider threat from multinational personnel | Low | Critical | Need-to-know enforcement; PAM; session recording; multi-person integrity for critical functions; continuous vetting | Low |
| R-009 | TEMPEST emanation beyond facility boundary | Low | High | SDIP-27 Level A in server rooms; annual TEMPEST testing; zone control | Low |
| R-010 | Cross-domain data spill (classified data to wrong domain) | Medium | High | CDS content filtering; DLP; mandatory labeling; automated classification checking | Medium |
| R-011 | Loss of facility availability (natural disaster, physical attack) | Low | Critical | Business continuity plan; potential warm standby at alternate facility; national WAN fallback | Medium |
| R-012 | Vendor lock-in or vendor unable to support classified environment | Medium | Medium | Multi-vendor strategy where possible; maintain cleared vendor support contracts; escrow arrangements | Low |

---

## 17. Reference Architecture Diagrams

### 17.1 High-Level Logical Architecture

```
+====================================================================+
||                    NATO SECRET MULTINATIONAL                       ||
||                    SHARED INFRASTRUCTURE                           ||
+====================================================================+
|                                                                      |
|  +----------------------------+    +-----------------------------+   |
|  |   NATO SECRET COMMON       |    |   MANAGEMENT DOMAIN         |   |
|  |   DOMAIN                   |    |   (Out-of-Band)             |   |
|  |                            |    |                             |   |
|  | +--------+ +--------+     |    | +----------+ +----------+   |   |
|  | | App    | | Collab | ... |    | | vCenter  | | SIEM     |   |   |
|  | | Servers| | Tools  |     |    | | /Mgmt    | | Collector|   |   |
|  | +--------+ +--------+     |    | +----------+ +----------+   |   |
|  | +--------+ +--------+     |    | +----------+ +----------+   |   |
|  | | DB     | | File   |     |    | | Patch    | | Monitoring|  |   |
|  | | Cluster| | Share  |     |    | | Server   | | (Nagios)  |  |   |
|  | +--------+ +--------+     |    | +----------+ +----------+   |   |
|  | +------------------------+|    +-----------------------------+   |
|  | | NATO SIEM              ||                                      |
|  | +------------------------+|                                      |
|  +--------+---+---------+----+                                      |
|           |   |         |                                           |
|      +----+   |    +----+                                           |
|      |   CDS-NOR   |  CDS-DNK   CDS-DEU                            |
|      |        |    |    |         |                                  |
|  +---+------+ | +--+---+---+ +---+--------+                        |
|  | NOR      | | | DNK      | | DEU        |                        |
|  | NATIONAL | | | NATIONAL | | NATIONAL   |                        |
|  | ENCLAVE  | | | ENCLAVE  | | ENCLAVE    |                        |
|  |          | | |          | |            |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  | | Apps | | | | | Apps | | | | Apps |   |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  | | Data | | | | | Data | | | | Data |   |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  | | SIEM | | | | | SIEM | | | | SIEM |   |                        |
|  | +------+ | | | +------+ | | +------+   |                        |
|  +----+-----+ | +----+-----+ +----+-------+                        |
|       |       |      |             |                                |
|  [NOR Crypto] | [DNK Crypto] [DEU Crypto]                          |
|       |       |      |             |                                |
|  NOR MilNet   | DNK MilNet   DEU MilNet                            |
|            [NATO Crypto]                                            |
|               |                                                     |
|          NATO WAN (NGCS/BICES)                                      |
+=====================================================================+
```

### 17.2 Physical Rack Layout (Simplified)

```
SERVER ROOM (Zone 3) - Indicative Layout
+================================================================+
|                                                                  |
|  +==============+  +==============+  +==============+           |
|  | NOR CAGE     |  | DNK CAGE     |  | DEU CAGE     |           |
|  | (Zone 3a)    |  | (Zone 3a)    |  | (Zone 3a)    |           |
|  |              |  |              |  |              |           |
|  | Rack 1: NOR  |  | Rack 1: DNK  |  | Rack 1: DEU  |           |
|  | Compute+Stor |  | Compute+Stor |  | Compute+Stor |           |
|  |              |  |              |  |              |           |
|  | Rack 2: NOR  |  | Rack 2: DNK  |  | Rack 2: DEU  |           |
|  | Network+Cryp |  | Network+Cryp |  | Network+Cryp |           |
|  +==============+  +==============+  +==============+           |
|                                                                  |
|  +==================================================+           |
|  | NATO COMMON CAGE (Zone 3)                         |           |
|  |                                                    |           |
|  | Rack 1-2: NATO Compute Nodes                       |           |
|  | Rack 3:   NATO Storage (Primary)                   |           |
|  | Rack 4:   NATO Storage (Secondary)                 |           |
|  | Rack 5:   NATO Network (Core SW, FW, LB)           |           |
|  | Rack 6:   CDS Cluster (CDS-NOR, CDS-DNK, CDS-DEU) |           |
|  | Rack 7:   NATO Crypto + HSM                        |           |
|  | Rack 8:   NATO SIEM + Backup                       |           |
|  +==================================================+           |
|                                                                  |
|  +=======================+                                       |
|  | MANAGEMENT CAGE        |                                       |
|  | (OOB Network)          |                                       |
|  | Rack 1: Mgmt servers   |                                       |
|  | Rack 2: Monitoring     |                                       |
|  +=======================+                                       |
|                                                                  |
+================================================================+
```

### 17.3 Data Flow: National-to-NATO Release Process

```
Step 1: Originator creates document in National Enclave
        [NOR HEMMELIG document in NOR Enclave]
                        |
Step 2: Originator marks for release
        [Apply STANAG 4774 label: NATO SECRET REL NOR/DNK/DEU]
        [Digital signature applied]
                        |
Step 3: Release authority reviews (if manual release required)
        [NOR National Security Officer approves release]
                        |
Step 4: Document submitted to CDS-NOR outbound queue
        [CDS ingests document]
                        |
Step 5: CDS-NOR processes document
        [a] Verify digital signature
        [b] Validate classification label
        [c] Content inspection (dirty word check, format validation)
        [d] Malware scan (multi-engine)
        [e] Policy evaluation (NOR Transfer Policy)
        [f] Audit log entry created
                        |
        [PASS]          |           [FAIL]
          |             |              |
Step 6a: Document      |    Step 6b: Document quarantined
released to NATO       |    Alert to NOR Security Officer
Common Domain          |    Audit log entry (rejection reason)
          |
Step 7: Document available in NATO SECRET Common Domain
        [Accessible to all NATO SECRET cleared project personnel]
```

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| BSI | Bundesamt fur Sicherheit in der Informationstechnik (German Federal Office for Information Security) |
| CFCS | Center for Cybersikkerhed (Danish Centre for Cyber Security) |
| CDS | Cross-Domain Solution |
| DAA | Designated Accreditation Authority |
| DLP | Data Loss Prevention |
| FE | Forsvarets Efterretningstjeneste (Danish Defence Intelligence Service) |
| HSM | Hardware Security Module |
| ISSM | Information System Security Manager |
| LIST-X | NATO term for industrial facility security clearance (used in various national equivalents) |
| MAC | Mandatory Access Control |
| MIRT | Multinational Incident Response Team |
| NCIRC | NATO Computer Incident Response Capability |
| NCSC | NATO Cyber Security Centre |
| NGCS | NATO General Communications System |
| NSA | National Security Authority (not to be confused with US NSA) |
| NSM | Nasjonal sikkerhetsmyndighet (Norwegian National Security Authority) |
| OOB | Out-of-Band |
| PAM | Privileged Access Management |
| PET | Politiets Efterretningstjeneste (Danish Security and Intelligence Service) |
| SAA | Security Accreditation Authority |
| SAB | Security Accreditation Board |
| SIEM | Security Information and Event Management |
| SOC | Security Operations Center |
| STANAG | Standardization Agreement (NATO) |
| TEMPEST | Telecommunications Electronics Material Protected from Emanating Spurious Transmissions |

## Appendix B: Key Assumptions

1. All three nations have existing bilateral security agreements that permit exchange of classified information at SECRET level.
2. The Norwegian facility can be accredited to handle all three nations' SECRET-level information concurrently.
3. A suitable CDS product exists that can achieve accreditation from all four accreditation authorities (NCIA SAB, NSM, CFCS, BSI) within the project timeline.
4. Sufficient cleared personnel from all three nations are available for the operational team.
5. National WAN connectivity (NOR MilNet, DNK FIIN, DEU BWI) can be extended to the Norwegian facility via national crypto devices.
6. NATO WAN connectivity (NGCS or BICES) is available at or near the facility location.
7. Budget is available for the significant hardware investment required for physical separation of four domains plus management infrastructure.
8. Germany will accept logical separation (within a physical cage) for DEU GEHEIM data, subject to BSI risk assessment; if hardware partitioning is required, the architecture supports this with additional cost.

## Appendix C: Open Items and Decision Points

| ID | Decision | Options | Deadline | Decision Maker |
|---|---|---|---|---|
| D-001 | CDS product selection | HIGHWALL / Nexor Sentinel / ISSE Guard | Month 3 | Joint SAB + CCB |
| D-002 | Hypervisor platform selection | VMware / Nutanix / KVM | Month 3 | CCB |
| D-003 | German hardware separation requirement | Logical only / Physical partition | Month 6 | BSI |
| D-004 | NATO WAN connectivity method | NGCS / BICES / Other | Month 3 | NCIA + NOR MOD |
| D-005 | Identity federation approach | AD trusts / SAML federation / Independent | Month 6 | CCB + SAB |
| D-006 | SOC operating model | Host nation led / Rotating lead / Contracted | Month 9 | Steering Committee |
| D-007 | Backup offsite strategy | Per-nation offsite / Shared offsite / Online only | Month 6 | CCB + national SecOs |
| D-008 | Business continuity / DR facility | No DR / Warm standby / Cold standby | Month 6 | Steering Committee |

---

*End of document.*
