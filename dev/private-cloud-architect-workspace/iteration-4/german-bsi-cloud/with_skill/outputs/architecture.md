# Private Cloud Architecture for VS-NfD Processing: German Bundesbehoerde

## Document Classification

This document provides an architecture recommendation for a German government agency (Bundesbehoerde) building a private cloud to process VS-NfD (Verschlusssache -- Nur fuer den Dienstgebrauch) data, compliant with BSI IT-Grundschutz and targeting BSI C5 attestation.

---

## 1. Executive Summary

This architecture describes an OpenStack-based private cloud designed to process VS-NfD classified data within Germany, fully compliant with BSI IT-Grundschutz and eligible for C5 attestation. The design maximizes FLOSS adoption while identifying the specific layers where BSI-approved products are mandatory. Data sovereignty is guaranteed through exclusive on-premises hosting within German territory, operated by security-cleared German personnel.

Key decisions:

- **OpenStack** as the IaaS platform (FLOSS, no licensing cost, full API-driven self-service)
- **Ceph** for unified storage (block, object, file)
- **BSI-approved VPN and encryption appliances** at the network boundary (non-negotiable for VS-NfD)
- **BSI IT-Grundschutz Bausteine** as the primary hardening baseline (not DISA STIGs, not CIS Benchmarks)
- **C5 attestation** achieved through systematic IT-Grundschutz implementation plus C5-specific cloud controls
- **All data remains within Germany**, all operations performed by cleared German personnel

---

## 2. Regulatory and Compliance Context

### 2.1 German Classification Levels

| Level | Description | Infrastructure Impact |
|---|---|---|
| OFFEN | Open / unclassified | Standard IT controls |
| **VS-NfD** | **Restricted -- for official use only** | **Encrypted storage and transport, access control, BSI-approved crypto, accredited systems** |
| VS-VERTRAULICH | Confidential | Dedicated infrastructure, stricter physical security |
| GEHEIM | Secret | Air-gapped, TEMPEST, high physical security |
| STRENG GEHEIM | Top Secret | Bespoke isolated infrastructure, highest controls |

**VS-NfD is the lowest classification level** but still requires specific BSI-approved products and controls that go beyond what standard commercial security provides.

### 2.2 Primary Frameworks

1. **BSI IT-Grundschutz** -- the primary security framework. Extremely thorough and prescriptive, organized into Bausteine (building blocks) covering every aspect of IT security. This is the authoritative hardening and security baseline.
2. **BSI C5 (Cloud Computing Compliance Criteria Catalogue)** -- the standard for cloud service attestation in Germany. C5:2020 defines criteria across 17 areas. Achieving C5 attestation requires an independent audit (Wirtschaftspruefer) against these criteria.
3. **BSI TR (Technische Richtlinien) series** -- technical guidelines for specific topics (cryptography, key management, etc.).
4. **VSA (Verschlusssachenanweisung)** -- the regulation governing handling of classified information, defining who may access VS-NfD and how.

### 2.3 International Context

As a German Bundesbehoerde within the EU:

- **NIS2 Directive**: If the agency falls under essential or important entity categories, NIS2 obligations apply (implemented into German law via NIS2UmsuCG). Security incident reporting, supply chain security, and management accountability requirements must be addressed.
- **GDPR / DSGVO**: All personal data processing within the cloud must comply with GDPR. Data staying within Germany satisfies data residency, but GDPR's security requirements (Art. 32) are independently mandatory.
- **EUCS (EU Cybersecurity Certification Scheme for Cloud Services)**: Emerging EU-level cloud certification. While C5 is the current German standard, EUCS will become relevant; designing for C5 provides strong alignment.
- **NATO**: If the agency handles NATO RESTRICTED data, additional NATO-specific controls and NATO-approved crypto would be required (separate from this architecture; VS-NfD and NATO RESTRICTED have similar intent but distinct requirements).

---

## 3. Architecture Overview

### 3.1 High-Level Design

```
                                  +---------------------------+
                                  |   BSI-Approved VPN        |
                                  |   (genuscreen / SINA)     |
                                  +---------------------------+
                                             |
                                  +---------------------------+
                                  |   Perimeter Firewall      |
                                  |   (BSI-approved or        |
                                  |    evaluated appliance)   |
                                  +---------------------------+
                                             |
                              +--------------+--------------+
                              |              |              |
                    +---------+--+  +--------+---+  +-------+--------+
                    |  DMZ Zone  |  | Mgmt Zone  |  | VS-NfD Zone    |
                    | (reverse   |  | (Ansible,  |  | (OpenStack     |
                    |  proxy,    |  |  monitoring,|  |  control plane |
                    |  bastion)  |  |  logging)  |  |  + compute)    |
                    +------------+  +------------+  +----------------+
                                                           |
                                                    +------+------+
                                                    |  Ceph       |
                                                    |  Storage    |
                                                    |  Cluster    |
                                                    +-------------+
```

