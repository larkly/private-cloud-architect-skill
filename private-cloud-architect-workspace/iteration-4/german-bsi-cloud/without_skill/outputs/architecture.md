# Private Cloud Architecture for VS-NfD Processing
## Bundesbehoerde — BSI IT-Grundschutz & C5 Attestation

---

## 1. Executive Summary

This document describes a reference architecture for a private cloud operated by a German federal agency (Bundesbehoerde) to process data classified as **VS-NfD** (Verschlusssache — Nur fuer den Dienstgebrauch). The architecture is based on OpenStack with FLOSS components where permissible and BSI-approved products where mandated. It addresses compliance with **BSI IT-Grundschutz** (based on the IT-Grundschutz-Kompendium), the **BSI C5** (Cloud Computing Compliance Criteria Catalogue) attestation, and the **Verschlusssachenanweisung (VSA)** requirements for handling classified information.

All data processing and storage occurs exclusively within the Federal Republic of Germany.

---

## 2. Regulatory and Compliance Framework

### 2.1 Applicable Regulations

| Regulation / Standard | Relevance |
|---|---|
| **VSA (Verschlusssachenanweisung)** | Governs handling of classified material; VS-NfD is the lowest classification tier but still carries mandatory technical controls |
| **BSI IT-Grundschutz-Kompendium** | Mandatory for federal agencies (per UP Bund — Umsetzungsplan Bund); defines baseline security controls |
| **BSI C5:2020** (Cloud Computing Compliance Criteria Catalogue) | Required for cloud services used by federal agencies; Type 2 attestation demonstrates operational effectiveness |
| **BSI TR-02102** (Technische Richtlinie — Kryptographische Verfahren) | Specifies approved cryptographic algorithms and key lengths |
| **BSI-approved products (BSI-Zulassung)** | For VS-NfD, certain security functions (especially encryption of data in transit across uncontrolled networks) require products with BSI approval ("Freigabe" or "Zulassung") |
| **BDSG / DSGVO** | General data protection; relevant for any personal data co-processed |
| **SGB (if applicable)** | If processing social data, additional constraints apply |

### 2.2 Key Distinction: FLOSS vs. BSI-Approved Products

The critical question is **where FLOSS is sufficient and where BSI-approved products are mandatory**:

**FLOSS is permissible for:**
- Hypervisor (KVM)
- Cloud management layer (OpenStack services)
- Operating systems (hardened Linux distributions, e.g., RHEL or a BSI-SLZ-approved variant)
- Container runtime, orchestration (if used)
- Monitoring, logging, configuration management
- Internal network switching and routing (software-defined networking)
- Storage backends (Ceph, etc.)

**BSI-approved products are required for:**
- **VPN / network encryption** for VS-NfD data crossing zone boundaries or leaving physically controlled areas — BSI requires products with "VS-NfD-Freigabe" (e.g., genuscreen, SINA Box, secunet SINA, NCP VS GovNet Connector). FLOSS VPN solutions such as WireGuard or OpenVPN do **not** have BSI approval for VS-NfD.
- **Disk/volume encryption at rest** when media could leave physically secured areas — BSI-approved full-disk encryption (e.g., secunet sure, Rohde & Schwarz SITLine) or hardware-based encryption may be required depending on risk analysis. Within a physically secured server room (Sicherheitsbereich Zone 2+), software encryption with BSI TR-02102-compliant algorithms may suffice per risk assessment.
- **Smartcard / token-based authentication** for administrative access — BSI-approved smartcard readers and eID solutions.
- **Hardware Security Modules (HSM)** for key management — while not always strictly mandated for VS-NfD, using a BSI-certified or Common Criteria-certified HSM (e.g., Utimaco, Thales Luna with BSI certification) is strongly recommended and may be required by the accrediting authority (Geheimschutzbeauftragter).

