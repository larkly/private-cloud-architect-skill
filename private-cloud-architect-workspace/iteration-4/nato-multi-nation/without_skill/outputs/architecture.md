# NATO SECRET Multi-Nation Shared Infrastructure Architecture

## Participating Nations: Norway, Denmark, Germany
## Classification: NATO SECRET / National SECRET equivalents
## Hosting: LIST-X Equivalent Facility, Norway

---

## 1. Executive Summary

This document describes the architecture for a shared NATO SECRET-level infrastructure supporting a multinational defense project between Norway, Denmark, and Germany. The platform must process NATO SECRET data on a common infrastructure while simultaneously handling nationally-classified data from each participating nation with proper separation. The facility is located in Norway under a LIST-X equivalent accreditation.

The architecture employs a hub-and-spoke model: a shared NATO SECRET processing domain at the center, with isolated national enclaves for each participating nation, connected through controlled cross-domain interfaces.

---

## 2. Regulatory and Policy Framework

### 2.1 NATO Requirements

- **NATO Security Policy (C-M(2002)49)**: Governs the handling of NATO classified information. All systems processing NATO SECRET must comply with NATO security regulations.
- **AC/322-D(2004)0066 (INFOSEC Technical and Implementation Directive)**: Defines technical security controls for NATO CIS systems.
- **NATO SDIP-27 (TEMPEST)**: Emanation security standards; the facility and equipment must meet SDIP-27 Level A or B depending on zone placement.
- **NATO NIST/COSMIC accreditation**: The shared NATO SECRET domain requires accreditation by the NATO Communications and Information Agency (NCIA) or delegated national authority.
- **STANAG 4774 / 4778**: Metadata binding and confidentiality labelling standards for data objects, critical for cross-domain operations.

### 2.2 Norwegian National Framework

- **Sikkerhetsloven (Security Act, 2019)**: Governs protection of national security interests including classified information, critical infrastructure, and security clearances.
- **Forskrift om informasjonssikkerhet (Information Security Regulation)**: Technical requirements for systems processing Norwegian classified information (BEGRENSET through STRENGT HEMMELIG).
- **NSM (Nasjonal sikkerhetsmyndighet)**: The Norwegian National Security Authority is the accreditation authority for Norwegian national systems and oversees LIST-X equivalent facility accreditation.
- **Norwegian BEGRENSET / KONFIDENSIELT / HEMMELIG / STRENGT HEMMELIG**: Norwegian classification levels. Norwegian HEMMELIG is the bilateral equivalent roughly corresponding to NATO SECRET.
- As the host nation, Norway holds particular responsibilities for physical security, facility accreditation, and personnel security of facility staff.

### 2.3 Danish National Framework

- **Sikkerhedscirkulaeret (Security Circular)**: Denmark's primary directive on classified information handling.
- **Centre for Cyber Security (CFCS)**: Danish national authority for accreditation of IT systems handling classified data.
- **Danish classification levels**: TIL TJENESTEBRUG / FORTROLIGT / HEMMELIGT / YDERST HEMMELIGT. Danish HEMMELIGT corresponds to NATO SECRET.
- **Danish CFCS Technical Requirements**: Systems processing Danish nationally classified data must meet CFCS technical security requirements, which may differ from Norwegian NSM requirements in specific controls.

### 2.4 German National Framework

- **Verschlusssachenanweisung (VSA, Classified Information Instruction)**: Primary German regulation governing classified information.
- **BSI (Bundesamt fuer Sicherheit in der Informationstechnik)**: The German Federal Office for Information Security provides technical guidance and evaluations.
- **German classification levels**: VS-NUR FUER DEN DIENSTGEBRAUCH / VS-VERTRAULICH / GEHEIM / STRENG GEHEIM. German GEHEIM corresponds to NATO SECRET.
- **BSI IT-Grundschutz**: German baseline security methodology; German national enclaves must comply.
- **Approval by the German National Security Authority (BMWi / BMVg depending on domain)**: Required for German national compartment operation on foreign soil.

### 2.5 Bilateral and Multilateral Agreements Required