### 3.2 Network Zones

| Zone | Purpose | Security Level |
|---|---|---|
| **External / WAN** | Connectivity to Regierungsnetz or agency WAN | BSI-approved VPN termination |
| **DMZ** | Reverse proxy, bastion hosts, data transfer gateway | Hardened, minimal services |
| **Management** | Ansible/AWX, monitoring (Prometheus/Grafana), logging (Loki/Wazuh), IPMI/Redfish | Restricted to ops personnel |
| **VS-NfD Processing** | OpenStack control plane, compute nodes, tenant workloads | Full VS-NfD controls |
| **Storage** | Ceph cluster network (separate from public network) | Isolated backend network |

All inter-zone traffic passes through firewalls with explicit allow rules. Default-deny everywhere.

---

## 4. FLOSS vs. BSI-Approved Products: Where You Can and Cannot Use FLOSS

This is the central question. The answer is: **FLOSS can be used for most of the stack, but BSI-approved products are mandatory at specific layers.**

### 4.1 Layers Where BSI-Approved Products Are MANDATORY for VS-NfD

| Layer | Requirement | Examples of BSI-Approved Products |
|---|---|---|
| **VPN / Network Encryption** | VS-NfD data in transit over untrusted networks must use BSI-approved VPN appliances | genuscreen (genua), SINA (secunet), NCP Secure VPN GovNet |
| **Disk/Volume Encryption** | Full-disk encryption with BSI-approved or evaluated solutions | BSI-evaluated crypto modules; check BSI product lists for approved FDE solutions |
| **Smartcard / Token Authentication** | Multifactor authentication for VS-NfD access typically requires BSI-evaluated smartcards | Smartcards from approved manufacturers (e.g., Bundesdruckerei), with evaluated card readers |
| **Perimeter Firewall** | While FLOSS firewalls (OPNsense) are technically capable, the accreditation authority will likely require a BSI-evaluated or certified firewall at the VS-NfD boundary | genugate (genua), secunet SINA, or other BSI-evaluated appliances |

**Key principle**: BSI maintains lists of approved products for VS-NfD. For any component that directly enforces a classification boundary (crypto, network boundary, authentication tokens), you must use a product from the BSI-approved list. The BSI "Liste der zugelassenen IT-Sicherheitsprodukte und -systeme" is the authoritative reference.

### 4.2 Layers Where FLOSS Is Fully Appropriate

| Layer | FLOSS Component | Notes |
|---|---|---|
| **Hypervisor / IaaS** | OpenStack (Keystone, Nova, Neutron, Cinder, Glance, Heat, Octavia, Horizon) | No BSI requirement for a specific hypervisor at VS-NfD level. KVM/QEMU under OpenStack is suitable. |
| **Operating System** | Linux (RHEL, SUSE, or Debian/Ubuntu with hardening) | Use a distribution with long-term support. RHEL and SUSE are commonly used in German government. Harden per BSI IT-Grundschutz SYS.1.3 (Linux Server). |
| **Storage** | Ceph (RBD, RGW, CephFS) | Encryption at rest via Ceph's built-in encryption or dm-crypt/LUKS on OSDs (ensure crypto meets BSI TR-02102 requirements). |
| **Monitoring** | Prometheus, Grafana, Alertmanager, Loki | No classification-specific requirements on monitoring tooling itself. |
| **SIEM / Security Monitoring** | Wazuh, OpenSearch | Must meet IT-Grundschutz logging requirements (DER.1 Detektion). |
| **Automation** | Ansible (with AWX), OpenTofu | Infrastructure as Code for reproducibility and audit trails. |
| **Identity (Internal)** | Keycloak, FreeIPA | For internal identity management within the cloud platform. Note: if integrating with existing government identity infrastructure (e.g., PKI, smartcard-based auth), the integration points must use approved mechanisms. |
| **Container Platform** | Kubernetes (RKE2 or kubeadm-based), Harbor, Trivy | If running containers on the cloud. Harden per BSI guidance. |
| **Backup** | Bareos, restic, Velero | Backup encryption must meet BSI TR-02102 cryptographic requirements. |
| **DNS / DHCP** | CoreDNS, PowerDNS, Kea | Standard infrastructure services. |
| **Load Balancing** | HAProxy, Octavia (OpenStack LBaaS) | Internal load balancing is fine with FLOSS. |
| **IPAM / DCIM** | NetBox | Asset management and IP planning. |
| **Network Monitoring** | LibreNMS, Zabbix | Infrastructure monitoring. |

### 4.3 Gray Areas: FLOSS Possible but Requires Careful Evaluation

