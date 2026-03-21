# NATO SECRET Multi-Nation Shared Infrastructure Architecture

## Norway, Denmark, and Germany -- Multinational Defense Project

---

## 1. Executive Summary

This document defines the architecture for a NATO SECRET-level shared infrastructure platform hosted in a LIST-X equivalent facility in Norway, serving a multinational defense project involving Norway, Denmark, and Germany. The platform must process NATO SECRET (NS) data on a common infrastructure while simultaneously handling nationally-classified data from each participating nation with proper cryptographic and logical separation. The architecture addresses the intersection of three national security frameworks, NATO security policy, and the operational requirement for controlled cross-domain data sharing between national enclaves.

---

## 2. Classification Hierarchy and Framework Mapping

### 2.1 NATO Classification Levels (Complete Hierarchy)

| NATO Level | Abbreviation | Infrastructure Implications |
|---|---|---|
| NATO UNCLASSIFIED (NU) | NU | Standard IT controls |
| NATO RESTRICTED (NR) | NR | Encrypted storage and transport, access control |
| NATO CONFIDENTIAL (NC) | NC | Dedicated infrastructure, accredited systems |
| NATO SECRET (NS) | NS | Air-gapped or heavily isolated, TEMPEST, accredited facility |
| COSMIC TOP SECRET (CTS) | CTS | Bespoke isolated infrastructure, highest physical security |

This platform operates at the **NATO SECRET (NS)** level.

### 2.2 National Classification Equivalence

| NATO SECRET | Norway | Denmark | Germany |
|---|---|---|---|
| NS equivalent | HEMMELIG | HEMMELIGT | GEHEIM |

Each nation's data at the national SECRET equivalent must be treated under the respective national framework, not merely under NATO rules. NATO SECRET data is governed by NATO security policy (C-M(2002)49 as amended). National SECRET data is governed by the respective national authority.

### 2.3 Governing Authorities

| Nation | Authority | Primary Framework |
|---|---|---|
| Norway | NSM (Nasjonal Sikkerhetsmyndighet) | Sikkerhetsloven (2018), NSM Grunnprinsipper for IKT-sikkerhet |
| Denmark | CFCS (Center for Cybersikkerhed) | Danish Security Act, CFCS guidelines |
| Germany | BSI (Bundesamt fur Sicherheit in der Informationstechnik) | BSI IT-Grundschutz, BSI C5 |
| NATO | NCIA / AC/35 | C-M(2002)49, AC/35 series for CIS security |

### 2.4 Applicable International Frameworks

- **NATO**: C-M(2002)49 as amended, AC/35 series, STANAG 4774 (confidentiality metadata), STANAG 4778 (metadata binding)
- **EU/EEA**: NIS2 Directive (Norway via EEA, Denmark and Germany as EU members), GDPR, ENISA guidance
- **National**: Each nation's Security Act and its implementing regulations

---

## 3. Accreditation Strategy

### 3.1 Multi-Authority Accreditation

This platform requires accreditation from multiple authorities simultaneously:

1. **NATO accreditation**: NCIA is responsible for accreditation of NATO CIS systems. The NATO common enclave must be accredited for NS processing.
2. **Norwegian SAA**: NSM must approve the overall facility and the Norwegian national enclave, as well as the hosting of foreign national enclaves on Norwegian soil.
3. **Danish SAA**: CFCS must approve the Danish national enclave and the cross-domain mechanisms connecting it to the NATO common area.
4. **German SAA**: BSI must approve the German national enclave and its cross-domain interfaces.

Each nation's Security Accreditation Authority (SAA) must independently approve their respective enclave. A bilateral/multilateral security agreement (often a Programme Security Instruction or PSI) must be in place covering:

- Permission to process each nation's SECRET-level data in a Norwegian facility
- Cross-domain data sharing rules between enclaves
- Incident response and breach notification obligations across all parties
- Personnel clearance mutual recognition arrangements

### 3.2 Accreditation Lifecycle