- **Host Nation Security Agreement (HNSA)**: Between Norway and the multinational project, covering physical security responsibilities.
- **Programme Security Instruction (PSI)**: Project-specific security document defining classification guidance, handling rules, and personnel security requirements.
- **Bilateral Security Agreements**: Norway-Denmark, Norway-Germany, Denmark-Germany bilateral general security agreements (these already exist but must be referenced in the PSI).
- **Facility Security Clearance (FSC)**: The LIST-X facility must hold an FSC at SECRET level, acknowledged by all three nations.
- **Memorandum of Understanding (MOU) / Technical Arrangement (TA)**: Defining the shared infrastructure governance, cost sharing, and operational responsibilities.

---

## 3. Facility and Physical Security

### 3.1 LIST-X Facility Accreditation

The hosting facility in Norway must be accredited as a LIST-X equivalent under Norwegian national procedures (NSM oversees this). Key requirements:

- **Physical security zone model**: The facility must implement concentric security zones:
  - **Zone 1 (Administrative Zone)**: Controlled access, no classified processing.
  - **Zone 2 (Security Zone)**: Access restricted to cleared personnel; NATO RESTRICTED / national equivalent processing permitted.
  - **Zone 3 (High-Security Zone)**: Where NATO SECRET and national SECRET processing occurs. Dual-person integrity, IDS (intrusion detection systems), CCTV, access logging, and alarm response within defined timelines.
- **TEMPEST zoning**: Equipment in Zone 3 must comply with SDIP-27 Level A or be installed with sufficient TEMPEST countermeasures (distance, shielding) per SDIP-27 Level B with an approved installation plan.
- **Access control**: Biometric + token + PIN (multi-factor). Escort procedures for uncleared visitors. Nation-specific enclave rooms may require additional access restrictions (e.g., only German-cleared personnel may access the German national enclave server room if physical separation is mandated by the BSI/German NSA).

### 3.2 National Enclave Physical Considerations

Depending on the national requirements (Germany in particular may require this):

- **Physically separated server racks or cages** for national enclave hardware, with nation-specific access controls.
- **Separate cable routing** for national enclaves where mandated, or at minimum cable labelling and separation within shared cable trays.
- **National security authority inspection access**: Each nation's security authority must be able to inspect their enclave hardware and configuration.

---

## 4. Logical Architecture

### 4.1 Domain Model

The architecture consists of four logical security domains:

```
+-------------------------------------------------------------+
|                    LIST-X FACILITY (Norway)                  |
|                                                              |
|  +-------------------+   +-------------------+               |
|  | Norwegian National|   | Danish National   |               |
|  | Enclave           |   | Enclave           |               |
|  | (NO HEMMELIG)     |   | (DK HEMMELIGT)    |               |
|  +--------+----------+   +--------+----------+               |
|           |                        |                         |
|           |   +----------------+   |                         |
|           +-->| Cross-Domain   |<--+                         |
|               | Service (CDS)  |                             |
|           +-->|                |<--+                         |
|           |   +-------+--------+   |                         |
|           |           |            |                         |
|  +--------+----------+|  +---------+---------+               |
|  | German National    ||  | NATO SECRET       |               |
|  | Enclave            ||  | Shared Domain     |               |
|  | (DE GEHEIM)        ||  | (Common Services) |               |
|  +--------------------+|  +-------------------+               |
|                        |                                     |
+-------------------------------------------------------------+
```

### 4.2 NATO SECRET Shared Domain

This is the common processing environment where multinational collaboration occurs on NATO-classified data.

- **Classification**: NATO SECRET
- **Access**: All project personnel with valid NATO SECRET clearance and need-to-know
- **Services hosted**:
  - Shared collaboration platform (e.g., document management, messaging)
  - Common project databases and applications
  - Shared development/test environments for project deliverables
  - Directory services (Active Directory or equivalent) for the NATO domain
  - DNS, NTP, log aggregation, SIEM for the NATO domain
- **Network**: Dedicated NATO SECRET VLAN/network segment, firewalled from national enclaves
- **Accreditation**: Jointly accredited under NATO procedures, with the lead nation (Norway/NSM) acting as the Security Accreditation Authority (SAA) under delegated authority or in coordination with NCIA