**Grey area (risk-assessment-dependent):**
- **Firewall** — BSI-certified firewalls (genuscreen, secunet) are preferred but for purely internal segmentation within a secured zone, a hardened FLOSS firewall (e.g., nftables with BSI-hardened Linux) may be acceptable after risk assessment and approval by the BSI.
- **IDS/IPS** — FLOSS tools (Suricata, OSSEC) are generally acceptable but must be integrated into the agency's SIEM and meet IT-Grundschutz requirements.

---

## 3. Architecture Overview

### 3.1 Physical Architecture

```
+================================================================+
|                   PHYSICALLY SECURED DATA CENTER                |
|                  (Sicherheitsbereich Zone 2/3)                  |
|                  Location: Germany (mandatory)                  |
|                                                                 |
|  +---------------------------+  +---------------------------+   |
|  |   Management Zone (Mgmt)  |  |   Production Zone (Prod)  |   |
|  |                           |  |                           |   |
|  |  - OpenStack Controllers  |  |  - Compute Nodes (KVM)   |   |
|  |  - Ansible/Salt Masters   |  |  - Ceph Storage Cluster  |   |
|  |  - Monitoring (Prometheus)|  |  - Tenant Workloads       |   |
|  |  - SIEM Collector         |  |  - Internal SDN (OVN)    |   |
|  |  - Barbican + HSM         |  |                           |   |
|  +---------------------------+  +---------------------------+   |
|                                                                 |
|  +---------------------------+  +---------------------------+   |
|  |   DMZ / Transit Zone      |  |   Storage Zone            |   |
|  |                           |  |                           |   |
|  |  - BSI-approved VPN GW    |  |  - Ceph OSD Nodes        |   |
|  |    (SINA / genuscreen)    |  |  - Backup Targets         |   |
|  |  - Reverse Proxies        |  |  - Object Storage (Swift/ |   |
|  |  - BSI-approved Firewall  |  |    RadosGW)               |   |
|  +---------------------------+  +---------------------------+   |
|                                                                 |
+================================================================+
         |                                          |
    [BSI-approved VPN]                    [Physically isolated or
    to agency WAN /                        BSI-approved encrypted
    NdB (Netze des Bundes)]               backup replication]
```

### 3.2 Network Architecture

#### Zone Model (aligned with BSI NET.1.1)

| Zone | Trust Level | Connectivity | Encryption Requirement |
|---|---|---|---|
| **Production** | High | Internal only | Encryption within zone optional (physically secured); encrypted if crossing zone boundary |
| **Management** | Highest | Isolated management network; jump host access only | mTLS between services; SSH with smartcard auth |
| **Storage** | High | Internal only (dedicated storage VLAN) | Ceph encryption at rest (dm-crypt); Ceph messenger v2 (on-wire encryption) |
| **DMZ/Transit** | Medium | Connects to NdB / agency WAN | **BSI-approved VPN mandatory** (VS-NfD-freigegebene Produkte) |
| **External** | Untrusted | Internet (if any exposure) | Must traverse DMZ + BSI-approved encryption |

#### Software-Defined Networking

- **OVN (Open Virtual Network)** as the Neutron backend — FLOSS, well-integrated with OpenStack
- Microsegmentation via OVN security groups and ACLs
- **No tenant traffic leaves the physically secured zone** without passing through BSI-approved encryption
- Dedicated physical NICs for management, storage, and tenant traffic (no convergence for VS-NfD)

### 3.3 Compute Layer

| Component | Choice | Rationale |
|---|---|---|
| Hypervisor | **KVM** (FLOSS) | Integrated in Linux kernel; BSI IT-Grundschutz SYS.1.5 applies; widely used in government deployments |
| Compute management | **OpenStack Nova** | Industry-standard orchestration for KVM |
| Host OS | **RHEL 9** or **SLES 15** (with BSI hardening per SiG) | Commercial Linux with long-term support; BSI provides hardening guides (SiSyPHuS); alternatively, a BSI-approved Linux if mandated by accrediting authority |
| Secure Boot | **UEFI Secure Boot + TPM 2.0** | Ensures boot chain integrity; TPM for measured boot and disk encryption key sealing |
| Hardware | Server hardware from **BSI-recommended vendors** or Common Criteria-certified platforms | Avoid hardware with known supply-chain risks; consider EU-manufactured options |