Following the universal accreditation pattern:

1. **Categorize** -- NS for NATO common area; national SECRET equivalent for each enclave
2. **Select controls** -- Union of NATO AC/35, NSM Grunnprinsipper, CFCS guidelines, and BSI IT-Grundschutz requirements (most restrictive control wins for shared infrastructure)
3. **Implement** -- Build with controls in place from day one
4. **Assess** -- Each national authority assesses their enclave independently; NCIA assesses the NATO common area
5. **Authorize** -- Each authority grants their respective approval
6. **Monitor** -- Continuous compliance monitoring; significant changes trigger reaccreditation from all authorities

Design for step 6 from the start: automated compliance scanning (OpenSCAP with profiles mapped to each framework), continuous monitoring dashboards per enclave, and automated alerting on control deviations.

---

## 4. Physical Infrastructure and Facility

### 4.1 Facility Requirements

The LIST-X equivalent facility in Norway must meet the most stringent requirements across all four accrediting authorities:

- **Physical security**: Sikkerhetsgraderte omrader (security-graded areas) at HEMMELIG level per Virksomhetsikkerhetsforskriften, which is the Norwegian baseline. German and Danish physical security requirements must also be satisfied -- the most restrictive requirement applies.
- **TEMPEST**: NATO SDIP-27 Zone A or Zone B compliance depending on facility risk assessment. TEMPEST-rated equipment throughout. Shielded rooms (Faraday cages) for processing areas. Red/black separation enforced for all cabling.
- **Access control**: Multi-factor physical access control. Access logs retained per the longest national retention requirement. Escort procedures for uncleared maintenance personnel.
- **Personnel**: Only personnel with appropriate national clearances AND NATO SECRET clearance may access the common areas. Each national enclave requires the respective national clearance at SECRET level. Clearance mutual recognition must be formalized in the programme security agreement.

### 4.2 Physical Separation Model

```
+------------------------------------------------------------------+
|                    LIST-X Facility (Norway)                       |
|                                                                   |
|  +--------------------+  +--------------------+                   |
|  | Norwegian Enclave   |  | Danish Enclave     |                  |
|  | HEMMELIG             |  | HEMMELIGT          |                 |
|  | NSM-accredited       |  | CFCS-accredited    |                 |
|  +--------+------------+  +--------+-----------+                  |
|           |                         |                              |
|           | CDS                     | CDS                          |
|           |                         |                              |
|  +--------v-------------------------v-----------+                  |
|  |           NATO Common Area (NS)              |                  |
|  |           NCIA-accredited                    |                  |
|  +--------^-------------------------^-----------+                  |
|           |                         |                              |
|           | CDS                     | CDS                          |
|           |                         |                              |
|  +--------+------------+  +--------+-----------+                  |
|  | German Enclave       |  | Cross-Domain       |                 |
|  | GEHEIM               |  | Exchange Zone      |                 |
|  | BSI-accredited        |  | (Controlled xfer)  |                |
|  +---------------------+  +--------------------+                  |
|                                                                   |
+------------------------------------------------------------------+
```

Each enclave occupies physically separate rack space with independent power distribution. Cabling between enclaves is prohibited -- all inter-enclave communication flows through accredited cross-domain solutions (CDS).

---

## 5. Network Architecture

### 5.1 Network Isolation Model

The architecture enforces strict network separation between enclaves. There is **no direct network path** between any two national enclaves. All inter-enclave data flow transits the NATO Common Area through cross-domain solutions.

#### Per-Enclave Network Stack

Each enclave (Norwegian, Danish, German, NATO Common) has:

- **Dedicated physical switches**: Separate Cisco Nexus 9000 series spine-leaf fabric per enclave. No shared switching infrastructure between enclaves.
- **Independent management plane**: Each enclave has its own out-of-band management network (IPMI/iDRAC/CIMC), not interconnected.
- **Independent firewall**: Dedicated firewall appliance per enclave boundary.
- **Independent DNS/DHCP**: CoreDNS and Kea DHCP per enclave, no cross-enclave resolution.