### 4.3 National Enclaves

Each nation operates an isolated enclave for nationally-classified data that must not be shared with other nations or processed in the NATO domain without explicit release.

#### 4.3.1 Norwegian Enclave (NO HEMMELIG)

- Processes Norwegian nationally-classified data up to HEMMELIG
- Accredited by NSM under Norwegian regulations
- Operated by Norwegian-cleared personnel
- May share infrastructure (hypervisor, SAN) with the NATO domain if NSM approves, since Norway is the host nation and Norwegian HEMMELIG is considered equivalent to NATO SECRET (same accreditation authority oversight)

#### 4.3.2 Danish Enclave (DK HEMMELIGT)

- Processes Danish nationally-classified data up to HEMMELIGT
- Accredited by CFCS under Danish regulations
- Operated by Danish-cleared (or dual-cleared) personnel
- CFCS may require logical or physical separation from other national enclaves and from the NATO domain
- Danish enclave must have its own encryption for any data in transit to Denmark (Danish-approved crypto)

#### 4.3.3 German Enclave (DE GEHEIM)

- Processes German nationally-classified data up to GEHEIM
- Accredited by the German national authority (BSI/BMVg)
- Germany typically has the most stringent requirements for operating classified systems on foreign soil:
  - May require physically dedicated hardware (servers, storage, network switches) not shared with any other domain
  - German-approved encryption (BSI-approved) mandatory for any connectivity back to Germany
  - German-cleared personnel only for administration
  - Potentially a requirement for a German-controlled HSM for key management
  - BSI may require a dedicated security officer on-site or regular inspection rights

### 4.4 Network Architecture

```
                     +---------------------------+
                     |   External Connectivity   |
                     |   (Encrypted WAN links)   |
                     +--+------+------+------+---+
                        |      |      |      |
                  [NO Crypto] [DK Crypto] [DE Crypto] [NATO Crypto]
                        |      |      |      |
                     +--+------+------+------+---+
                     |  Border Firewall Cluster   |
                     |  (NATO SECRET boundary)    |
                     +--+------+------+------+---+
                        |      |      |      |
            +-----------+--+ +-+----------+ +-+-----------+ +--+-----------+
            | NO Enclave   | | DK Enclave | | DE Enclave  | | NATO SECRET |
            | Network      | | Network    | | Network     | | Shared Net  |
            | (VLAN/VRF)   | | (VLAN/VRF) | | (VLAN/VRF)  | | (VLAN/VRF)  |
            +-----------+--+ +-+----------+ +-+-----------+ +--+-----------+
                        |      |      |      |
                     +--+------+------+------+---+
                     |   Cross-Domain Service     |
                     |   (Controlled interface)   |
                     +----------------------------+
```

Key network design principles:

- **Separate VLANs / VRFs** for each domain (NO, DK, DE, NATO), enforced at the switch level with 802.1Q trunking and strict ACLs.
- **Dedicated firewall contexts or separate firewall appliances** between each domain. The firewalls must be approved for the classification level (common-criteria evaluated or nationally-approved products).
- **No direct routing** between national enclaves. All cross-domain data flow goes through the CDS.
- **Encrypted WAN links** to each nation use nation-approved cryptographic devices:
  - Norway: NSM-approved IP encryptors (e.g., Thales/Rohde & Schwarz devices approved by NSM)
  - Denmark: CFCS-approved crypto
  - Germany: BSI-approved crypto (e.g., SINA or equivalent)
  - NATO: NATO-approved COMSEC (e.g., NATO-certified encryptors per SDIP-28)
- **Network monitoring**: Each domain has its own IDS/IPS. The NATO shared domain has a central SIEM. National enclaves may forward security events to their national SOCs via the encrypted WAN links.

---

## 5. Cross-Domain Solution (CDS)

### 5.1 Purpose

The CDS is the controlled gateway enabling data exchange between the national enclaves and the NATO SECRET shared domain. It is the most critical security component in the architecture.

### 5.2 CDS Design Principles