| Layer | Consideration |
|---|---|
| **TLS / Internal Encryption** | OpenSSL/GnuTLS are fine for internal TLS, but the cryptographic algorithms and key lengths must comply with BSI TR-02102 (Kryptographische Verfahren: Empfehlungen und Schluessellaengen). This is about configuration, not product choice. |
| **Host-based Firewall** | nftables/iptables on Linux hosts is fine, but the perimeter enforcement must be BSI-approved. Defense in depth: use both. |
| **Intrusion Detection** | Suricata/Snort are technically strong. The accreditation body may accept them if properly configured and monitored. Document the rationale. |

### 4.4 Summary Decision Matrix

```
+------------------------------------------------------------------+
|  MUST be BSI-approved        |  CAN be FLOSS                     |
|------------------------------|-----------------------------------|
|  VPN appliances              |  OpenStack (all services)         |
|  Perimeter firewall          |  Linux operating system           |
|  Smartcard/token auth        |  Ceph storage                     |
|  Disk encryption (evaluate)  |  Prometheus/Grafana monitoring    |
|                              |  Ansible/AWX automation           |
|                              |  Keycloak/FreeIPA identity        |
|                              |  Wazuh SIEM                       |
|                              |  HAProxy load balancing           |
|                              |  Kubernetes (if needed)           |
|                              |  Bareos/restic backup             |
|                              |  NetBox DCIM/IPAM                 |
+------------------------------------------------------------------+
```

---

## 5. OpenStack Architecture

### 5.1 Deployment Approach

- **Deployment tooling**: Kolla-Ansible (containerized OpenStack services, reproducible deployments, straightforward upgrades)
- **Control plane**: 3-node HA cluster (Galera for MariaDB, RabbitMQ cluster, HAProxy + keepalived for API endpoints)
- **Compute nodes**: Scale based on workload; NUMA-aware placement for performance-sensitive workloads
- **Network**: OVN (Open Virtual Network) as the Neutron backend -- FLOSS, performant, well-integrated

### 5.2 OpenStack Services

| Service | Purpose | VS-NfD Consideration |
|---|---|---|
| **Keystone** | Identity and authentication | Integrate with smartcard-based auth via SAML/OIDC federation to Keycloak; Keycloak handles smartcard login |
| **Nova** | Compute (KVM/QEMU) | CPU pinning and NUMA for isolation between tenants if multi-tenant |
| **Neutron + OVN** | Networking | Microsegmentation via security groups; network isolation between tenants |
| **Cinder** | Block storage (backed by Ceph RBD) | Encryption at rest via dm-crypt on Ceph OSDs or Cinder volume encryption (with LUKS) |
| **Glance** | Image service (backed by Ceph RBD) | Golden image pipeline; all images hardened per IT-Grundschutz |
| **Heat** | Orchestration | Infrastructure as Code for tenant workloads |
| **Octavia** | Load balancing | Internal LBaaS |
| **Horizon** | Web dashboard | Accessible only from management zone; MFA enforced |
| **Barbican** | Key management | Stores encryption keys; consider integration with HSM for key protection |

### 5.3 Storage Architecture

- **Ceph cluster**: Minimum 3 nodes, each with NVMe SSDs for OSD journals and HDDs or SSDs for data (depending on performance requirements)
- **Storage tiers**: NVMe pool for high-performance workloads, SSD pool for general workloads
- **Encryption at rest**: LUKS on all Ceph OSDs; keys managed via Barbican or dedicated key management
- **Ceph public network** (client-facing) and **Ceph cluster network** (replication) on separate VLANs
- **RBD** for Cinder and Glance, **RGW** for S3-compatible object storage, **CephFS** for shared file systems (Manila)

### 5.4 Network Design

```
+--------------------------------------------------+
|  Spine-Leaf Physical Topology                     |
|                                                   |
|  2x Spine Switches (25/100GbE)                    |
|       |          |          |         |            |
|  Leaf-1      Leaf-2     Leaf-3    Leaf-4           |
|  (Control)   (Compute)  (Storage) (Mgmt/DMZ)      |
+--------------------------------------------------+

Logical Networks (VLANs):
- VLAN 10: Management / IPMI (out-of-band)
- VLAN 20: OpenStack API / Internal
- VLAN 30: Tenant overlay (GENEVE via OVN)
- VLAN 40: Ceph public network
- VLAN 50: Ceph cluster network (replication)
- VLAN 60: DMZ
- VLAN 70: External / WAN facing
```

- **Spine-leaf topology** for predictable latency and horizontal scalability
- **25GbE** for compute and storage nodes (bond two ports for redundancy)
- **100GbE** spine uplinks if storage performance demands it
- **OVN** for software-defined overlay networking (GENEVE tunnels)
- **BSI-approved firewall** between zones (DMZ, management, VS-NfD, external)

---

## 6. Security Architecture

### 6.1 Cryptographic Requirements

All cryptographic implementations must comply with **BSI TR-02102** (current version). Key requirements:

| Use Case | Minimum Requirement (BSI TR-02102) |
|---|---|
| TLS | TLS 1.2 with approved cipher suites, prefer TLS 1.3 |
| VPN | IPsec with BSI-approved algorithms; use BSI-approved VPN appliance |
| Disk encryption | AES-256; key management via HSM or Barbican |
| Hashing | SHA-256 minimum, SHA-384/512 preferred |
| Key exchange | ECDHE with approved curves (brainpoolP256r1 or P-256 minimum) |
| Signatures | ECDSA or RSA >= 3000 bit (per BSI TR-02102 timeline) |

**Important**: BSI TR-02102 specifies that certain algorithms have end-of-life dates. Design the crypto architecture to be algorithm-agile so that cryptographic transitions (e.g., post-quantum readiness) can be performed without full re-architecture.

### 6.2 Identity and Access Management

```
+-------------------+       SAML/OIDC       +-------------------+
|  Smartcard/Token  | --------------------> |    Keycloak       |
|  (BSI-approved)   |                       |    (IdP)          |
+-------------------+                       +-------------------+
                                                    |
                                    +---------------+---------------+
                                    |               |               |
                              +-----+---+    +-----+---+    +------+--+
                              | Keystone |    |  AWX    |    | Wazuh   |
                              | (OS API) |    | (Ansible|    | (SIEM)  |
                              +----------+    +---------+    +---------+
```

- **MFA mandatory** for all administrative and user access to VS-NfD systems
- **Smartcard-based authentication** using BSI-approved smartcards (e.g., from Bundesdruckerei)
- **Keycloak** as the central IdP, federated to OpenStack Keystone via OIDC
- **Role-based access control (RBAC)** with least-privilege principle
- **Need-to-know enforcement**: even with valid clearance, access limited to specific projects/tenants
- **All access logged** and forwarded to SIEM

### 6.3 Network Security

- **Perimeter**: BSI-approved firewall (genugate or equivalent) at the external boundary
- **VPN**: BSI-approved VPN appliance (genuscreen, SINA, or NCP GovNet) for all WAN connectivity
- **Internal segmentation**: OVN security groups + nftables on hosts for defense in depth
- **Management plane isolation**: Management network (VLAN 10/20) never reachable from tenant networks or DMZ
- **Bastion hosts**: All SSH access via hardened bastion hosts with session recording (e.g., using teleport or a simpler approach with auditd + script recordings)
- **No direct internet access**: The VS-NfD zone has no internet connectivity. Software updates via a controlled repository mirror in the DMZ with integrity verification (GPG signatures, checksums).

### 6.4 Physical Security

VS-NfD requires controlled physical access:

- **Data center within Germany** (mandatory for data sovereignty)
- **Access-controlled server room** with logging of all physical access
- **Equipment room** meets BSI IT-Grundschutz INF.2 (Rechenzentrum) requirements
- **No TEMPEST requirement** at VS-NfD level (TEMPEST applies at VS-VERTRAULICH and above)
- **Tamper-evident seals** on server chassis (detect unauthorized hardware access)
- **Supply chain**: Hardware procured through trusted channels with chain-of-custody documentation

### 6.5 Logging, Monitoring, and Audit

IT-Grundschutz requires comprehensive logging (Baustein DER.1):

| Component | Tool | What Is Logged |
|---|---|---|
| Central SIEM | Wazuh + OpenSearch | All security events, correlated |
| OS-level audit | auditd | System calls, file access, privilege escalation |
| OpenStack API | Keystone middleware + Oslo.log | All API calls with user attribution |
| Network flows | OVN flow logs + Suricata | Network traffic patterns, IDS alerts |
| Authentication | Keycloak audit logs | All login attempts, MFA events, federation events |
| Storage | Ceph audit log | Pool/object access, administrative actions |
| Infrastructure | Prometheus + Grafana | Performance metrics, capacity, health |
| Log aggregation | Loki or OpenSearch | Centralized log storage with retention per policy |

**Retention**: VS-NfD audit logs must be retained per VSA and IT-Grundschutz requirements. Plan for minimum 1 year retention, verify with BSI/accreditation authority for exact requirements.

**Integrity**: Log integrity must be ensured. Forward logs to a write-once destination (immutable storage or a dedicated log server with append-only permissions).

---

## 7. BSI IT-Grundschutz Compliance Mapping

### 7.1 Relevant Bausteine (Building Blocks)

IT-Grundschutz is organized into Bausteine. The following are the most relevant for this architecture:

| Baustein | Topic | Architecture Component |
|---|---|---|
| **ISMS.1** | Security management | Organizational: ISMS processes, risk management |
| **ORP.1-5** | Organization and personnel | Personnel clearances, training, separation of duties |
| **INF.2** | Data center | Physical security, power, cooling, access control |
| **NET.1.1** | Network architecture | Zone model, firewall placement, segmentation |
| **NET.1.2** | Network management | VLAN design, switch hardening, network monitoring |
| **NET.3.1** | Router and switches | Spine-leaf switch hardening |
| **NET.3.2** | Firewall | Perimeter and internal firewall configuration |
| **NET.3.3** | VPN | BSI-approved VPN configuration |
| **SYS.1.1** | General server | Base OS hardening (all nodes) |
| **SYS.1.3** | Linux server | Linux-specific hardening (kernel, services, users) |
| **SYS.1.5** | Virtualization | KVM/QEMU hypervisor security |
| **SYS.1.8** | Storage systems | Ceph cluster security |
| **APP.4.4** | Kubernetes | If K8s is deployed on top of OpenStack |
| **OPS.1.1** | General operations | Change management, patch management, incident management |
| **OPS.1.2** | Outsourcing | If any services are externally provided |
| **OPS.2.2** | Cloud usage | C5 criteria mapping |
| **DER.1** | Detection | SIEM, IDS, monitoring requirements |
| **DER.2** | Incident management | Incident response procedures |
| **DER.4** | Contingency planning | Disaster recovery, business continuity |
| **CON.1** | Cryptography | Crypto policy per BSI TR-02102 |
| **CON.3** | Data backup | Backup architecture (Bareos/restic) |

### 7.2 IT-Grundschutz Process

1. **Strukturanalyse (Structure Analysis)**: Document all IT assets, applications, communication links, rooms, and personnel
2. **Schutzbedarfsfeststellung (Protection Needs Assessment)**: Determine protection needs (normal, high, very high) for confidentiality, integrity, availability per asset -- VS-NfD data drives "high" confidentiality minimum
3. **Modellierung (Modeling)**: Map applicable Bausteine to each asset
4. **IT-Grundschutz-Check**: Assess implementation status of all requirements in each applicable Baustein (yes/partially/no)
5. **Risikoanalyse (Risk Analysis)**: For high/very high protection needs, perform additional risk analysis per BSI-Standard 200-3
6. **Implement**: Address gaps identified in the check
7. **Audit**: Independent auditor verifies implementation

---

## 8. C5 Attestation Process

### 8.1 What Is C5?

BSI C5 (Cloud Computing Compliance Criteria Catalogue, version C5:2020) is the German standard for cloud service provider security. It defines criteria across 17 domains:

1. Organisation of information security (OIS)
2. Security policies and work instructions (SP)
3. Personnel (HR)
4. Asset management (AM)
5. Physical security (PS)
6. Operations security (OPS)
7. Identity and access management (IDM)
8. Cryptography and key management (CRY)
9. Communication security (COS)
10. Portability and interoperability (PI)
11. Procurement and development (DEV)
12. Governance, risk, compliance (GRC)
13. Monitoring and logging (LOG)
14. Incident management (SIM)
15. Business continuity (BCM)
16. Compliance (COM)
17. Handling investigation requests (INQ)

### 8.2 C5 Attestation Types

- **Type 1**: Design of controls at a point in time (snapshot)
- **Type 2**: Effectiveness of controls over a period of time (typically 6-12 months) -- this is what you want

### 8.3 Path to C5 Attestation

```
Phase 1: Preparation (3-6 months)
  - Map IT-Grundschutz implementation to C5 criteria
  - Gap analysis: identify where IT-Grundschutz alone does not cover C5
  - Implement missing controls (C5 has cloud-specific criteria not in IT-Grundschutz)
  - Document all controls with evidence

Phase 2: Type 1 Attestation (1-2 months)
  - Engage a Wirtschaftspruefer (auditor, e.g., PwC, KPMG, Deloitte with BSI C5 experience)
  - Auditor assesses control design
  - Receive Type 1 attestation report (SOC 2 style, ISAE 3402 based)

Phase 3: Operational Period (6-12 months)
  - Operate the cloud with all controls active
  - Collect evidence of control effectiveness continuously
  - Automated compliance monitoring feeds evidence collection

Phase 4: Type 2 Attestation (2-3 months)
  - Same auditor assesses control effectiveness over the operational period
  - Receive Type 2 attestation report
  - This is the attestation that carries weight

Phase 5: Continuous (ongoing)
  - Annual re-attestation
  - Continuous monitoring and evidence collection
  - Address any findings from the auditor
```

### 8.4 C5 and IT-Grundschutz Relationship

IT-Grundschutz and C5 overlap significantly but are not identical:

- **IT-Grundschutz** covers the full breadth of IT security for any system
- **C5** adds cloud-specific criteria (multi-tenancy isolation, portability, cloud-specific incident handling, supply chain for cloud)
- Implementing IT-Grundschutz for the cloud infrastructure covers roughly 70-80% of C5 criteria
- The remaining C5 criteria require additional cloud-specific documentation and controls

**Recommendation**: Implement IT-Grundschutz first (it is the more comprehensive framework), then map to C5 and fill gaps. This gives you both IT-Grundschutz certification AND C5 attestation readiness.

---

## 9. Data Sovereignty

### 9.1 Technical Controls