#### Network addressing

Each enclave uses non-overlapping RFC 1918 address space. There is no routing between enclaves at the IP layer. Cross-domain solutions operate at the application layer with content inspection.

### 5.2 Cross-Domain Solutions (CDS)

Cross-domain data flow is the most critical architectural element. Every data transfer between enclaves must pass through an accredited CDS.

#### CDS Placement

Four CDS instances are required:

1. **Norway <-> NATO Common**: Bidirectional guard for Norwegian national data release to NATO and NATO data import to Norwegian enclave
2. **Denmark <-> NATO Common**: Bidirectional guard for Danish national data release to NATO and NATO data import to Danish enclave
3. **Germany <-> NATO Common**: Bidirectional guard for German national data release to NATO and NATO data import to German enclave
4. **NATO Common <-> Cross-Domain Exchange Zone**: For controlled release of sanitized/downgraded data (if required for lower-classification interfaces)

#### CDS Requirements

- **Hardware data diodes** for any one-way flows (e.g., threat intelligence feeds into enclaves): Advenica SecuriCDS, Owl Cyber Defense, or Fox-IT DataDiode
- **Content-inspecting guards** for bidirectional flows: These must inspect, log, and enforce data marking per STANAG 4774/4778
- **Transfer approval workflow**: Every cross-enclave transfer requires explicit authorization. Automated policy enforcement for routine transfers (e.g., NATO-marked data flowing to the NATO common area). Manual approval gates for national-to-NATO release and any downgrading.
- **Audit trail**: Every transfer logged with full chain-of-custody -- who authorized, what was transferred, source enclave, destination enclave, timestamp, content hash

#### CDS Accreditation

Each CDS must be independently accredited by:
- The national authority on each side of the boundary
- NCIA for any CDS touching the NATO common area
- All four authorities must agree on the data flow policies enforced by each CDS

### 5.3 TEMPEST and Emanation Security

- All network equipment meets NATO SDIP-27 requirements for the assessed zone
- Red/black separation enforced: classified (red) and unclassified (black) signals never share conductors, conduits, or equipment
- TEMPEST-rated KVM switches for any multi-enclave administrative workstations (if permitted by accreditation -- single-enclave workstations are preferred)
- Cable routing maintains required separation distances between enclave cabling runs
- NSM conducts TEMPEST inspection and certification for the Norwegian facility; German and Danish authorities verify their enclaves meet their respective TEMPEST standards

---

## 6. Compute Platform

### 6.1 Platform Selection

**RKE2 (Rancher Kubernetes Engine 2)** as the container orchestration platform across all enclaves, with KubeVirt for any VM workloads. Rationale:

- RKE2 is designed for security-focused and air-gapped environments
- FIPS-capable cryptographic modules available
- CIS Kubernetes Benchmark hardened by default
- Supports air-gapped installation with pre-pulled images
- Well-suited for classified Kubernetes deployments

Each enclave runs an **independent, fully isolated RKE2 cluster**. There is no multi-cluster federation or shared control plane between enclaves. The clusters are operationally independent.

### 6.2 Per-Enclave Compute Stack

Each enclave includes:

- **Bare-metal servers**: Dedicated rack-mount servers (e.g., Dell PowerEdge or HPE ProLiant with iDRAC/iLO on isolated management network)
- **RKE2 cluster**: Minimum 3 control plane nodes, N worker nodes sized to workload
- **KubeVirt**: For legacy VM workloads that cannot be containerized
- **Container registry**: Self-hosted Harbor instance per enclave with strict image provenance and signing (cosign/Sigstore with enclave-specific signing keys)
- **Storage**: Rook-Ceph per enclave for block and object storage, with encryption at rest using nationally-approved cryptographic modules
- **Networking**: Cilium as CNI with mandatory default-deny network policies

### 6.3 Hardening Baseline