- **One-way by default**: Data flows from national enclaves to the NATO domain only when explicitly released. Reverse flows (NATO to national) are permitted but controlled.
- **No direct national-to-national transfer**: A Danish user cannot send data directly to the German enclave. Data must be released to the NATO domain first, then the receiving nation can pull from the NATO domain into their enclave (two-step release).
- **Content inspection**: The CDS must inspect all data transfers for:
  - Classification marking validation (STANAG 4774/4778 metadata labels)
  - Malware scanning
  - Data type filtering (block unauthorized file types)
  - Size and rate limiting
- **Human-in-the-loop release**: Transfers of nationally-classified data to the NATO domain require explicit human approval by an authorized releasing officer from the originating nation.
- **Audit logging**: Every transfer (attempted and completed) must be logged with full metadata (source domain, destination domain, user, timestamp, classification label, file hash, approval reference).

### 5.3 CDS Implementation Options

- **Hardware-based CDS appliance**: Purpose-built cross-domain guards (e.g., Oakley Systems, Forcepoint Trusted Gateway, or nationally-approved equivalents). These are Common Criteria evaluated and often mandated by national authorities.
- **Data diode + review station**: For strictly one-way flows, a hardware data diode (e.g., Advenica SecuriCDS, VADO/Nexor) combined with a manual review station.
- **API-based controlled gateway**: For structured data (databases, web services), a validated API proxy that enforces schema validation and label checking.

The choice must be agreed upon by all three national security authorities and the NATO accreditation authority. In practice, this is often the longest lead-time item in accreditation.

### 5.4 Classification Label Management

- All data objects in the system must carry security labels per STANAG 4774 (Confidentiality Metadata Label Syntax) and STANAG 4778 (Metadata Binding Mechanism).
- Labels encode: classification level, national caveats (e.g., "NO EYES ONLY", "DEU EYES ONLY"), releasability markings (e.g., "REL TO NATO", "REL TO NO, DK, DE").
- The CDS validates labels before permitting transfer. A document marked "DEU EYES ONLY / GEHEIM" must not transit to the NATO shared domain unless re-marked by an authorized German releasing officer.

---

## 6. Compute and Storage Architecture

### 6.1 Virtualization Strategy

Two viable approaches, depending on national authority acceptance:

**Option A: Shared Hypervisor with Strong Isolation (Preferred for cost/efficiency)**

- A common virtualization platform (e.g., VMware vSphere with NSX, or an evaluated alternative) hosts VMs for all four domains.
- Isolation enforced through:
  - Separate virtual switches (no shared port groups between domains)
  - Hypervisor-level micro-segmentation (NSX distributed firewall or equivalent)
  - Separate vCenter instances or RBAC-restricted management for each domain
  - Encrypted VM disks (vSAN encryption or equivalent) with per-domain key management
- This option requires all three national authorities to accept the shared hypervisor as a trusted separation mechanism. Germany (BSI) may not accept this for GEHEIM without a specifically-evaluated hypervisor (e.g., one with a BSI-recognized separation kernel or a VS-NfD-approved virtualization stack extended by waiver to GEHEIM).

**Option B: Physically Dedicated Hardware per Domain (Conservative)**

- Each domain runs on its own physical servers, storage arrays, and network switches.
- Higher cost and lower resource utilization but simpler accreditation arguments.
- Germany will likely require this option for the DE GEHEIM enclave.
- Norway and Denmark may accept shared infrastructure for their enclaves and the NATO domain.

**Recommended hybrid approach**: Shared hypervisor for the NATO domain, Norwegian enclave, and Danish enclave (subject to NSM and CFCS approval). Physically dedicated hardware for the German GEHEIM enclave.

### 6.2 Storage Architecture

- **NATO SECRET shared storage**: SAN or HCI (hyper-converged) storage accessible to the NATO domain. Encrypted at rest with keys managed by the NATO domain key management system.
- **National enclave storage**: Each enclave has logically or physically separate storage volumes. Encryption at rest with nation-specific key management.
- **No shared storage LUNs across domains**. Each domain sees only its own storage. For the shared hypervisor option, this is enforced through storage zoning (FC zoning or iSCSI ACLs) and separate datastores.
- **Backup**: Each domain manages its own backup. Backup media is classified at the domain's classification level and must be stored and handled accordingly. Cross-domain backup consolidation is not permitted.