### 3.4 Storage Layer

| Component | Choice | Rationale |
|---|---|---|
| Block/Object Storage | **Ceph** (FLOSS) | Mature, scalable, integrated with OpenStack (Cinder, Glance, Manila) |
| Encryption at rest | **dm-crypt (LUKS)** on OSD volumes | BSI TR-02102-compliant algorithm (AES-256); keys managed via Barbican + HSM |
| Ceph on-wire encryption | **Ceph msgr2 with encryption enabled** | Protects storage replication traffic |
| Backup | **Restic or BorgBackup** to offline/air-gapped targets | Encrypted backups; backup media stay within Sicherheitsbereich |

### 3.5 Identity and Access Management

| Component | Choice | Rationale |
|---|---|---|
| OpenStack Identity | **Keystone** with LDAP backend | Federation with existing agency directory (Active Directory or 389DS) |
| MFA | **Smartcard / eID** (BSI-approved reader) | Mandatory for admin access to VS-NfD systems per VSA |
| SSH access | **Certificate-based SSH + smartcard** | No password-based SSH; jump host architecture |
| RBAC | **OpenStack policy.yaml** + **Keystone domains** | Least-privilege per IT-Grundschutz ORP.4 |
| Secrets management | **Barbican** + **HSM** (Utimaco or similar) | Centralized key management; HSM for root-of-trust |

### 3.6 Key Management and Cryptography

This is one of the most critical areas for VS-NfD compliance:

1. **All cryptographic implementations must comply with BSI TR-02102** (Parts 1-4):
   - AES-256 for symmetric encryption
   - RSA >= 3000 bit or ECDSA with brainpoolP256r1 or higher (BSI-preferred curves, not NIST P-256 alone)
   - TLS 1.2 minimum (TLS 1.3 preferred) with BSI-approved cipher suites
   - SHA-256 minimum for hashing

2. **HSM (Hardware Security Module)**:
   - Recommended: Utimaco CryptoServer or Thales Luna Network HSM with Common Criteria EAL4+ certification
   - Integrated with OpenStack Barbican via PKCS#11 backend
   - Stores root keys for Ceph encryption, TLS certificate private keys, and Keystone signing keys

3. **Certificate Authority**:
   - Internal PKI (e.g., EJBCA or step-ca) for internal service certificates
   - All internal communication uses mTLS where technically feasible

### 3.7 OpenStack Services Deployment

| Service | Purpose | Notes |
|---|---|---|
| **Keystone** | Identity & Auth | LDAP-backed; MFA-enabled |
| **Nova** | Compute | KVM backend; CPU pinning, NUMA-aware for sensitive workloads |
| **Neutron + OVN** | Networking | Microsegmentation; no external provider networks without VPN |
| **Cinder** | Block Storage | Ceph RBD backend; volume encryption via Barbican |
| **Glance** | Image Service | Ceph backend; signed images only |
| **Barbican** | Key Management | HSM-backed; PKCS#11 |
| **Horizon** | Dashboard | TLS-only; optional if API-only operations preferred |
| **Heat** | Orchestration | Infrastructure-as-Code for reproducible deployments |
| **Octavia** | Load Balancing | Internal LBs only; HAProxy-based |
| **Designate** | DNS | Internal zones only |
| **Ironic** | Bare Metal | Optional: for workloads requiring dedicated hardware (higher assurance) |

**Deployment method**: **Kolla-Ansible** or **TripleO/Director** for reproducible, auditable deployments. All deployment configurations stored in version control (Git) with signed commits.

---

## 4. BSI IT-Grundschutz Compliance Mapping

### 4.1 Relevant Bausteine (Building Blocks)

The following IT-Grundschutz Kompendium building blocks are directly applicable:

| Baustein | Title | Application |
|---|---|---|
| **OPS.1.1.3** | Patch- und Aenderungsmanagement | Automated patching pipeline with testing |
| **OPS.1.1.4** | Schutz vor Schadprogrammen | ClamAV/YARA on management nodes; host-based IDS |
| **OPS.1.2.5** | Fernwartung | BSI-approved remote admin (VPN + jump host + smartcard) |
| **OPS.2.2** | Cloud-Nutzung | Applies even for private cloud; C5 alignment |
| **SYS.1.1** | Allgemeiner Server | Baseline hardening for all hosts |
| **SYS.1.3** | Server unter Linux | Linux-specific hardening (CIS + BSI SiSyPHuS) |
| **SYS.1.5** | Virtualisierung | KVM hypervisor security requirements |
| **SYS.1.6** | Containerisierung | If containers are used (Kubernetes/Podman) |
| **NET.1.1** | Netzarchitektur und -design | Zone model, segmentation |
| **NET.1.2** | Netzmanagement | Management network isolation |
| **NET.3.1** | Router und Switches | Network device hardening |
| **NET.3.2** | Firewall | Firewall architecture and rules |
| **INF.2** | Rechenzentrum | Physical data center security |
| **INF.12** | Verkabelung | Cabling security |
| **CON.1** | Kryptokonzept | Cryptographic concept document (mandatory) |
| **APP.4.4** | Kubernetes | If K8s is deployed on top |
| **DER.1** | Detektion von sicherheitsrelevanten Ereignissen | SIEM / SOC requirements |
| **DER.2.1** | Behandlung von Sicherheitsvorfaellen | Incident response |
| **DER.4** | Notfallmanagement | Business continuity / disaster recovery |

### 4.2 Schutzbedarf (Protection Requirement)

For VS-NfD processing, the **Schutzbedarf** (protection requirement) is at least **hoch (high)** for confidentiality. This means:

- Standard IT-Grundschutz baseline alone is **not sufficient** — additional risk analysis (Risikoanalyse) is required per BSI-Standard 200-3
- An explicit **Sicherheitskonzept** (security concept) per BSI-Standard 200-2 must be created
- The Sicherheitskonzept must be approved by the agency's **IT-Sicherheitsbeauftragter (CISO)** and the **Geheimschutzbeauftragter**

---

## 5. BSI C5 Attestation Process

### 5.1 C5:2020 Overview

The C5 catalogue defines **125 criteria** across 17 domains. For a private cloud operated by a federal agency, a **C5 Type 2 attestation** is the target, demonstrating that controls have been effectively operating over a defined period (typically 6-12 months).

### 5.2 C5 Domains and Key Requirements

| C5 Domain | Key Controls for This Architecture |
|---|---|
| **SP (Sicherheitsrichtlinien)** | Documented cloud security policy |
| **OIS (Organisation der Informationssicherheit)** | ISMS roles, CISO appointment |
| **PS (Personal)** | Security clearance for admins (Sicherheitsueberpruefung SU1 minimum for VS-NfD) |
| **AM (Asset Management)** | CMDB for all cloud components |
| **PHY (Physische Sicherheit)** | Data center physical controls |
| **OPS (Betriebssicherheit)** | Change management, capacity planning, backup |
| **IDM (Identitaets- und Berechtigungsmanagement)** | MFA, RBAC, recertification |
| **CRY (Kryptographie)** | BSI TR-02102 compliance, key lifecycle |
| **KOM (Kommunikationssicherheit)** | Network segmentation, encryption in transit |
| **PI (Portabilitaet und Interoperabilitaet)** | Standard APIs (OpenStack APIs satisfy this) |
| **ENT (Beschaffung und Entwicklung)** | Secure SDLC, supply chain |
| **LFR (Lieferantenmanagement)** | Vendor risk for hardware and support contracts |
| **BEI (Umgang mit Sicherheitsvorfaellen)** | Incident response, CERT-Bund notification |
| **BCM (Geschaeftskontinuitaet)** | DR plan, RTO/RPO definitions |
| **COM (Compliance)** | Regulatory mapping, audit trails |
| **MON (Monitoring)** | Logging, SIEM, anomaly detection |
| **MDV (Umgang mit Datentraegern)** | Secure media handling, destruction |