The hardening baseline is a **composite** derived from all applicable frameworks, with the most restrictive control winning for shared infrastructure components:

1. **Primary (per enclave)**:
   - Norwegian enclave: NSM Grunnprinsipper for IKT-sikkerhet
   - Danish enclave: CFCS guidelines
   - German enclave: BSI IT-Grundschutz Bausteine and BSI TR series
   - NATO common: AC/35 series requirements
2. **Technical implementation**: CIS Kubernetes Benchmark applied via kube-bench, supplemented by national-specific controls where CIS is insufficient
3. **Gap analysis**: Where national frameworks do not provide specific technical guidance for a component, CIS Benchmarks and NIST 800-190 (container security) serve as supplementary references
4. **Automated compliance**: OpenSCAP profiles mapped to each national framework, with per-enclave compliance dashboards

NSM Grunnprinsipper mapping for the Norwegian enclave (and as the host-nation baseline for shared facility infrastructure):

- **Identifisere (Identify)**: Asset inventory per enclave, risk assessment per national framework, threat intelligence from NorCERT
- **Beskytte (Protect)**: Network segmentation (enclave isolation), access control (clearance-based), encryption (NSM-approved crypto), secure configuration (composite hardening baseline)
- **Oppdage (Detect)**: Continuous monitoring per enclave, anomaly detection, security event logging to per-enclave SIEM
- **Handtere (Respond)**: Incident response plan spanning all four authorities, NorCERT coordination for Norwegian facility

---

## 7. Cryptography

### 7.1 Crypto Separation

This is a critical architectural constraint: **different cryptographic products are required for different data types**.

| Data Type | Required Crypto Approval |
|---|---|
| NATO SECRET data | NATO-approved cryptographic products only |
| Norwegian HEMMELIG data | NSM-approved cryptographic products |
| Danish HEMMELIGT data | CFCS-approved cryptographic products |
| German GEHEIM data | BSI-approved cryptographic products |

National crypto products are **NOT** acceptable for NATO-classified data, even if approved by the national authority. Conversely, NATO-approved crypto may or may not satisfy national requirements -- each SAA must confirm.

### 7.2 Crypto Implementation

- **Encryption at rest**: Each enclave uses its nationally-approved encryption for local storage (LUKS/dm-crypt with approved crypto modules). The NATO common area uses NATO-approved encryption.
- **Encryption in transit**: All intra-enclave communication uses TLS with nationally-approved or NATO-approved cipher suites. There is no cross-enclave network traffic (CDS handles the boundary).
- **Key management**: Each enclave operates independent key management infrastructure. No shared key material between enclaves. Hardware Security Modules (HSMs) per enclave where required by the national framework.
- **etcd encryption**: RKE2 etcd encryption at rest enabled in every cluster with enclave-appropriate crypto.

---

## 8. Identity and Access Management

### 8.1 Per-Enclave Identity

Each enclave runs an independent identity provider:

- **Norwegian enclave**: FreeIPA or national IdP integration, NSM-compliant authentication
- **Danish enclave**: Independent IdP per CFCS requirements
- **German enclave**: Independent IdP per BSI requirements
- **NATO common area**: Independent IdP aligned to NATO identity standards

There is **no identity federation between enclaves**. A user with access to the Norwegian enclave does not automatically have access to the NATO common area or any other enclave. Access to each enclave requires:

1. Appropriate national clearance at SECRET level
2. NATO SECRET clearance (for NATO common area)
3. Explicit need-to-know for the specific project data in that enclave
4. MFA authentication per the enclave's IdP

### 8.2 Personnel Constraints

- Only Norwegian-cleared personnel at HEMMELIG level may administer the Norwegian enclave
- Only Danish-cleared personnel at HEMMELIGT level may administer the Danish enclave
- Only German-cleared personnel at GEHEIM level may administer the German enclave
- NATO common area administration requires NATO SECRET clearance; the host nation (Norway) typically provides the primary operations team, but all three nations may have administrative personnel with appropriate clearances
- On-call rotations must account for clearance requirements -- uncleared personnel cannot perform emergency maintenance on classified systems
- Vendor/contractor access requires appropriate clearances and escort procedures defined in the programme security agreement