### 6.3 Key Management

- Each domain operates its own key management infrastructure.
- Hardware Security Modules (HSMs) are recommended for each domain. Germany will likely mandate a BSI-approved HSM for the German enclave.
- No shared key material across domains.

---

## 7. Identity and Access Management

### 7.1 Directory Services

- **NATO shared domain**: Dedicated Active Directory forest (or equivalent) for the NATO domain. User accounts provisioned for all project personnel with NATO SECRET clearance.
- **National enclaves**: Each enclave operates its own directory or has accounts in the NATO forest with additional national role-based access controls. The German enclave will likely require its own independent directory.

### 7.2 Authentication

- **Multi-factor authentication (MFA)** required for all domains: smartcard/PKI certificate + PIN as the baseline.
- **National PKI**: Each nation issues its own smartcards/tokens through national PKI hierarchies. Cross-recognition of certificates between national PKIs and the NATO PKI must be configured (certificate trust chains).
- **NATO PKI**: For the shared NATO domain, either NATO-issued certificates or nationally-issued certificates cross-certified with the NATO trust chain.

### 7.3 Authorization and Need-to-Know

- Role-Based Access Control (RBAC) supplemented by Attribute-Based Access Control (ABAC) for fine-grained need-to-know enforcement.
- Security labels on data objects (per STANAG 4774) are matched against user clearance attributes (clearance level, national caveats, project access) to determine access.
- Privileged access (system administration) is strictly separated per domain. A Norwegian sysadmin for the NO enclave has no administrative access to the DK or DE enclaves.

---

## 8. Accreditation Strategy

### 8.1 Accreditation Approach

Given the multi-national nature, a phased accreditation approach is necessary:

1. **Facility accreditation**: NSM accredits the LIST-X facility for NATO SECRET and Norwegian HEMMELIG. Denmark (CFCS) and Germany (German NSA) conduct facility inspections and issue bilateral acceptance.
2. **NATO SECRET shared domain accreditation**: Lead by NSM (as host nation SAA) with review/concurrence from NCIA and the other two nations' NSAs. Produce a Security Accreditation Documentation Package (SADP) including:
   - System Security Plan (SSP)
   - Risk Assessment (RA) per NATO risk management methodology
   - Security Operating Procedures (SecOPs)
   - Interconnection agreements
3. **National enclave accreditation**: Each nation accredits its own enclave:
   - Norway: NSM accredits the NO enclave
   - Denmark: CFCS accredits the DK enclave
   - Germany: German NSA accredits the DE enclave
4. **CDS accreditation**: The CDS is the interconnection point and must be accredited by all four authorities (three national + NATO). This is typically the most contentious and time-consuming element.

### 8.2 Accreditation Documentation

Each domain requires:

| Document | NATO Domain | NO Enclave | DK Enclave | DE Enclave |
|---|---|---|---|---|
| Security Target / Protection Profile | Yes | Yes | Yes | Yes (BSI PP) |
| Risk Assessment | NATO RA methodology | NSM methodology | CFCS methodology | BSI IT-Grundschutz |
| Security Operating Procedures | Shared SecOPs | NO-specific | DK-specific | DE-specific |
| TEMPEST Assessment | Facility-wide (single assessment) | - | - | - |
| Interconnection Agreement (CDS) | All parties sign | All parties sign | All parties sign | All parties sign |
| Penetration Test Report | Yes | Yes | Yes | Yes (BSI may require own testers) |

### 8.3 Governance

- **Multinational Security Accreditation Board (SAB)**: Comprising representatives from NSM, CFCS, German NSA, and NCIA. Meets periodically to review security posture, approve changes, and address incidents.
- **Configuration Control Board (CCB)**: Any change to shared infrastructure or CDS requires CCB approval with representation from all nations.
- **Security Incident Response**: Defined in SecOPs. Incidents in the NATO domain are reported to all nations. Incidents in a national enclave are reported to that nation's NSA, with notification to other nations if the shared domain may be affected.