### 5.3 Attestation Roadmap

```
Phase 1: Preparation (Months 1-6)
  |
  |-- Establish ISMS (BSI-Standard 200-1)
  |-- Create Sicherheitskonzept (BSI-Standard 200-2)
  |-- Conduct Risikoanalyse (BSI-Standard 200-3)
  |-- Map IT-Grundschutz Bausteine to architecture
  |-- Identify and procure BSI-approved products
  |-- Begin physical infrastructure build-out
  |
Phase 2: Implementation (Months 4-12)
  |
  |-- Deploy and harden infrastructure
  |-- Implement all technical controls
  |-- Deploy OpenStack with security configurations
  |-- Integrate BSI-approved VPN/encryption
  |-- Configure monitoring, logging, SIEM
  |-- Staff security clearances (SU1/SU2)
  |
Phase 3: Operational Maturity (Months 10-16)
  |
  |-- Operate cloud platform under full controls
  |-- Internal audits against C5 criteria
  |-- Remediate findings
  |-- Document operational evidence
  |-- Minimum 6 months of operational evidence for Type 2
  |
Phase 4: C5 Attestation (Months 16-20)
  |
  |-- Engage BSI-recognized auditor (Wirtschaftspruefer)
  |     (e.g., PwC, KPMG, Deloitte with BSI C5 experience)
  |-- Auditor performs ISAE 3402 / C5 examination
  |-- Receive attestation report (Pruefbericht)
  |-- Address any qualifications
  |
Phase 5: Continuous Compliance (Ongoing)
  |
  |-- Annual re-attestation
  |-- Continuous monitoring and improvement
  |-- BSI IT-Grundschutz re-certification (every 3 years)
```

### 5.4 Parallel: IT-Grundschutz Certification

In addition to C5, the agency should pursue **ISO 27001 certification on the basis of IT-Grundschutz** (commonly called "BSI-Grundschutz-Zertifikat"). This involves:

1. **Audit by a BSI-certified auditor** (BSI-zertifizierter IS-Revisor/Auditor)
2. **Submission to BSI** for certificate issuance
3. **Certificate validity**: 3 years with annual surveillance audits

The IT-Grundschutz certification and C5 attestation have significant overlap — plan them in parallel to reduce effort.

---

## 6. VS-NfD-Specific Requirements

### 6.1 Personnel

- All administrators with access to VS-NfD systems must hold a valid **Sicherheitsueberpruefung (SU1)** minimum
- The agency's **Geheimschutzbeauftragter** must approve the system and the operating personnel
- **Need-to-know** principle strictly enforced

### 6.2 Physical Security

- Server rooms must meet **Zone 2** requirements (Sicherheitsbereich) at minimum
- Access control: electronic access with logging, minimum two-person rule for critical operations
- No unauthorized devices (mobile phones, USB drives) in server rooms
- Cabling must be protected against interception (INF.12)

### 6.3 Data Handling

- VS-NfD data must **never** leave German territory
- All backup media must remain within physically secured areas
- Decommissioned storage media must be destroyed per **DIN 66399** (security level P-4 minimum for VS-NfD, or degaussing/shredding for HDDs)
- No cloud federation or data replication to external/foreign sites

### 6.4 Network Boundaries

- **Any VS-NfD data traversing networks outside the physically secured zone must be encrypted with BSI-approved (VS-NfD-freigegebene) products**
- Connection to **Netze des Bundes (NdB)** via BSI-approved VPN gateways
- Internet connectivity (if any) must be mediated through a **Demilitarized Zone** with BSI-approved firewalls
- Consider **SINA Workstations** for administrative access from outside the secure zone

---

## 7. FLOSS vs. Proprietary Decision Matrix