- **All data at rest within Germany**: Ceph cluster physically located in German data centers owned or controlled by the Bundesbehoerde
- **All data in transit encrypted**: BSI-approved VPN for WAN links; TLS 1.3 for internal API communication
- **No cloud bursting to public cloud**: VS-NfD data must never leave the private cloud. No hybrid connectivity for VS-NfD workloads.
- **Software supply chain**: All packages from controlled mirrors within the management zone; no direct internet access from VS-NfD systems
- **Personnel**: All administrators with valid VS-NfD Sicherheitsuebersuepruefung (Ue1 minimum, per SueG)

### 9.2 Organizational Controls

- **German-registered legal entity** operates the infrastructure (the Bundesbehoerde itself or a German contractor with VS-NfD Geheimschutzbetreuung)
- **No foreign jurisdiction access**: No US CLOUD Act exposure, no foreign government access paths
- **Vendor support**: Any vendor remote support requires approved secure access mechanisms with session recording and German-cleared personnel on the vendor side, or no remote access at all (on-site support only)

---

## 10. Automation and Infrastructure as Code

### 10.1 Deployment Pipeline

```
+----------+     +----------+     +----------+     +-----------+
|  Git     | --> |  CI/CD   | --> |  Ansible | --> | OpenStack |
|  (GitLab | --> |  (GitLab | --> |  (AWX)   | --> | + Ceph    |
|   on-prem)|    |   Runner)|    |          |     |           |
+----------+     +----------+     +----------+     +-----------+
```

- **GitLab** (self-hosted, FLOSS Community Edition) for version control and CI/CD
- **Kolla-Ansible** for OpenStack deployment and upgrades
- **Ansible + AWX** for day-2 operations (patching, configuration management, compliance remediation)
- **OpenTofu** for tenant infrastructure provisioning (VMs, networks, volumes)
- **Packer** for golden image pipeline (hardened images per IT-Grundschutz SYS.1.3)

### 10.2 Compliance Automation

- **OpenSCAP** with BSI-aligned SCAP content for automated compliance scanning
- **Ansible roles** for IT-Grundschutz hardening (automate the Bausteine requirements)
- **Wazuh** agents on all nodes for continuous security monitoring and SCA (Security Configuration Assessment)
- **Automated evidence collection** for C5 attestation: scripts that continuously gather control evidence (access logs, configuration states, patch status) and store in an evidence repository

### 10.3 Patch Management

- **Local repository mirrors** (Pulp or Aptly) in the DMZ, synced from upstream with GPG verification
- **Staged rollout**: Dev/Test environment first, then production with change approval
- **Automated patching** via Ansible with pre/post-check playbooks
- **Emergency patching procedure** documented and tested for critical CVEs

---

## 11. Disaster Recovery and Business Continuity

### 11.1 Architecture

| Component | RPO | RTO | Strategy |
|---|---|---|---|
| OpenStack control plane | 1 hour | 4 hours | 3-node HA, Galera replication, automated failover |
| Tenant VMs | 24 hours | 8 hours | Ceph RBD snapshots, Bareos backup to secondary site |
| Ceph storage | 0 (replication) | Automatic | 3x replication within cluster; daily backup to secondary |
| Configuration (IaC) | 0 | 2 hours | Git repository with full IaC; rebuild from code |

### 11.2 Secondary Site

- If a secondary German data center is available: Ceph RBD mirroring for async replication
- Full IaC means the entire stack can be rebuilt at a secondary site from Ansible playbooks and OpenTofu state
- Regular DR drills (minimum quarterly) per IT-Grundschutz DER.4

---

## 12. Recommended Hardware

### 12.1 Bill of Materials (Reference)

| Role | Quantity | Spec | Notes |
|---|---|---|---|
| Control plane | 3 | 2x CPU (16+ cores), 256GB RAM, 2x NVMe 1TB (OS/state), 2x 25GbE | OpenStack services, MariaDB, RabbitMQ |
| Compute | 6+ | 2x CPU (32+ cores), 512GB-1TB RAM, 2x NVMe 480GB (OS), 2x 25GbE | Scale as needed; NUMA-aware |
| Ceph OSD | 5+ | 2x CPU (16+ cores), 128GB RAM, 2x NVMe (WAL/DB), 10-12x SSD/HDD (data), 2x 25GbE | 3x replication; separate public/cluster networks |
| Ceph MON/MGR | 3 | Collocated on control plane or dedicated; 64GB RAM, NVMe | Monitor and manager daemons |
| Network | 2x spine, 4x leaf | 25/100GbE switches | Spine-leaf; FLOSS NOS (SONiC, Cumulus) or vendor (evaluate) |
| Firewall | 2 (HA pair) | BSI-approved appliance | genugate, secunet, or equivalent |
| VPN | 2 (HA pair) | BSI-approved appliance | genuscreen, SINA, NCP GovNet |
| Management | 2 | 2x CPU, 128GB RAM, NVMe | AWX, GitLab, monitoring stack, Wazuh manager |

