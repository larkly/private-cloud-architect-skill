# NATO SECRET Multinational Shared Infrastructure Architecture

## Multinational Defense Project: Norway, Denmark, Germany
### Classification: NATO SECRET / RELEASABLE TO NOR, DNK, DEU

---

**Document Version:** 1.0
**Date:** 2026-03-20
**Classification Marking:** This document itself is UNCLASSIFIED — it describes the architecture of a system intended to process NATO SECRET material.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Governance and Legal Framework](#2-governance-and-legal-framework)
3. [National Security Frameworks](#3-national-security-frameworks)
4. [Facility and Physical Security](#4-facility-and-physical-security)
5. [Network Architecture](#5-network-architecture)
6. [Enclave Design and Separation Model](#6-enclave-design-and-separation-model)
7. [Cross-Domain Solution Architecture](#7-cross-domain-solution-architecture)
8. [Identity, Access Management and Personnel Security](#8-identity-access-management-and-personnel-security)
9. [Data Classification and Labeling](#9-data-classification-and-labeling)
10. [Cryptographic Architecture](#10-cryptographic-architecture)
11. [Operational Security and Monitoring](#11-operational-security-and-monitoring)
12. [Accreditation Strategy](#12-accreditation-strategy)
13. [Risk Assessment Summary](#13-risk-assessment-summary)
14. [Deployment and Operations Model](#14-deployment-and-operations-model)
15. [Appendices](#15-appendices)

---

## 1. Executive Summary

This document defines the reference architecture for a shared NATO SECRET infrastructure supporting a multinational defense project between Norway, Denmark, and Germany. The platform is physically hosted in a LIST-X equivalent facility in Norway and must process:

- **NATO SECRET** data releasable to the three participating nations
- **National SECRET** data from Norway (HEMMELIG), Denmark (HEMMELIGT), and Germany (GEHEIM)
- **Cross-domain shared data** that has been reviewed and released between national enclaves

The architecture implements a multi-enclave model with strict separation between national domains and a shared NATO domain, connected through accredited Cross-Domain Solutions (CDS). Each national enclave enforces the originating nation's security policy, while the shared NATO enclave operates under NATO security policy as the common baseline.

### Key Design Principles

1. **Sovereignty preservation** — Each nation retains full control over its nationally classified data, including release decisions.
2. **NATO policy as common denominator** — The shared domain operates under NATO Security Policy (C-M(2002)49 and successor documents), providing the interoperability baseline.
3. **Defense in depth** — Multiple independent security layers prevent single points of compromise.
4. **Least privilege and need-to-know** — Access is granted at the intersection of clearance level, national authorization, and project need-to-know.
5. **Accreditation-driven design** — Every design decision traces to an accreditation requirement from the relevant SAA/DAA.

---

## 2. Governance and Legal Framework

### 2.1 Applicable Agreements and Regulations

| Framework | Applicability | Authority |
|-----------|--------------|-----------|
| NATO Security Policy (C-M(2002)49-REV) | All NATO SECRET processing | NATO Office of Security (NOS) |
| AC/35 INFOSEC directives (including AC/35-D/2004) | CIS security for NATO systems | NATO Cyber Security Centre (NCSC) |
| ATOMAL regulations (if applicable) | Nuclear-related data handling | As per specific program requirements |
| Norwegian Security Act (Sikkerhetsloven, 2018) | Norwegian national classified data, facility operations | Nasjonal Sikkerhetsmyndighet (NSM) |
| Danish Security Circular (Sikkerhedscirkulaeret) | Danish national classified data | Centre for Cybersikkerhed (CFCS) / Forsvarets Efterretningstjeneste (FE) |
| German Federal Security Screening Act (SUA) & VSA (Verschlusssachenanweisung) | German national classified data | Bundesamt fuer Sicherheit in der Informationstechnik (BSI) / Bundesministerium fuer Verteidigung (BMVg) |
| Bilateral/Trilateral Security Agreements | Cross-national data handling | Respective national NSAs |
| Programme MOU/MOA | Project-specific security requirements | Programme Security Officer (PSO) |
| Norway-NATO SOFA / Host Nation Support Agreements | Hosting of multinational classified systems | Norwegian MOD |

### 2.2 Governance Structure

```
+-----------------------------------------------+
|          Programme Steering Committee          |
|        (NOR MOD, DNK MOD, DEU BMVg)           |
+-----------------------------------------------+
                      |
         +------------+------------+
         |                         |
+--------v--------+    +-----------v-----------+
| Programme Mgmt  |    | Programme Security    |
| Office (PMO)    |    | Officer (PSO)         |
+-----------------+    +----------+------------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
    +---------v------+  +---------v------+  +---------v------+
    | NOR National   |  | DNK National   |  | DEU National   |
    | Security       |  | Security       |  | Security       |
    | Representative |  | Representative |  | Representative |
    +----------------+  +----------------+  +----------------+
```

**Programme Security Officer (PSO):** Responsible for the overall security governance of the programme, including the Programme Security Instruction (PSI). Appointed by consensus of the three nations.

**National Security Representatives (NSR):** Each nation appoints an NSR who ensures compliance with national regulations and serves as the liaison to their National Security Authority (NSA) and Security Accreditation Authority (SAA).

**Security Accreditation Board (SAB):** A joint board composed of representatives from NSM (Norway), CFCS (Denmark), and BSI (Germany), plus a NATO representative. The SAB is the collective accreditation authority for the shared platform. Operates under the principle of mutual recognition where possible, with national veto on matters affecting national classified information.

### 2.3 Security Accreditation Authorities

| Domain | SAA | Notes |
|--------|-----|-------|
| NATO SECRET shared enclave | SAB (joint) with NOS oversight | Primary accreditation via NSM as host nation, with mutual recognition |
| Norwegian national enclave | NSM | Norwegian requirements paramount |
| Danish national enclave | CFCS | Danish requirements paramount |
| German national enclave | BSI / BMVg | German requirements paramount, VSA compliance |
| Cross-Domain Solutions | SAB (joint) | All three nations plus NATO must approve |
| Physical facility | NSM | As host nation facility authority |

---

## 3. National Security Frameworks

### 3.1 Norway (Nasjonal Sikkerhetsmyndighet — NSM)

**Legal basis:** Sikkerhetsloven (Security Act) of 2018, with associated regulations (Virksomhetsikkerhetsforskriften, Klareringsforskriften, Sikkerhetsgraderte anskaffelser).

**Classification levels:** STRENGT HEMMELIG (TOP SECRET), HEMMELIG (SECRET), KONFIDENSIELT (CONFIDENTIAL), BEGRENSET (RESTRICTED).

**Key requirements for IT systems processing HEMMELIG:**
- Systems must be accredited by NSM before processing classified data.
- Follows NSM's "Rammeverk for styring av informasjonssikkerhet" and specific ICT security guidelines.
- TEMPEST requirements per SDIP-27 NATO standards — Zone 1 or Zone 2 depending on risk assessment.
- Personnel must hold valid Norwegian security clearance at appropriate level.
- Continuous monitoring and incident reporting to NSM.
- Norwegian encryption requirements: NSM-approved cryptographic products for national data.

**Facility requirements:** The LIST-X equivalent facility in Norway operates under NSM oversight. Norway uses the term "Sikkerhetsgradert anskaffelse" for classified contracts and the facility must meet requirements per the Security Act's facility security regulations.

### 3.2 Denmark (Centre for Cybersikkerhed — CFCS)

**Legal basis:** Sikkerhedscirkulaeret (Security Circular), supplemented by specific IT security directives from CFCS and Forsvarets Efterretningstjeneste (FE).

**Classification levels:** YDERST HEMMELIGT (TOP SECRET), HEMMELIGT (SECRET), FORTROLIGT (CONFIDENTIAL), TIL TJENESTEBRUG (RESTRICTED).

**Key requirements for IT systems processing HEMMELIGT:**
- CFCS accreditation required, based on risk assessment methodology.
- Denmark aligns closely with NATO INFOSEC standards but has specific national additions.
- Danish-approved crypto for nationally classified data.
- Personnel must hold Danish security clearance (granted by PET/FE for defense personnel).
- TEMPEST requirements per national interpretation of SDIP-27.
- Specific requirements for remote administration and monitoring of Danish data when processed outside Denmark — requires bilateral agreement framework.

**Cross-border considerations:** Danish nationally classified data processed in Norway requires a bilateral security arrangement. Denmark historically has well-established bilateral security cooperation with Norway through Nordic defense cooperation frameworks (NORDEFCO).

### 3.3 Germany (Bundesamt fuer Sicherheit in der Informationstechnik — BSI)

**Legal basis:** Verschlusssachenanweisung (VSA — Classified Information Instruction), SUA (Sicherheitsueberprufungsgesetz), BSI-Gesetz.

**Classification levels:** STRENG GEHEIM (TOP SECRET), GEHEIM (SECRET), VS-VERTRAULICH (CONFIDENTIAL), VS-NUR FUER DEN DIENSTGEBRAUCH (RESTRICTED).

**Key requirements for IT systems processing GEHEIM:**
- BSI accreditation and compliance with BSI IT-Grundschutz and specific VS-IT guidelines.
- Germany has particularly stringent requirements for GEHEIM processing, including:
  - Hardware and software evaluation requirements (BSI-certified or evaluated products preferred).
  - Specific zoning and emanation security (TEMPEST) requirements — Germany often requires Zone 1 for GEHEIM.
  - German-approved cryptographic equipment for national classified data (BSI-approved Type A crypto).
  - Detailed system security concept (Sicherheitskonzept) required.
- Personnel must hold German security clearance (Ue2 for GEHEIM, issued by BMWK MAD or relevant authority).
- Germany has strict requirements regarding the location of nationally classified data processing and typically requires strong guarantees when such data is processed on foreign territory.

**Cross-border considerations:** German GEHEIM data processed outside Germany is a sensitive matter. Requires:
- Bilateral security agreement between Germany and Norway covering IT systems.
- German SAA (BSI/BMVg) must explicitly approve the system and facility.
- Germany may require German-controlled cryptographic equipment and may insist on German personnel having physical access control to the German enclave.
- VSA requirements for contractor facilities apply even when the facility is in Norway.

### 3.4 Comparative Matrix

| Requirement | Norway (NSM) | Denmark (CFCS) | Germany (BSI) | NATO (NOS/NCSC) |
|------------|-------------|----------------|---------------|-----------------|
| Classification label at SECRET | HEMMELIG | HEMMELIGT | GEHEIM | NATO SECRET |
| Crypto for national data | NSM-approved | CFCS-approved | BSI Type A | NATO-approved (NCOIC catalogue) |
| TEMPEST zone (typical for SECRET) | Zone 1-2 (risk-based) | Zone 1-2 (risk-based) | Zone 1 (typically required) | Zone 1-2 (per SDIP-27) |
| System accreditation approach | Risk-based, NSM framework | Risk-based, CFCS framework | BSI IT-Grundschutz + VS-specific | AC/35 directives, NIST-like RA |
| Cross-border data processing | Bilateral agreement | Bilateral agreement | Bilateral agreement + explicit BSI approval | NATO nations hosting per C-M(2002)49 |
| Personnel clearance recognition | National clearance + NATO clearance | National clearance + NATO clearance | National clearance + NATO clearance | NATO SECRET PSC via national process |

---

## 4. Facility and Physical Security

### 4.1 Facility Classification

The facility operates as a LIST-X equivalent in Norway. Under Norwegian regulations, this is a contractor facility with a Facility Security Clearance (FSC) at the HEMMELIG level, overseen by NSM.

**Physical security zones (per NATO and Norwegian standards):**

```
+================================================================+
|  ZONE 0 — Public / Uncontrolled Area                           |
|  (Outside facility perimeter)                                  |
+================================================================+
|  ZONE 1 — Controlled Area (Reception, badge check)             |
|  - Visitor escort required                                     |
|  - CCTV, access logging                                        |
+================================================================+
|  ZONE 2 — Restricted Area (Office spaces, cleared personnel)   |
|  - Badge + PIN access control                                  |
|  - NATO RESTRICTED / national RESTRICTED processing            |
+================================================================+
|  ZONE 3 — Security Area / Class II (Server rooms, operations)  |
|  - Two-person integrity where required                         |
|  - Badge + biometric access control                            |
|  - IDS (Intrusion Detection System) alarmed 24/7               |
|  - NATO SECRET processing                                      |
|  +----------------------------------------------------------+  |
|  |  ZONE 3+ — National Enclave Rooms (physically separated) |  |
|  |  - Additional national access controls                    |  |
|  |  - National lock/seal requirements                        |  |
|  |  - Only nationally cleared personnel with need-to-know    |  |
|  +----------------------------------------------------------+  |
+================================================================+
```

### 4.2 Enclave Physical Separation

Each national enclave has dedicated physical space within the Zone 3 area:

| Enclave | Physical Space | Access Control | Notes |
|---------|---------------|----------------|-------|
| NATO SECRET Shared | Primary server room | All three nations' cleared personnel with NATO SECRET PSC | Shared operational space |
| NOR National | Dedicated cage/room within server area | Norwegian cleared personnel only | NSM key/lock requirements |
| DNK National | Dedicated cage/room within server area | Danish cleared personnel only | CFCS requirements |
| DEU National | Dedicated cage/room within server area | German cleared personnel only | German lock (BSI-approved), German personnel escort requirement |
| CDS Room | Isolated room with controlled cable routing | Joint access with two-person rule (multi-national) | Highest physical security |

### 4.3 TEMPEST and Emanation Security

Given Germany's typical Zone 1 requirement for GEHEIM:

- **The entire Zone 3 area is designed to SDIP-27 Zone 1 standards** (most restrictive, satisfies all three nations).
- Alternatively, if cost-prohibitive: the German national enclave and CDS room are Zone 1; the remaining enclaves are Zone 2 with documented risk acceptance from Norway and Denmark.
- TEMPEST testing and certification performed by NSM TEMPEST authority (or BSI-recognized TEMPEST testing laboratory for the German enclave).
- Cable routing follows SDIP-29 standards — classified and unclassified cables are physically separated with prescribed minimum distances.

### 4.4 Power and Environmental

- Dedicated UPS and generator for the classified processing areas.
- Redundant cooling (N+1) for server rooms.
- Fire suppression: gas-based (inert gas) system to protect equipment without water damage.
- Environmental monitoring (temperature, humidity, water ingress) with alerting.
- Power filtering / isolation as required by TEMPEST Zone 1 specifications.

---

## 5. Network Architecture

### 5.1 High-Level Network Topology

```
                         NATO WAN
                    (NGCS / BICES / Dedicated Link)
                             |
                      +------v------+
                      | WAN Crypto  |
                      | (NATO-appr) |
                      +------+------+
                             |
                    +--------v--------+
                    |   NATO SECRET   |
                    |   Shared Core   |
                    |   Network       |
                    +---+----+----+---+
                        |    |    |
           +------------+    |    +------------+
           |                 |                 |
    +------v------+   +------v------+   +------v------+
    | CDS         |   | CDS         |   | CDS         |
    | NOR <-> NATO|   | DNK <-> NATO|   | DEU <-> NATO|
    +------+------+   +------+------+   +------+------+
           |                 |                 |
    +------v------+   +------v------+   +------v------+
    |  NOR Natl   |   |  DNK Natl   |   |  DEU Natl   |
    |  Enclave    |   |  Enclave    |   |  Enclave    |
    |  Network    |   |  Network    |   |  Network    |
    +------+------+   +------+------+   +------+------+
           |                 |                 |
    +------v------+   +------v------+   +------v------+
    | NOR Natl    |   | DNK Natl    |   | DEU Natl    |
    | WAN Crypto  |   | WAN Crypto  |   | WAN Crypto  |
    +------+------+   +------+------+   +------+------+
           |                 |                 |
     NOR National      DNK National      DEU National
     Network           Network           Network
     (FISBasis/NbF)    (DKCIS)           (BWI/FueInfoSysKdo)
```

### 5.2 Network Segmentation

**Five physically and logically separated networks:**

1. **NATO SECRET Shared Network** — The common working domain for all three nations' personnel working on the multinational project. Only NATO-classified data at NATO SECRET and below.

2. **Norwegian National Network (HEMMELIG)** — Processes Norwegian nationally classified data. Connected to Norwegian national defense networks via NSM-approved crypto.

3. **Danish National Network (HEMMELIGT)** — Processes Danish nationally classified data. Connected to Danish national defense networks via CFCS-approved crypto.

4. **German National Network (GEHEIM)** — Processes German nationally classified data. Connected to German national defense networks via BSI-approved crypto.

5. **Management/OOB Network** — Out-of-band management network for infrastructure components. Classified at system-high (NATO SECRET). Physically separated cabling. Dedicated management terminals in the operations room.

**Additionally:**

6. **Unclassified Administrative Network** — For facility management, building systems, non-classified email. Completely air-gapped from all classified networks. Separate physical infrastructure.

### 5.3 Network Separation Enforcement

- **Physical separation (air gap)** between each national enclave network. No routable path between national enclaves exists without traversing a CDS.
- **No direct national-to-national connections.** All cross-domain data flow is: National Enclave -> CDS -> NATO Shared -> CDS -> Other National Enclave. This ensures NATO policy is the common gatekeeping layer.
- Each network uses dedicated switches, routers, firewalls, and cabling.
- Cable colors per network (industry best practice for classified environments):

| Network | Cable Color | Connector Label |
|---------|-------------|-----------------|
| NATO SECRET Shared | Red | NATO-S |
| NOR National | Blue | NOR-H |
| DNK National | Green | DNK-H |
| DEU National | Yellow | DEU-G |
| Management/OOB | Orange | MGMT |
| Unclassified | White | UNCLAS |

### 5.4 Firewall and Boundary Protection

Each network boundary employs defense-in-depth:

```
[Network A] -> [Firewall A-side] -> [CDS Appliance] -> [Firewall B-side] -> [Network B]
```

- Firewalls are from different vendors on each side of the CDS (product diversity).
- All firewalls operate in explicit deny-all / allow-by-exception mode.
- Firewall rules are reviewed and approved by the SAB.
- Stateful inspection at minimum; application-layer filtering where technically feasible within classified constraints.
- All boundary crossings are logged to the centralized SIEM (within the management network).

### 5.5 WAN Connectivity

| Connection | Crypto Device | Approval | Circuit |
|------------|--------------|----------|---------|
| NATO SECRET WAN | NATO-approved (e.g., Thales Datacryptor, Rohde & Schwarz SITLine) | NATO NCOIC catalogue | Dedicated leased line or MPLS VPN (black transport) |
| NOR National WAN | NSM-approved crypto | NSM | Connection to FISBasis or relevant Norwegian classified network |
| DNK National WAN | CFCS-approved crypto | CFCS | Connection to Danish defense network |
| DEU National WAN | BSI-approved crypto (e.g., SINA, Genua genuscreen) | BSI | Connection to BWI/FueInfoSysKdo network |

---

## 6. Enclave Design and Separation Model

### 6.1 Enclave Architecture Overview

Each enclave is a self-contained computing environment with its own:
- Compute infrastructure (servers/hypervisors)
- Storage (SAN/NAS)
- Directory services
- Application services
- Backup infrastructure
- Security monitoring agents

### 6.2 NATO SECRET Shared Enclave

**Purpose:** Common workspace for the multinational project. All personnel from the three nations collaborate here on NATO-classified project data.

**Infrastructure:**
- Hypervisor cluster: Minimum 3-node cluster for HA (e.g., VMware vSphere with appropriate hardening, or evaluated alternative).
- Storage: Dedicated SAN with encryption at rest (NATO-approved or nationally accepted crypto module).
- Directory: Standalone Active Directory forest (or equivalent) — `nato-project.mission.local`.
- Collaboration services: Document management, project management tools, NATO-standard messaging (e.g., MMHS-compliant for formal messaging).
- Email: NATO SECRET-level email connected to NATO communication systems.

**Data policy:** Only NATO-classified data. No national caveats beyond REL NOR/DNK/DEU. Data that is nationally originated must be reviewed and formally released to NATO before entering this enclave.

### 6.3 Norwegian National Enclave

**Purpose:** Process Norwegian HEMMELIG data related to the project. Norwegian national-eyes-only workspace.

**Infrastructure:**
- Dedicated hypervisor cluster (minimum 2 nodes).
- Dedicated storage.
- Directory: Norwegian national AD forest or integration with Norwegian defense directory services via WAN.
- Norwegian national applications and tools as required.
- Managed by Norwegian-cleared personnel only.

**Data policy:** Norwegian HEMMELIG and below. No foreign national access. Data release to the NATO shared enclave requires explicit Norwegian release authority approval (typically the Norwegian NSR or designated release officer) and is performed via the CDS.

### 6.4 Danish National Enclave

**Purpose:** Process Danish HEMMELIGT data related to the project.

**Infrastructure:** Mirrors the Norwegian enclave pattern with Danish-specific requirements.
- Dedicated hypervisor cluster.
- Dedicated storage.
- Danish national directory and authentication.
- Managed by Danish-cleared personnel.

**Data policy:** Danish HEMMELIGT and below. Release to NATO shared enclave requires Danish release authority approval.

### 6.5 German National Enclave

**Purpose:** Process German GEHEIM data related to the project.

**Infrastructure:** Mirrors the pattern with German-specific additions:
- Dedicated hypervisor cluster.
- Dedicated storage with BSI-approved encryption at rest.
- German national directory services.
- **BSI-evaluated or BSI-certified components preferred** (Germany has stricter product evaluation requirements at GEHEIM).
- Managed exclusively by German-cleared personnel.
- **German-specific:** May require a separate physical safe/strong room within the German enclave area for German COMSEC material storage per VSA requirements.

**Data policy:** German GEHEIM and below. VS-NfD through GEHEIM. Release to NATO shared enclave requires German release authority approval. Germany may require formal "Freigabe" (release) process with documented authorization chain.

### 6.6 Compute and Virtualization Standards

**Hypervisor hardening (all enclaves):**
- Hypervisor selected from evaluated/accredited products where available.
- CIS Benchmark or DISA STIG hardening applied.
- No internet-connected management; management only via OOB network.
- Virtual machine isolation enforced — VMs from different enclaves never share a hypervisor host.
- Dedicated physical hosts per enclave (no shared hypervisor between enclaves).
- SR-IOV or PCI passthrough for network interfaces to reduce hypervisor attack surface for network traffic.
- Regular patching with tested patches via offline/controlled update mechanism.

**Storage:**
- Encryption at rest on all storage arrays (AES-256 minimum).
- Crypto modules: FIPS 140-2/3 Level 2+ or equivalent national approval.
- Dedicated storage arrays per enclave — no shared storage fabric between enclaves.
- Secure erase capability for decommissioned media per NATO CM(2002)49 Annex IV or national equivalent.

---

## 7. Cross-Domain Solution Architecture

### 7.1 CDS Requirement

The Cross-Domain Solutions are the most security-critical components in this architecture. They enable controlled data transfer between national enclaves and the NATO shared enclave while preventing unauthorized data leakage.

**Three CDS instances are required:**
1. NOR National <-> NATO Shared
2. DNK National <-> NATO Shared
3. DEU National <-> NATO Shared

**No direct CDS between national enclaves.** If Norway needs to share data with Germany, the data flow is: NOR -> CDS -> NATO Shared -> CDS -> DEU. This enforces NATO policy as the arbitration layer and ensures proper release markings.

### 7.2 CDS Product Selection Considerations

Candidate CDS products (subject to accreditation by all three national SAAs and NATO):

| Product | Vendor | Notes |
|---------|--------|-------|
| Advenica SecuriCDS | Advenica (Sweden) | EU/NATO-focused, Common Criteria evaluated |
| Thales Elips | Thales (France) | NATO-accredited cross-domain solution |
| NEXOR Sentinel | NEXOR (UK) | NATO-standard MMHS guard, widely deployed |
| Deep-Secure (Forcepoint) | Forcepoint | Content inspection guard |
| National CDS solutions | Various | Germany may require BSI-specific CDS |

**Critical:** Germany may require that the CDS interfacing with the German enclave is BSI-approved. This could mean deploying a German-specific CDS product for the DEU <-> NATO link while using a different product for the other two links. This is architecturally acceptable and may be necessary for accreditation.

### 7.3 CDS Functional Architecture

Each CDS instance implements the following pipeline:

```
Source Enclave                                          Destination Enclave
     |                                                        ^
     v                                                        |
+----+--------+    +----------+    +-----------+    +---------+----+
| Export       |    | Content  |    | Security  |    | Import       |
| Review &     |--->| Inspect  |--->| Label     |--->| Validation   |
| Release      |    | & Filter |    | Transform |    | & Delivery   |
| Queue        |    |          |    |           |    |              |
+-------------+    +----------+    +-----------+    +--------------+
      ^                  |               |                  |
      |                  v               v                  v
  Human release    Malware scan     Label check        Audit log
  officer review   Format valid.    Policy check       Integrity check
  (for manual)     Content filter   Releasability      Delivery confirm
```

### 7.4 CDS Operational Modes

**Mode 1: Automated transfer with policy enforcement (for pre-approved data types)**
- Structured data with machine-readable security labels (e.g., XML with NATO STANAG 4774/4778 confidentiality metadata).
- CDS validates labels, checks releasability against policy rules, performs content inspection.
- Suitable for routine data exchanges (e.g., formatted reports, database replication of shared datasets).

**Mode 2: Human-reviewed transfer (for ad-hoc or unstructured data)**
- User submits data for cross-domain transfer.
- Data is placed in a review queue.
- Designated release officer (from the originating nation) reviews and approves.
- CDS performs content inspection and transfers upon approval.
- Suitable for documents, presentations, ad-hoc files.

**Mode 3: One-way data diode (for specific high-assurance requirements)**
- Hardware-enforced one-way transfer (e.g., from high to low, or for specific audit log forwarding).
- May be required by Germany for certain data flows into the German enclave.
- Products: Advenica Data Diode, Owl Cyber Defense, Waterfall Security.

### 7.5 CDS Security Policy Rules

Each CDS enforces a policy rule set approved by the SAB:

```
RULE: NOR_to_NATO
  Source: NOR National Enclave
  Destination: NATO Shared Enclave
  Conditions:
    - Data must bear release marking "REL NATO" or "REL NOR/DNK/DEU"
    - Release officer approval (Mode 2) or valid auto-release tag (Mode 1)
    - Content inspection passed (no malware, no forbidden content patterns)
    - Security label present and valid per STANAG 4774
  Action: TRANSFER with audit log entry

RULE: NATO_to_DEU
  Source: NATO Shared Enclave
  Destination: DEU National Enclave
  Conditions:
    - Data must be marked NATO SECRET or below
    - German personnel authorization verified
    - Content inspection passed
    - No national caveats from other nations that exclude Germany
  Action: TRANSFER with audit log entry and relabeling to include GEHEIM marking

RULE: DEFAULT
  Action: DENY with alert
```

### 7.6 CDS Accreditation

CDS accreditation is the longest-lead and highest-risk item in the programme:

- Each CDS must be accredited by all parties whose data it handles.
- The SAB provides joint accreditation, but each national SAA retains veto authority.
- **Timeline estimate:** 12-18 months for CDS accreditation from initial documentation to Authority to Operate (ATO).
- **Risk mitigation:** Begin CDS accreditation process early; use "sneakernet" (manual media transfer with two-person review) as interim cross-domain mechanism while CDS accreditation is pending.

---

## 8. Identity, Access Management and Personnel Security

### 8.1 Personnel Security Clearance Requirements

| Enclave | Required Clearance | Issuing Authority | Additional Requirements |
|---------|--------------------|-------------------|------------------------|
| NATO Shared | NATO SECRET PSC | National NSA (via national clearance) | Project need-to-know briefing |
| NOR National | Norwegian HEMMELIG | NSM | Norwegian citizenship (typically) |
| DNK National | Danish HEMMELIGT | PET/FE | Danish citizenship (typically) |
| DEU National | German Ue2 (GEHEIM) | MAD/BMWK | German citizenship (typically) |
| CDS Operations | NATO SECRET PSC + national clearance | National + NATO | Additional CDS operator vetting |

**Multi-national personnel considerations:**
- Each person working on the project needs:
  1. A valid national security clearance at the appropriate level.
  2. A NATO Security Clearance Certificate (SCC) at NATO SECRET level (derived from national clearance).
  3. A project-specific need-to-know briefing and acknowledgment.
- Personnel access national enclaves ONLY for their own nation.
- All personnel access the NATO Shared enclave.

### 8.2 Identity Architecture

```
+--------------------+     +--------------------+     +--------------------+
| NOR National AD    |     | DNK National AD    |     | DEU National AD    |
| (nor-enclave.local)|     | (dnk-enclave.local)|     | (deu-enclave.local)|
+--------+-----------+     +---------+----------+     +---------+----------+
         |                           |                           |
         |    (NO trust relationship between national ADs)       |
         |                           |                           |
         |   +-------------------+   |                           |
         +-->| NATO Shared AD    |<--+                           |
             | (nato-proj.local) |<------------------------------+
             +---------+---------+
                       |
              Separate accounts per
              user in NATO Shared domain
              (not federated — discrete identity)
```

**Key design decisions:**
- **No AD trust relationships between enclaves.** Each enclave maintains an independent directory.
- Users have **separate accounts** in each enclave they are authorized to access (e.g., a Norwegian user has one account in the NOR enclave and a separate account in the NATO Shared enclave).
- This prevents credential-based lateral movement between enclaves.
- Account provisioning and deprovisioning follows a formal process managed by the PSO office.

### 8.3 Authentication

| Mechanism | Implementation | Notes |
|-----------|---------------|-------|
| Primary authentication | Smartcard / PKI certificate | National PKI cards (e.g., Norwegian BuyPass/Commfides, German PKI card) for national enclaves; dedicated project PKI card for NATO Shared |
| Multi-factor | Smartcard (something you have) + PIN (something you know) | Required for all SECRET-level access |
| Privileged access | Additional factor or separate admin account with enhanced logging | Admin accounts are separate from user accounts |
| NATO Shared enclave | Dedicated project smartcard or token | Issued by project PKI authority, linked to NATO SCC |

### 8.4 Authorization and Access Control

**Role-Based Access Control (RBAC)** is the baseline, augmented with **Attribute-Based Access Control (ABAC)** for fine-grained data access:

Attributes used in access decisions:
- **Clearance level** (from personnel security system)
- **Nationality** (determines enclave access)
- **Project role** (determines functional access within enclaves)
- **Need-to-know group** (determines access to specific project work packages)
- **Data sensitivity label** (matched against user attributes)

### 8.5 Privileged Access Management

- Dedicated Privileged Access Management (PAM) solution in each enclave.
- Administrative access via jump host / bastion host on the management network.
- All admin sessions recorded (keystroke logging and screen recording for critical systems).
- Time-limited admin sessions with explicit check-out/check-in.
- Two-person integrity for critical operations (e.g., CDS configuration changes, firewall rule changes, crypto key management).

---

## 9. Data Classification and Labeling

### 9.1 Classification Scheme

The project uses a unified labeling approach compliant with STANAG 4774 (Confidentiality Metadata Label Syntax) and STANAG 4778 (Metadata Binding):

```
NATO Classification + National Caveat + Releasability + Project Marking

Examples:
- NATO SECRET // REL NOR, DNK, DEU // PROJECT AURORA
- NATO SECRET // NOFORN [specific restrictions if applicable]
- HEMMELIG // NOR NATIONAL // PROJECT AURORA NATIONAL ANNEX
- GEHEIM // DEU NATIONAL // PROJECT AURORA NATIONAL ANNEX
- HEMMELIGT // DNK NATIONAL // PROJECT AURORA NATIONAL ANNEX
```

### 9.2 Data Domains and Labeling Policy

| Data Domain | Classification | Allowed Locations | Releasability |
|-------------|---------------|-------------------|---------------|
| Shared project data | NATO SECRET REL NOR/DNK/DEU | NATO Shared enclave only | All three nations |
| Norwegian national contributions | HEMMELIG | NOR enclave; NATO Shared (after release) | NOR only (unless released) |
| Danish national contributions | HEMMELIGT | DNK enclave; NATO Shared (after release) | DNK only (unless released) |
| German national contributions | GEHEIM | DEU enclave; NATO Shared (after release) | DEU only (unless released) |
| Programme management data | NATO SECRET REL NOR/DNK/DEU | NATO Shared enclave | All three nations |
| CDS audit logs | NATO SECRET | Management network + CDS archive | SAB members |

### 9.3 Metadata and Labeling Implementation

- All documents created in the system must bear a classification header and footer (enforced by document templates and DLP).
- Electronic files carry metadata labels in conformance with STANAG 4774.
- The document management system in the NATO Shared enclave enforces label selection at creation time.
- DLP (Data Loss Prevention) tools in each enclave scan for unlabeled or mislabeled data.
- The CDS validates labels as part of the transfer policy (see Section 7.5).

### 9.4 Data Lifecycle and Destruction

- Retention periods defined per project MOU and national regulations (typically the most restrictive national requirement applies).
- Data at rest must be encrypted (see Section 10).
- Data destruction follows NATO CM(2002)49 Annex IV:
  - Electronic media: Degaussing + physical destruction, or NSA/CSE-approved erasure for reusable media.
  - Germany may require BSI-approved destruction methods for GEHEIM media.
- Destruction events are logged and certificates of destruction issued.

---

## 10. Cryptographic Architecture

### 10.1 Crypto Governance

Cryptographic material management is one of the most complex aspects of this multinational setup. Three national crypto regimes plus NATO crypto must coexist.

**COMSEC Custodian Structure:**

| Domain | COMSEC Custodian | Authority |
|--------|-----------------|-----------|
| NATO crypto material | Project COMSEC Custodian (typically Norwegian as host nation) | NATO NCOIC |
| Norwegian national crypto | Norwegian COMSEC Custodian | NSM |
| Danish national crypto | Danish COMSEC Custodian | CFCS |
| German national crypto | German COMSEC Custodian | BSI/ZDv |

Each national COMSEC custodian is responsible for the handling, storage, and destruction of their national cryptographic material. The crypto storage requirements per nation must be met (e.g., Germany requires BSI-approved safes for crypto material).

### 10.2 Encryption Requirements

| Function | Requirement | Product Type |
|----------|-------------|-------------|
| WAN encryption (NATO) | NATO-approved bulk encryptor | NCOIC-listed (e.g., Thales Datacryptor CP, R&S SITLine) |
| WAN encryption (NOR) | NSM-approved encryptor | NSM crypto catalogue |
| WAN encryption (DNK) | CFCS-approved encryptor | CFCS crypto catalogue |
| WAN encryption (DEU) | BSI-approved encryptor | BSI crypto catalogue (e.g., SINA box, Genua) |
| Data at rest (NATO Shared) | AES-256, approved crypto module | FIPS 140-2 L2+ or CC-evaluated |
| Data at rest (NOR) | NSM-approved or accepted | Per NSM guidance |
| Data at rest (DNK) | CFCS-approved or accepted | Per CFCS guidance |
| Data at rest (DEU) | BSI-approved | BSI-certified module required for GEHEIM |
| TLS internal (each enclave) | TLS 1.2+ with approved cipher suites | Internal PKI per enclave |
| CDS link encryption | As approved per CDS accreditation | May be built into CDS product |

### 10.3 Key Management

- Each enclave maintains independent PKI infrastructure.
- NATO Shared enclave: Standalone CA hierarchy (Root CA offline, Issuing CA online).
- National enclaves: Either standalone CA or subordinate to national defense PKI (via WAN connection).
- Key escrow and recovery procedures defined per enclave, following the most restrictive applicable policy.
- Crypto key destruction follows national and NATO COMSEC procedures.
- Hardware Security Modules (HSMs) for CA private keys and critical key material — CC EAL4+ certified minimum.

---

## 11. Operational Security and Monitoring

### 11.1 Security Operations Center (SOC)

A **local SOC capability** operates within the facility to monitor all enclaves:

```
+-----------------------------------------------------------+
|                    SOC Operations Room                      |
|  (Zone 3 — NATO SECRET cleared, multi-national team)       |
|                                                             |
|  +------------------+  +------------------+                 |
|  | NATO Shared SIEM |  | Aggregated Alert |                 |
|  | Console          |  | Dashboard        |                 |
|  +------------------+  +------------------+                 |
|                                                             |
|  +------------------+  +------------------+                 |
|  | NOR Enclave SIEM |  | DNK Enclave SIEM |                 |
|  | (NOR personnel)  |  | (DNK personnel)  |                 |
|  +------------------+  +------------------+                 |
|                                                             |
|  +------------------+  +------------------+                 |
|  | DEU Enclave SIEM |  | CDS Monitor      |                 |
|  | (DEU personnel)  |  | Console          |                 |
|  +------------------+  +------------------+                 |
+-----------------------------------------------------------+
```

**Staffing model:**
- Each national enclave's SIEM console is operated by personnel cleared for that nation's data.
- The NATO Shared SIEM and aggregated dashboard are operated by the multi-national SOC team.
- National SOC operators can flag cross-enclave concerns to the joint SOC lead without revealing national data content.
- 24/7 monitoring during operational periods (or 24/7 with automated alerting and on-call roster).

### 11.2 Logging Architecture

**All enclaves generate logs covering:**
- Authentication events (success, failure, account lockout)
- Authorization events (access granted, denied, privilege escalation)
- System events (service start/stop, configuration changes, patch application)
- Network events (firewall allow/deny, IDS/IPS alerts, network flow data)
- CDS events (transfer requests, approvals, denials, content inspection results)
- Physical access events (door access, alarm events)

**Log storage:**
- Logs are stored within each enclave's SIEM (national logs stay within the national enclave).
- Aggregated alert metadata (not content) can be forwarded to the NATO Shared SIEM for correlation.
- Log retention: Minimum 12 months online, 36 months archived (or as required by the most restrictive national policy).
- Tamper-evident logging: Write-once storage or cryptographic chaining for audit logs.

### 11.3 Intrusion Detection and Prevention

- **Network IDS/IPS** at each enclave boundary and at the CDS boundaries.
- **Host-based IDS (HIDS)** on all servers and critical workstations.
- **Endpoint Detection and Response (EDR)** on all endpoints where nationally approved EDR products are available.
- Signature updates via controlled, offline update mechanism (no internet connectivity from classified networks).
- IDS/IPS tuned to minimize false positives while maintaining detection capability — tuning documented and approved by SAB.

### 11.4 Incident Response

**Incident Response Framework:**

1. **Detection** — SOC identifies potential incident via SIEM alert, IDS, user report, or CDS anomaly.
2. **Triage** — SOC lead assesses severity and determines which enclaves are affected.
3. **Notification** — Affected national NSRs are immediately informed. For incidents potentially affecting NATO data, NOS/NCSC is notified.
4. **Containment** — Affected systems or network segments are isolated. CDS transfers may be suspended.
5. **Investigation** — Led by a joint team; national enclave investigation is led by national personnel with national authority oversight.
6. **Eradication and Recovery** — Following approved procedures, systems are cleaned or rebuilt from known-good baselines.
7. **Reporting** — Formal incident report to SAB, all three national NSAs, and NATO (NCIRC/NCSC) as applicable.
8. **Lessons Learned** — Post-incident review with security improvements fed back into the accreditation documentation.

**Incident severity levels:**

| Level | Description | Response Time | Notification |
|-------|-------------|---------------|-------------|
| Critical | Confirmed compromise of SECRET data or CDS bypass | Immediate | All NSAs, NOS, PSO within 1 hour |
| High | Attempted compromise, malware detection on classified system | < 1 hour | NSRs, PSO within 4 hours, NSAs within 24 hours |
| Medium | Policy violation, anomalous activity, contained incident | < 4 hours | PSO, relevant NSR within 24 hours |
| Low | Minor policy deviation, false positive confirmed | Next business day | Logged and reported in periodic security report |

### 11.5 Vulnerability Management

- Regular vulnerability assessments (at least quarterly) of all enclaves.
- Penetration testing annually or after significant changes, performed by SAB-approved team with appropriate clearances.
- Patch management: Security patches applied within defined SLAs (critical: 72 hours, high: 7 days, medium: 30 days) after testing in a non-production environment.
- Patch sources: Vendor patches obtained through controlled supply chain (not downloaded from internet on classified systems).

---

## 12. Accreditation Strategy

### 12.1 Accreditation Approach

The system follows a **phased accreditation** approach to enable early capability delivery:

**Phase 1: Facility and Infrastructure Accreditation**
- Physical facility accreditation by NSM (as host nation SAA for the facility).
- Bilateral/trilateral security arrangement confirmation.
- Timeline: Months 1-6

**Phase 2: Single Enclave Accreditation (NATO Shared)**
- NATO Shared enclave accreditation by SAB.
- Enables initial multinational collaboration on NATO-classified data.
- Interim cross-domain via manual media transfer (sneakernet) with two-person review.
- Timeline: Months 6-12

**Phase 3: National Enclave Accreditation**
- Each national enclave accredited by respective national SAA.
- Can proceed in parallel.
- Timeline: Months 9-18 (Germany likely longest due to GEHEIM requirements)

**Phase 4: CDS Accreditation**
- Each CDS instance accredited by SAB with national SAA concurrence.
- Highest risk and longest lead time.
- Timeline: Months 12-24

**Phase 5: Full Operational Capability**
- All enclaves and CDS instances operational.
- Continuous monitoring and periodic re-accreditation.
- Timeline: Month 24+

### 12.2 Accreditation Documentation

Each accreditation requires a documentation package. The following is the minimum expected set:

| Document | Scope | Responsibility |
|----------|-------|---------------|
| System Security Concept (Sicherheitskonzept / Sikkerhetskonsept) | Per enclave + overall system | System ISSO with national guidance |
| Risk Assessment | Per enclave + CDS + overall | ISSO + national risk assessment methodology |
| Security Operating Procedures (SecOPs) | Per enclave + CDS + facility | ISSO + operations team |
| TEMPEST Assessment / Certificate | Facility and enclave rooms | NSM TEMPEST authority (+ BSI for DEU) |
| Interconnection Security Agreement (ISA) | Each CDS connection, each WAN connection | Joint (both sides of connection) |
| Configuration Management Plan | Overall system | Configuration manager |
| Incident Response Plan | Overall system + national specifics | ISSO + SOC lead |
| Business Continuity / Disaster Recovery Plan | Overall system | Operations + PSO |
| Penetration Test Report | Per accreditation phase | Independent test team |
| Residual Risk Statement | Per enclave + overall | ISSO, accepted by SAB |

### 12.3 Ongoing Accreditation

- **Continuous monitoring** reports submitted to SAB quarterly.
- **Annual security review** with all three national SAAs.
- **Re-accreditation triggers:** Major system change, security incident, change in threat level, national policy change, or maximum 3-year accreditation validity (whichever comes first).

---

## 13. Risk Assessment Summary

### 13.1 Key Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | German SAA does not approve GEHEIM processing in Norway | Medium | Critical | Early engagement with BSI/BMVg; consider processing only VS-VERTRAULICH nationally with GEHEIM data remaining in Germany via WAN link |
| R2 | CDS accreditation takes longer than 24 months | Medium | High | Interim sneakernet procedure; begin CDS procurement and documentation immediately |
| R3 | No single CDS product acceptable to all three nations | Medium | High | Accept different CDS products per national link; increases cost but reduces accreditation risk |
| R4 | TEMPEST Zone 1 cost overrun | Low | Medium | Early TEMPEST assessment; design for Zone 1 from the start |
| R5 | Personnel clearance delays (cross-national) | High | Medium | Begin clearance processes early; identify cleared personnel from existing pools |
| R6 | Supply chain compromise of classified IT equipment | Low | Critical | Procure through national defense procurement channels; use trusted supplier lists |
| R7 | Insider threat (multinational environment increases complexity) | Medium | Critical | Enclave separation; monitoring; two-person integrity for critical operations; personnel vetting |
| R8 | Incompatible national crypto requirements | Medium | High | Early engagement with all three COMSEC authorities; separate crypto per enclave |
| R9 | National policy change during programme lifecycle | Medium | Medium | Modular enclave design allows adaptation; SAB provides change management forum |
| R10 | Interoperability challenges between national tools | High | Medium | Standardize on common tools in NATO Shared enclave; accept national tools in national enclaves |

### 13.2 Risk R1 Deep Dive: German GEHEIM Processing Abroad

This is the highest-impact risk. Germany has historically been conservative about allowing GEHEIM processing outside German territory. Mitigations:

1. **Option A (preferred):** BSI approves GEHEIM processing in Norway based on the combined physical security, German enclave controls, and German COMSEC custodian presence. Requires detailed Sicherheitskonzept and likely a BSI site inspection.

2. **Option B (fallback):** The German national enclave in Norway processes only VS-VERTRAULICH and below. German GEHEIM data remains in Germany and is accessed by German team members via the German national WAN link. This limits what can be locally stored but may be more acceptable to BSI.

3. **Option C (hybrid):** German GEHEIM data is processed in Germany, but a CDS link between the German national network in Germany and the NATO Shared enclave in Norway enables cross-domain transfer. This adds a WAN hop but keeps GEHEIM data on German soil.

**Recommendation:** Pursue Option A with Option C as the ready fallback. Engage BSI early (month 1) with a preliminary security concept.

---

## 14. Deployment and Operations Model

### 14.1 Deployment Phases

```
Month  1-3:  Facility preparation, TEMPEST installation, power/cooling
Month  3-6:  Infrastructure hardware installation, network cabling
Month  4-6:  Facility accreditation (NSM)
Month  6-9:  NATO Shared enclave build and hardening
Month  9-12: NATO Shared enclave accreditation + initial operations
Month  6-12: National enclave hardware + build (parallel)
Month 12-18: National enclave accreditations (parallel)
Month  6-18: CDS procurement, installation, testing
Month 18-24: CDS accreditation
Month 24:    Full Operational Capability (FOC)
```

### 14.2 Operations Team Structure

| Role | Nationality Requirement | Clearance | Responsibility |
|------|------------------------|-----------|---------------|
| Infrastructure Manager | Any (NOR preferred as host) | NATO SECRET | Overall infrastructure operations |
| NATO Shared Enclave Admin | Any participating nation | NATO SECRET | Shared enclave day-to-day |
| NOR Enclave Admin | Norwegian | HEMMELIG | Norwegian enclave operations |
| DNK Enclave Admin | Danish | HEMMELIGT | Danish enclave operations |
| DEU Enclave Admin | German | GEHEIM (Ue2) | German enclave operations |
| CDS Operator | Multi-national (requires special vetting) | NATO SECRET + national | CDS operations and monitoring |
| SOC Analyst (x3 minimum) | One per nation minimum | National SECRET + NATO SECRET | Security monitoring |
| COMSEC Custodians (x4) | One per nation + NATO | National SECRET + NATO SECRET | Crypto material management |
| ISSO / System Security Officer | Any (NOR preferred) | NATO SECRET | Security documentation, liaison to SAB |
| Facility Security Officer | Norwegian | HEMMELIG | Physical security, Norwegian regulatory interface |

### 14.3 Maintenance and Change Management

- **Change Advisory Board (CAB):** Includes representatives from all three nations. All changes to shared infrastructure and CDS require CAB approval.
- **National enclave changes:** National admin proposes, national NSR approves, CAB is informed.
- **Emergency changes:** Defined emergency change procedure allowing rapid response with retrospective CAB approval.
- **Maintenance windows:** Coordinated across enclaves to minimize operational disruption. Monthly scheduled maintenance window.

### 14.4 Backup and Recovery

- Each enclave maintains independent backup infrastructure.
- Backups are encrypted with enclave-specific keys.
- Backup media stored in rated security containers within the facility (or off-site at an equivalent-rated facility if required).
- Recovery procedures tested quarterly.
- Cross-enclave recovery dependencies minimized — each enclave can be independently restored.
- Recovery Time Objective (RTO) and Recovery Point Objective (RPO) defined per operational requirements (recommended: RTO 24h, RPO 4h for critical services).

### 14.5 Supply Chain Security

- Hardware procured through national defense procurement channels where possible.
- Trusted supplier verification for all components entering the classified environment.
- Hardware inspection upon delivery (visual inspection, tamper evidence check).
- Firmware/BIOS integrity verification before deployment.
- Software sourced from verified vendor channels; integrity verified via cryptographic hashes.
- No COTS products from adversary nations in classified processing equipment (per national supply chain policies).

---

## 15. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|-----------|
| BSI | Bundesamt fuer Sicherheit in der Informationstechnik (German Federal Office for Information Security) |
| CAB | Change Advisory Board |
| CDS | Cross-Domain Solution |
| CFCS | Centre for Cybersikkerhed (Danish Centre for Cyber Security) |
| COMSEC | Communications Security |
| DAA | Designated Accreditation Authority |
| EDR | Endpoint Detection and Response |
| FE | Forsvarets Efterretningstjeneste (Danish Defense Intelligence Service) |
| FOC | Full Operational Capability |
| FSC | Facility Security Clearance |
| HIDS | Host-based Intrusion Detection System |
| HSM | Hardware Security Module |
| IDS/IPS | Intrusion Detection System / Intrusion Prevention System |
| ISSO | Information System Security Officer |
| LIST-X | NATO term for a cleared contractor facility |
| MAD | Militaerischer Abschirmdienst (German Military Counterintelligence Service) |
| MMHS | Military Message Handling System |
| NCOIC | NATO Communications and Information Organisation Catalogue |
| NCSC | NATO Cyber Security Centre |
| NOS | NATO Office of Security |
| NSA | National Security Authority |
| NSM | Nasjonal Sikkerhetsmyndighet (Norwegian National Security Authority) |
| NSR | National Security Representative |
| OOB | Out-of-Band |
| PAM | Privileged Access Management |
| PKI | Public Key Infrastructure |
| PSC | Personnel Security Clearance |
| PSI | Programme Security Instruction |
| PSO | Programme Security Officer |
| SAA | Security Accreditation Authority |
| SAB | Security Accreditation Board |
| SecOPs | Security Operating Procedures |
| SIEM | Security Information and Event Management |
| SINA | Sichere Inter-Netzwerk Architektur (BSI-approved crypto platform) |
| SOC | Security Operations Center |
| STANAG | NATO Standardization Agreement |
| TEMPEST | Telecommunications Electronics Material Protected from Emanating Spurious Transmissions |
| VSA | Verschlusssachenanweisung (German Classified Information Instruction) |

### Appendix B: Reference Documents

1. NATO C-M(2002)49 — Security within the North Atlantic Treaty Organisation
2. NATO AC/35-D/2004 — Primary Directive on CIS Security
3. STANAG 4774 — Confidentiality Metadata Label Syntax
4. STANAG 4778 — Metadata Binding Mechanism
5. SDIP-27 — NATO TEMPEST Standards
6. SDIP-29 — NATO Cabling Standards
7. Norwegian Security Act (Sikkerhetsloven) 2018
8. Danish Security Circular (Sikkerhedscirkulaeret)
9. German VSA (Verschlusssachenanweisung)
10. BSI IT-Grundschutz Compendium
11. NATO NIST-equivalent Risk Assessment Methodology

### Appendix C: Architecture Decision Records

**ADR-001: Physical separation over logical separation for national enclaves**
- *Decision:* Each national enclave uses physically dedicated compute, storage, and networking hardware.
- *Rationale:* Logical separation (e.g., VLANs on shared switches) is insufficient for SECRET-level multi-national separation. Physical separation is required for accreditation by all three national SAAs, particularly BSI for GEHEIM.
- *Consequence:* Higher hardware cost, more complex management, but achievable accreditation.

**ADR-002: No direct national-to-national CDS**
- *Decision:* All cross-national data flow traverses the NATO Shared enclave as intermediary.
- *Rationale:* Reduces CDS count from 6 (full mesh) to 3 (star topology via NATO Shared). NATO policy serves as the common release arbitration framework. National bilateral release would require additional bilateral CDS accreditation that is disproportionately complex.
- *Consequence:* Slightly higher latency for cross-national transfers; all cross-national data must have NATO releasability markings.

**ADR-003: Separate identity stores per enclave (no federation)**
- *Decision:* No AD trust relationships or identity federation between enclaves.
- *Rationale:* Federation creates a trust dependency and potential lateral movement path. Each enclave must be independently compromisable only — a breach in one enclave must not provide authenticated access to another.
- *Consequence:* Users manage multiple credentials; user provisioning process must be robust.

**ADR-004: German enclave fallback architecture**
- *Decision:* Design supports both local GEHEIM processing (Option A) and remote GEHEIM with local VS-VERTRAULICH (Option B/C).
- *Rationale:* BSI approval for GEHEIM processing in Norway is uncertain. The architecture must accommodate both outcomes.
- *Consequence:* German enclave hardware is provisioned for GEHEIM capability; if BSI approves only VS-VERTRAULICH locally, the hardware is still usable with additional WAN-based workflow.

**ADR-005: TEMPEST Zone 1 for entire classified area**
- *Decision:* Design the entire Zone 3 area to SDIP-27 Zone 1 standards.
- *Rationale:* Germany requires Zone 1 for GEHEIM. Building mixed zones within the same area is complex and error-prone. Uniform Zone 1 simplifies design and satisfies all national requirements.
- *Consequence:* Higher initial construction cost, but eliminates TEMPEST zone boundary management complexity.

---

*End of Document*