| Layer | FLOSS Viable? | Recommendation | Notes |
|---|---|---|---|
| Hypervisor (KVM) | Yes | **FLOSS** | Proven in BSI-audited environments |
| Host OS (Linux) | Yes | **FLOSS** (RHEL/SLES with BSI hardening) | Must follow BSI SiSyPHuS hardening |
| Cloud Platform (OpenStack) | Yes | **FLOSS** | Dominant in European gov clouds |
| SDN (OVN) | Yes | **FLOSS** | Internal networking only |
| Storage (Ceph) | Yes | **FLOSS** | With dm-crypt encryption |
| Key Mgmt (Barbican) | Yes | **FLOSS** (with approved HSM) | Barbican is FLOSS; HSM is proprietary but certified |
| Monitoring (Prometheus/Grafana) | Yes | **FLOSS** | Standard stack |
| SIEM (Wazuh/ELK) | Yes | **FLOSS** | Or commercial (Splunk) per preference |
| VPN Gateway | **No** | **BSI-approved** (SINA, genuscreen) | Mandatory for VS-NfD data in transit |
| Perimeter Firewall | Conditional | **BSI-approved preferred** | Risk assessment may allow FLOSS for internal segmentation only |
| Admin Authentication | Conditional | **BSI-approved smartcard** | Reader/token must be approved; backend can be FLOSS (FreeIPA + SSSD) |
| HSM | **No** | **CC-certified / BSI-approved** | Root of trust must be hardware-certified |
| Disk Encryption (at zone boundary) | Conditional | **BSI-approved if media leaves zone** | dm-crypt acceptable within secured zone |

**Bottom line**: Approximately 80% of the stack can be FLOSS. The remaining 20% — primarily network boundary encryption, HSMs, and admin authentication tokens — requires BSI-approved products. This is a significant cost and complexity saving compared to a fully proprietary stack.

---

## 8. Operational Architecture

### 8.1 Monitoring and Logging

```
Compute/Storage Nodes                 Management Zone
+-------------------+               +---------------------------+
| node_exporter     |--metrics----->| Prometheus                |
| ceph_exporter     |               | Grafana Dashboards        |
| libvirt_exporter  |               |                           |
|                   |               | Loki (Log Aggregation)    |
| rsyslog/journald  |--logs------->| or Elasticsearch          |
| auditd            |               |                           |
| Wazuh agent       |--alerts----->| Wazuh Manager / SIEM      |
+-------------------+               | --> CERT-Bund reporting   |
                                     +---------------------------+
```

- **auditd** on all hosts with rules aligned to IT-Grundschutz DER.1
- **Log retention**: minimum 12 months online, 5 years archive (per BSI recommendation)
- **Tamper-proof logging**: write-once storage or cryptographic log chaining
- **Alerting**: Critical security events forwarded to SOC / CERT-Bund within defined SLAs

### 8.2 Patch Management

- **Automated vulnerability scanning** (OpenVAS/Greenbone) on a scheduled basis
- **Patch pipeline**: Dev/Test environment mirrors production; patches tested before rollout
- **Emergency patches**: Defined process for critical CVEs (< 24h for CVSS >= 9.0)
- **OpenStack upgrades**: Follow stable release cycle; skip-level upgrades avoided

### 8.3 Backup and Disaster Recovery

- **Backup targets**: Physically separate fire compartment within same Sicherheitsbereich, or second site within Germany connected via BSI-approved VPN
- **RTO/RPO**: Defined per workload classification (VS-NfD systems typically RPO <= 4h, RTO <= 24h)
- **Backup encryption**: AES-256 with keys in HSM
- **Regular restore tests**: At least quarterly, documented

---

## 9. Procurement Considerations

### 9.1 BSI-Approved Products to Procure

| Product Category | Example Products | Approximate Lead Time |
|---|---|---|
| VPN Gateway (VS-NfD) | secunet SINA Box, genua genuscreen | 3-6 months |
| SINA Workstation | secunet SINA Workstation | 3-6 months |
| HSM | Utimaco CryptoServer Se, Thales Luna | 2-4 months |
| Smartcard Reader + Cards | Reiner SCT cyberJack, Bundesdruckerei eID | 1-3 months |
| (Optional) BSI-approved FW | genua genugate, secunet | 3-6 months |