### 12.2 Hardware Procurement

- **Trusted supply chain**: Procure through established German/EU distributors
- **Firmware verification**: Verify firmware checksums and signatures before deployment
- **BIOS/UEFI hardening**: Disable unnecessary features, set admin passwords, enable Secure Boot
- **Out-of-band management**: IPMI/iLO/iDRAC on dedicated management VLAN, not accessible from tenant networks

---

## 13. Implementation Roadmap

### Phase 1: Foundation (Months 1-3)

- Finalize requirements with BSI / accreditation authority (Geheimschutzstelle)
- Procure hardware and BSI-approved appliances
- Physical data center preparation (INF.2)
- Deploy base OS on all nodes (hardened per SYS.1.3)
- Deploy Ceph storage cluster
- Deploy OpenStack via Kolla-Ansible
- Deploy management stack (AWX, GitLab, monitoring)

### Phase 2: Security Hardening (Months 3-5)

- Implement IT-Grundschutz Bausteine systematically
- Deploy BSI-approved VPN and firewall
- Configure Keycloak with smartcard authentication
- Implement full logging pipeline (Wazuh + Loki)
- OpenSCAP baseline scanning and remediation
- Network segmentation validation
- Encryption at rest and in transit verification

### Phase 3: Compliance and Testing (Months 5-8)

- IT-Grundschutz-Check: self-assessment against all applicable Bausteine
- Gap remediation
- C5 gap analysis (additional cloud-specific controls)
- Penetration testing
- DR drill
- Documentation: security concept (Sicherheitskonzept), operating procedures, incident response plan

### Phase 4: Accreditation (Months 8-12)

- Engage Wirtschaftspruefer for C5 Type 1 attestation
- Submit Sicherheitskonzept to BSI / accreditation authority for VS-NfD approval
- Address any findings
- Begin operational period for C5 Type 2

### Phase 5: Operations and Type 2 (Months 12-18)

- Operate under full controls
- Continuous evidence collection
- C5 Type 2 audit after 6-12 months of operation
- Ongoing IT-Grundschutz maintenance and continuous improvement

---

## 14. Key Recommendations

1. **Engage BSI early**: Contact the BSI and your Geheimschutzstelle before finalizing the architecture. They may have specific requirements or preferences that affect product selection. Early engagement prevents costly rework.

2. **FLOSS is viable for the majority of the stack**: OpenStack, Ceph, Linux, Ansible, Prometheus, Wazuh, Keycloak -- all are suitable for VS-NfD environments. The BSI does not mandate proprietary software for these layers.

3. **BSI-approved products are non-negotiable at the boundary**: VPN appliances, perimeter firewalls, and smartcard/token authentication must be from the BSI-approved list. Do not attempt to substitute FLOSS alternatives here -- it will fail accreditation.

4. **BSI TR-02102 governs your crypto choices**: Even when using FLOSS (OpenSSL, dm-crypt), the algorithms and key lengths must comply with BSI TR-02102. This is a configuration matter, not a product selection matter.

5. **IT-Grundschutz first, C5 second**: IT-Grundschutz is more comprehensive and is the foundation. C5 adds cloud-specific criteria on top. Do not try to do C5 in isolation.

6. **Automate compliance from day one**: Manual compliance evidence collection will not scale and will make C5 Type 2 attestation painful. Invest in OpenSCAP, Wazuh SCA, and automated evidence pipelines early.

7. **Personnel security is as important as technical security**: All personnel with access to VS-NfD systems need Sicherheitsuebersuepruefung (Ue1). Plan for this -- the clearance process takes time.

8. **Document everything**: The Sicherheitskonzept is the central document for accreditation. Use structured documentation (consider OSCAL for machine-readable security documentation) and keep it current with automated updates where possible.

9. **Plan for upgrades**: OpenStack releases every 6 months. Kolla-Ansible supports rolling upgrades. Build upgrade procedures into your operational model from the start and test them in a staging environment.

10. **Consider SUSE or Red Hat for commercial support**: While the OS is FLOSS, having a commercial support contract (SUSE Linux Enterprise Server or Red Hat Enterprise Linux) provides a safety net for the accreditation authority and for operational incidents. German government agencies commonly use SLES.

---

## 15. Architecture Decision Records (ADRs)

### ADR-001: OpenStack as IaaS Platform

- **Status**: Accepted
- **Context**: Need API-driven, self-service IaaS for VS-NfD workloads. Alternatives: Nutanix (proprietary, licensing cost), VMware (proprietary, Broadcom uncertainty), Proxmox (smaller ecosystem for government scale).
- **Decision**: OpenStack with Kolla-Ansible deployment
- **Rationale**: FLOSS, no licensing cost, large ecosystem, proven in government deployments (CERN, European public sector), full API compatibility, Ceph integration is mature. Team can be trained; Canonical, SUSE, and Red Hat offer OpenStack support contracts if needed.