---

## 9. Data Flow Architecture

### 9.1 Data Classification and Marking

All data must be marked per STANAG 4774 (confidentiality metadata) and STANAG 4778 (metadata binding) for NATO data. National data must be marked per the respective national marking scheme.

Data categories on the platform:

1. **NATO SECRET (NS)**: Originates from NATO or is explicitly released to NATO by a participating nation. Processed in the NATO common area.
2. **Norwegian national HEMMELIG**: Norwegian-origin data not released to NATO. Processed only in the Norwegian enclave.
3. **Danish national HEMMELIGT**: Danish-origin data not released to NATO. Processed only in the Danish enclave.
4. **German national GEHEIM**: German-origin data not released to NATO. Processed only in the German enclave.
5. **NATO SECRET -- releasable to [nation(s)]**: NATO data with specific releasability markings, processed in the NATO common area but transferable to designated national enclaves via CDS.

### 9.2 Data Flow Rules

```
Rule 1: National data NEVER flows directly between national enclaves
        (NO: Norway -> Denmark, Norway -> Germany, Denmark -> Germany)

Rule 2: National data flows to NATO common area ONLY after explicit
        national release review and CDS transfer with approval

Rule 3: NATO data flows to a national enclave ONLY if releasability
        markings permit AND CDS policy enforcement confirms

Rule 4: Data downgrading (SECRET -> RESTRICTED or below) requires
        formal sanitization review and release authority approval

Rule 5: All cross-boundary transfers are logged, hashed, and
        attributable to an individual authorizer
```

### 9.3 Collaborative Workflow Pattern

For multinational collaboration on NATO SECRET material:

1. Each nation prepares nationally-originated inputs in their national enclave
2. National authority reviews and authorizes release to NATO (marking changes to NS or NS REL NO/DK/DE)
3. Released data transfers through CDS to the NATO common area
4. Multinational team collaborates on NATO-marked data in the NATO common area
5. Results marked as NATO SECRET are available in the common area
6. If results need to flow back to a national enclave, CDS enforces releasability markings

This ensures that national intelligence sources and methods embedded in national-only data are never exposed to other nations, while enabling genuine collaboration on shared NATO material.

---

## 10. Air-Gapped Operations

### 10.1 Disconnected Infrastructure

The entire platform is air-gapped from the internet and from any lower-classification network. There is no direct or indirect network path to unclassified networks.

### 10.2 Software Supply Chain

- **Disconnected container registries**: Harbor per enclave, populated via offline sync with verified image bundles
- **Disconnected package repositories**: Local mirrors (Pulp or Aptly) per enclave for OS packages, populated via approved media import
- **Offline Kubernetes deployment**: Pre-pulled images, Helm chart bundles, Hauler for artifact transport
- **Offline Ansible execution**: Bundled collections with local Galaxy mirrors per enclave
- **Artifact validation**: All imported media verified via checksums, GPG signatures, and SBOM verification. Chain-of-custody documentation for all imported artifacts.
- **Import process**: Dedicated media import station per enclave with malware scanning (using approved scanning tools) before artifacts are admitted to the enclave network. Import requires authorization and is logged.

### 10.3 Patch Management

- Security patches assessed on an unclassified development/test environment first
- Approved patches packaged into verified bundles
- Bundles imported via the controlled media import process
- Staged rollout: test nodes first, then production, with rollback capability
- Patch status tracked per enclave with compliance reporting to the respective national authority

---

## 11. Monitoring and Observability

### 11.1 Per-Enclave Monitoring Stack

Each enclave runs an independent monitoring stack with no cross-enclave data flow:

- **Metrics**: Prometheus with per-enclave Alertmanager
- **Logs**: Loki or ELK stack per enclave for log aggregation
- **Dashboards**: Grafana per enclave
- **Security monitoring**: Wazuh or equivalent SIEM per enclave, with Falco for container runtime security and Tetragon for kernel-level observability
- **Audit logging**: All administrative actions logged with individual attribution, retained per the longest applicable retention requirement across the governing frameworks
- **Hardware health**: IPMI/Redfish monitoring per enclave management network

### 11.2 Facility-Level Monitoring

Physical facility monitoring (power, cooling, physical access) operates on a separate network from all classified enclaves. Physical security monitoring feeds into the facility security officer's systems, not into the classified enclaves.

### 11.3 Compliance Dashboards

Each enclave presents a compliance dashboard showing:

- OpenSCAP scan results mapped to the applicable national framework
- CIS Benchmark scan results (kube-bench for Kubernetes)
- Patch compliance status
- Certificate expiry tracking
- Access control audit summary
- CDS transfer log summary

---

## 12. Disaster Recovery and Business Continuity

### 12.1 RTO/RPO Targets

Defined per the programme security agreement, but typical for NS-level systems:

- **RPO**: 1 hour (data loss tolerance)
- **RTO**: 4 hours (time to restore service)

### 12.2 Backup Architecture

- **Per-enclave backups**: Velero for Kubernetes resources, Ceph snapshot-based backup for persistent volumes
- **Backup encryption**: Per-enclave nationally-approved crypto for backup encryption
- **Backup storage**: On-site backup to separate storage within the same facility (classified backups cannot leave the accredited facility without explicit authorization)
- **Off-site DR**: If required, a secondary accredited facility must be identified and accredited by all four authorities. This is a significant lead-time item.
- **Backup testing**: Regular restore tests per enclave, documented and reported to the respective national authority

### 12.3 Failover

Within each enclave:
- RKE2 control plane HA (3 control plane nodes minimum)
- Ceph replication factor 3 for storage resilience
- Automated node failover within the enclave
- No cross-enclave failover (each enclave is independently resilient)

---

## 13. Infrastructure as Code and Automation

### 13.1 Per-Enclave Automation

Each enclave is managed via GitOps with independent toolchains:

- **ArgoCD**: Per-enclave ArgoCD instance for Kubernetes GitOps
- **Ansible**: Per-enclave AWX/AAP for configuration management and day-2 operations, with Ansible Vault for secrets
- **OpenTofu**: Per-enclave state management for infrastructure provisioning
- **Git repositories**: Per-enclave Gitea or GitLab instance (air-gapped) holding all IaC, with signed commits

### 13.2 Consistency Across Enclaves

While each enclave is operationally independent, shared infrastructure patterns (Ansible roles, Helm charts, OpenTofu modules) can be developed on an unclassified development environment and imported into each enclave via the controlled media import process. This ensures consistency without creating cross-enclave dependencies.

---

## 14. Implementation Phasing

### Phase 1: Foundation (Months 1-4)
- Facility preparation and TEMPEST assessment
- Programme security agreement finalization between Norway, Denmark, and Germany
- Network infrastructure installation (per-enclave spine-leaf fabrics)
- Physical separation verification by all four authorities

### Phase 2: NATO Common Area (Months 3-6)
- NATO common area compute and storage deployment
- RKE2 cluster deployment and hardening
- CDS procurement and initial configuration
- NCIA accreditation assessment begins

### Phase 3: National Enclaves (Months 5-9)
- Norwegian enclave deployment (NSM assessment)
- Danish enclave deployment (CFCS assessment)
- German enclave deployment (BSI assessment)
- Per-enclave identity, monitoring, and automation stack deployment

### Phase 4: Cross-Domain Integration (Months 8-12)
- CDS accreditation by all relevant authorities
- Cross-domain data flow testing with test data
- Transfer approval workflow validation
- End-to-end collaborative workflow testing

### Phase 5: Operational Acceptance (Months 11-14)
- Final accreditation reviews by all four authorities
- Operational readiness assessment
- Operations team handover and training
- Continuous monitoring baseline established