### 9.2 Support Contracts

For the FLOSS components, consider commercial support:
- **Red Hat** or **SUSE** for OS support
- **Canonical** (Ubuntu Pro) as alternative
- **Red Hat OpenStack Platform** or **SUSE OpenStack Cloud** for integrated OpenStack support — alternatively, pure upstream with in-house expertise
- **Ceph**: Red Hat Ceph Storage or SUSE Enterprise Storage for commercial backing
- **Open Telekom Cloud / Dataport** may offer consulting for BSI-compliant OpenStack deployments within Germany

---

## 10. Risk Assessment Summary

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| BSI rejects FLOSS component for specific layer | Medium | High | Early engagement with BSI; document risk analysis per BSI-Standard 200-3; have BSI-approved alternatives identified |
| Supply chain compromise in hardware | Low-Medium | Critical | Procure from EU-based vendors; verify hardware integrity (TPM, Secure Boot); consider hardware from BSI-listed vendors |
| OpenStack vulnerability (zero-day) | Low | High | Network segmentation limits blast radius; BSI-approved perimeter; rapid patching process |
| Staff with required clearances unavailable | Medium | High | Start SU1 clearance process early (takes 3-6 months); cross-train team members |
| C5 attestation delay | Medium | Medium | Begin documentation and operational evidence collection early; hire experienced auditor |
| Vendor lock-in from BSI-approved products | Medium | Medium | Limit proprietary components to network boundary; keep core stack on open standards |

---

## 11. Recommendations

1. **Engage BSI early**: Schedule a consultation with BSI (Referat responsible for cloud security and VS-NfD IT) during the planning phase. Their guidance on acceptable FLOSS usage for your specific Schutzbedarf is authoritative and saves costly rework.

2. **Start with IT-Grundschutz Sicherheitskonzept**: This is the foundational document. Without an approved Sicherheitskonzept, no VS-NfD processing is permitted. Use the BSI GSTOOL or verinice (FLOSS) for documentation.

3. **Prioritize BSI-approved product procurement**: Long lead times for SINA and genuscreen products can delay the project. Order early.

4. **Build a reference environment first**: Deploy a non-classified test cloud to validate the architecture, then promote it to VS-NfD after all controls are in place and verified.

5. **Hire or contract BSI IT-Grundschutz expertise**: An experienced IT-Grundschutz-Berater (preferably BSI-certified) significantly accelerates the Sicherheitskonzept creation and avoids common pitfalls.

6. **Plan for personnel clearances**: SU1 security clearances (Sicherheitsueberpruefung Stufe 1) for all operations staff must be initiated early — the process involves the Bundesamt fuer Verfassungsschutz and can take 3-6 months.

7. **Use verinice for ISMS tooling**: verinice is FLOSS, BSI-recognized, and integrates the IT-Grundschutz-Kompendium directly. It supports C5 control mapping.

8. **Consider joining the IT-Planungsrat / govdigital cooperative**: For shared knowledge and potentially shared procurement of BSI-approved products at government-negotiated rates.

---

## 12. Conclusion

An OpenStack-based private cloud for VS-NfD processing is architecturally sound and achievable. The vast majority of the stack (approximately 80%) can be built with FLOSS components — OpenStack, KVM, Ceph, OVN, Linux — providing sovereignty, auditability, and cost efficiency. The mandatory BSI-approved components are limited to network boundary encryption (VPN gateways), hardware security modules, and authentication tokens. This represents the industry-consensus approach for German government cloud deployments and is consistent with reference architectures from organizations such as Dataport, ITZBund, and BWI.

The path to C5 attestation and IT-Grundschutz certification is well-defined but requires disciplined execution over 16-20 months. Early engagement with BSI, timely procurement of approved products, and rigorous documentation are the critical success factors.