### ADR-002: BSI-Approved VPN and Firewall at Boundary

- **Status**: Accepted
- **Context**: VS-NfD requires approved crypto at classification boundaries.
- **Decision**: genuscreen (VPN) and genugate (firewall) from genua GmbH, or equivalent secunet SINA products.
- **Rationale**: Mandatory per BSI requirements. No FLOSS alternative will pass VS-NfD accreditation at the network boundary.

### ADR-003: Ceph for Unified Storage

- **Status**: Accepted
- **Context**: Need block (Cinder), object (Swift-compatible), and file (Manila) storage.
- **Decision**: Ceph with RBD, RGW, and CephFS
- **Rationale**: Single FLOSS storage platform covers all three use cases. Eliminates need for separate SAN/NAS. Proven at scale. Encryption at rest via LUKS on OSDs meets BSI TR-02102 when configured with AES-256.

### ADR-004: Wazuh for SIEM

- **Status**: Accepted
- **Context**: IT-Grundschutz DER.1 requires detection capabilities.
- **Decision**: Wazuh (FLOSS) with OpenSearch backend
- **Rationale**: FLOSS, comprehensive HIDS + SIEM + SCA capabilities, active community, sufficient for VS-NfD detection requirements. Commercial alternatives (Splunk) offer no compliance advantage at this classification level and add significant licensing cost.

### ADR-005: Keycloak for Identity Federation

- **Status**: Accepted
- **Context**: Need central IdP that supports smartcard authentication and OIDC federation to OpenStack Keystone.
- **Decision**: Keycloak
- **Rationale**: FLOSS, supports X.509 client certificate authentication (smartcard), OIDC/SAML federation, fine-grained authorization. Bridges BSI-approved smartcard auth to OpenStack's Keystone.

---

## 16. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| BSI accreditation authority rejects FLOSS component | Medium | High | Engage BSI early; document security rationale; offer to demonstrate equivalent security |
| Personnel clearance delays | High | Medium | Start Ue1 clearance process immediately for all team members |
| OpenStack upgrade breaks compatibility | Medium | Medium | Staging environment; Kolla-Ansible tested upgrades; rollback procedures |
| Ceph data loss | Low | Critical | 3x replication + daily backup to secondary site; regular scrubbing |
| Supply chain compromise (hardware/software) | Low | Critical | Trusted procurement; firmware verification; GPG-signed packages; SBOM tracking |
| C5 audit findings require major rework | Medium | High | Self-assessment against C5 criteria before engaging auditor; continuous compliance monitoring |

---

## 17. Cost Considerations

### 17.1 FLOSS Advantage

By using OpenStack, Ceph, and FLOSS tooling, the primary costs are:

- **Hardware** (servers, storage, networking): Capital expenditure
- **BSI-approved appliances** (VPN, firewall): Typically 50,000 to 150,000 EUR per HA pair depending on throughput
- **Personnel**: Skilled OpenStack/Linux engineers (invest in training)
- **Commercial Linux support** (SLES or RHEL): ~1,000-3,000 EUR per server per year
- **C5 audit fees**: 50,000-150,000 EUR per attestation cycle (varies by auditor and scope)
- **Optional**: OpenStack support contract from Canonical, SUSE, or Red Hat

### 17.2 Compared to Proprietary

A comparable Nutanix or VMware-based solution would add:

- Nutanix licensing: significant per-node cost
- VMware licensing: per-CPU licensing (post-Broadcom, costs have increased substantially)
- Neither eliminates the need for BSI-approved boundary devices

The FLOSS approach yields substantial TCO savings (estimated 40-60% over 5 years) while meeting all compliance requirements.

---

## 18. Conclusion

A fully FLOSS-based private cloud for VS-NfD is achievable with the critical exception of BSI-approved boundary devices (VPN, firewall, smartcard authentication). OpenStack with Ceph provides a mature, proven IaaS platform that meets all technical requirements. The path to BSI IT-Grundschutz compliance and C5 attestation is well-defined: implement Bausteine systematically, automate compliance, engage BSI early, and plan for continuous attestation.

The architecture prioritizes:

- **Security**: Defense in depth, BSI-approved crypto at boundaries, comprehensive logging
- **Data sovereignty**: All data within Germany, no foreign jurisdiction exposure
- **Operational excellence**: Full IaC, automated compliance, reproducible deployments
- **Cost efficiency**: FLOSS stack minimizes licensing spend; budget allocated to hardware and skilled personnel
- **Compliance**: IT-Grundschutz as primary framework, C5 attestation built on top, NIS2/GDPR addressed

---

*Document prepared as architecture guidance. All specific product selections and accreditation decisions must be validated with the BSI and the agency's Geheimschutzstelle/Informationssicherheitsbeauftragter (ISB) before implementation.*