---

## 9. Operational Considerations

### 9.1 Personnel Security

- All facility staff must hold Norwegian security clearance at HEMMELIG level (host nation requirement).
- Staff accessing the NATO domain must hold NATO SECRET clearance.
- Staff accessing a national enclave must hold that nation's clearance at the corresponding level.
- System administrators for the German enclave must hold German GEHEIM clearance, which may require German nationality (BSI/German NSA determination).

### 9.2 Supply Chain Security

- Hardware and software procurement must follow NATO and national supply chain security requirements.
- COTS products should be from NATO/EU-origin vendors where possible. Products from adversarial-origin vendors are prohibited.
- Hardware for the German enclave may need to be procured through German-approved supply chains.

### 9.3 TEMPEST and Emanation Security

- Facility-wide SDIP-27 assessment determines equipment placement and any required shielding.
- If national enclaves have different TEMPEST requirements (e.g., Germany mandating Zone A compliance for GEHEIM), these must be accommodated in the facility layout.

### 9.4 Disaster Recovery and Continuity

- Each national enclave defines its own DR requirements. DR sites must meet the same accreditation standards.
- The NATO shared domain DR plan must be agreed upon by all nations.
- Backup and recovery procedures are domain-specific and do not cross domain boundaries.

---

## 10. Risk Summary and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| German NSA rejects shared hypervisor for GEHEIM | High | Medium (cost increase) | Plan for physically dedicated German hardware from the outset |
| CDS accreditation delays | High | High (blocks operations) | Begin CDS procurement and evaluation early; engage all NSAs in CDS product selection |
| Differing national requirements create accreditation conflicts | Medium | High | Establish SAB early; negotiate common baseline with national deviations documented |
| Host nation (Norway) personnel cannot obtain all three national clearances | Medium | Medium | Use nation-specific admin teams; accept travel/rotation costs |
| TEMPEST requirements conflict between nations | Low | Medium | Commission TEMPEST survey early; design facility layout to meet the most stringent requirement |
| Supply chain objections to specific vendors | Medium | Medium | Maintain an approved product list agreed by all three NSAs before procurement |

---

## 11. Implementation Roadmap

### Phase 1: Agreements and Governance (Months 1-6)
- Execute MOU/TA between nations
- Draft and agree Programme Security Instruction (PSI)
- Establish SAB and CCB
- Engage all three NSAs and NCIA early

### Phase 2: Facility Preparation (Months 3-12)
- LIST-X facility accreditation (NSM lead)
- TEMPEST survey and remediation
- Physical security upgrades for national enclave zones
- Bilateral facility inspections by CFCS and German NSA

### Phase 3: Infrastructure Deployment (Months 9-18)
- Procure and install hardware (shared + German-dedicated)
- Deploy hypervisor, storage, network infrastructure
- Deploy domain-specific directory services, PKI, SIEM
- CDS procurement and installation (long lead time - begin in Phase 1)

### Phase 4: Domain Accreditation (Months 15-24)
- NATO SECRET shared domain accreditation (NSM + NCIA)
- National enclave accreditations (parallel, nation-led)
- CDS accreditation (all parties, critical path)

### Phase 5: Operational Capability (Months 22-27)
- Initial operating capability with NATO domain
- Progressive national enclave activation
- Full operating capability with CDS-enabled cross-domain sharing

---

## 12. Conclusion

The architecture separates concerns into four distinct security domains -- one shared NATO SECRET domain and three national enclaves -- connected through a controlled cross-domain service. The key challenges are not primarily technical but rather regulatory and procedural: aligning three national accreditation frameworks and NATO requirements, agreeing on shared infrastructure boundaries (particularly with Germany's historically stringent requirements for classified processing on foreign soil), and achieving timely CDS accreditation from all parties.

Early and sustained engagement with all three national security authorities (NSM, CFCS, German NSA) and NCIA is the single most important success factor. Technical design decisions should be validated with the accreditation authorities before procurement to avoid costly re-work.