---

## 15. Key Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Multi-authority accreditation delay | Schedule slip | Engage all four SAAs from Phase 1; parallel accreditation tracks; dedicated accreditation liaison per nation |
| CDS procurement lead time | Cross-domain sharing delayed | Procure CDS early (Phase 2); have fallback to manual (sneakernet) transfer with approval workflows |
| Crypto product incompatibility | Cannot meet all national + NATO crypto requirements simultaneously | Separate crypto per enclave from day one; validate approved product lists with each authority before procurement |
| Personnel clearance bottlenecks | Cannot staff operations teams | Begin clearance processes 12+ months before operational need; leverage existing cleared personnel from participating nations |
| Divergent national hardening requirements | Compliance complexity | Composite hardening baseline with per-enclave delta; automated compliance scanning maps to each framework independently |
| Supply chain integrity concerns | Accreditation failure | Establish trusted procurement channels per each national framework; chain-of-custody documentation for all hardware and software; firmware verification on delivery |

---

## 16. Architectural Decision Records

### ADR-001: Independent clusters per enclave (not multi-tenant)
- **Decision**: Each enclave runs a fully independent RKE2 cluster rather than using namespace-based multi-tenancy on a shared cluster.
- **Rationale**: At SECRET level, logical separation (namespaces, network policies) is insufficient. Physical and cryptographic separation between nationally-classified enclaves is required by all four accrediting authorities. Shared kernel, shared etcd, and shared control plane create unacceptable risk of cross-enclave data leakage.

### ADR-002: No identity federation between enclaves
- **Decision**: Each enclave has an independent IdP with no federation.
- **Rationale**: Federation creates implicit trust relationships between enclaves. Each nation must independently control access to their national data. Cross-enclave collaboration happens through CDS-mediated data transfer, not through shared identity.

### ADR-003: CDS for all inter-enclave communication
- **Decision**: All data flow between enclaves passes through accredited cross-domain solutions with content inspection.
- **Rationale**: This is the only mechanism acceptable to all four accrediting authorities for cross-boundary data transfer at SECRET level. Direct network connectivity between enclaves is prohibited.

### ADR-004: NSM Grunnprinsipper as facility baseline, with national overlays
- **Decision**: Since the facility is in Norway, NSM Grunnprinsipper for IKT-sikkerhet serves as the baseline for shared facility infrastructure (physical security, facility management systems). Each national enclave applies its own national framework as the primary hardening baseline within that enclave.
- **Rationale**: The host nation's framework governs the facility. Each nation retains sovereignty over the security controls applied to their national data.

### ADR-005: RKE2 as container platform
- **Decision**: RKE2 across all enclaves.
- **Rationale**: Designed for air-gapped and security-focused deployments, CIS hardened by default, FIPS-capable, strong air-gapped installation support. Provides consistency across enclaves while allowing per-enclave independent operation. OpenShift was considered but introduces additional licensing complexity in a multinational context.

### ADR-006: FLOSS-first where accreditation permits
- **Decision**: Prefer FLOSS components (RKE2, Cilium, Rook-Ceph, Harbor, ArgoCD, Prometheus/Grafana, FreeIPA, AWX, OpenTofu) over proprietary alternatives.
- **Rationale**: Reduces licensing complexity across three nations and NATO. Source code availability supports security audit requirements from all four authorities. Avoids vendor lock-in in a multinational context where procurement rules differ by nation.

---

## 17. Summary

This architecture delivers a NATO SECRET-capable multinational platform through strict physical and cryptographic separation of national enclaves, mediated by accredited cross-domain solutions for controlled data sharing. The design respects the sovereignty of each participating nation's security framework while enabling genuine collaboration on NATO-classified material. The most critical success factors are: early and continuous engagement with all four accrediting authorities, realistic timelines for multi-authority accreditation, and disciplined adherence to the principle that classification drives architecture -- not the other way around.
