# Private Cloud Architecture for HEMMELIG-Classified Processing
## Norwegian Defense Contract -- Forsvaret

**Classification**: This document describes architecture for a system intended to process HEMMELIG (Secret) classified data.
**Regulatory Framework**: Sikkerhetsloven (Norwegian Security Act, 2018) and associated regulations.
**Accreditation Authority**: NSM (Nasjonal Sikkerhetsmyndighet).
**Date**: 2026-03-21

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Regulatory and Compliance Framework](#regulatory-and-compliance-framework)
3. [Classification and Security Context](#classification-and-security-context)
4. [Personnel and Clearance Constraints](#personnel-and-clearance-constraints)
5. [Platform Decision: Kubernetes (RKE2) with KubeVirt](#platform-decision)
6. [Physical Architecture](#physical-architecture)
7. [Network Architecture](#network-architecture)
8. [Compute and Storage Architecture](#compute-and-storage-architecture)
9. [Air-Gapped Operations](#air-gapped-operations)
10. [NATO Interoperability](#nato-interoperability)
11. [Security Architecture](#security-architecture)
12. [Monitoring and Observability](#monitoring-and-observability)
13. [Disaster Recovery](#disaster-recovery)
14. [Automation and Infrastructure as Code](#automation-and-infrastructure-as-code)
15. [Accreditation Approach](#accreditation-approach)
16. [Risk Register](#risk-register)
17. [Architectural Decision Records](#architectural-decision-records)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform capable of processing HEMMELIG (Secret) classified data for Forsvaret. The platform is fully air-gapped, physically located in Norway, and designed for accreditation by NSM under Sikkerhetsloven and its regulations (Virksomhetsikkerhetsforskriften, Klareringsforskriften, Sikkerhetsgraderte anskaffelser).

Key design drivers:

- **Classification level**: HEMMELIG -- requires physical air gap, TEMPEST considerations, NSM-approved cryptography, and security-graded areas (sikkerhetsgraderte omrader).
- **Air-gapped**: No internet connectivity. No bridging to lower-classification networks without explicit cross-domain solutions.
- **Data sovereignty**: All data remains in Norway. No cloud provider involvement for classified processing.
- **NATO interoperability**: A subset of data must be shareable with NATO partners at NATO SECRET (NS) level, requiring NATO-approved crypto and compliance with STANAG 4774/4778 for metadata marking.
- **Team constraint**: 20 people total, only 8 with HEMMELIG klarering. This is the single most binding operational constraint.

The recommended platform is **Kubernetes (RKE2)** with **KubeVirt** for VM workloads, backed by **Ceph** for storage, running on hardened bare-metal servers in a spine-leaf network fabric.

---

## 2. Regulatory and Compliance Framework

### Primary Framework (Authoritative)

All design decisions must trace to these Norwegian authorities and standards:

| Framework | Applicability |
|---|---|
| **Sikkerhetsloven (2018)** | Overarching security act governing protection of classified information |
| **Virksomhetsikkerhetsforskriften** | Organizational security requirements, physical security for graded areas |
| **Klareringsforskriften** | Personnel clearance requirements and process |
| **Sikkerhetsgraderte anskaffelser** | Procurement of security-graded systems and components |
| **NSM Grunnprinsipper for IKT-sikkerhet** | Primary ICT security baseline (Identifisere, Beskytte, Oppdage, Handtere) |
| **NSM crypto product list** | Only NSM-approved cryptographic products may be used |
| **NSM technical guidance** | Specific advisories and hardening requirements from NSM |

### International Context

| Framework | Applicability |
|---|---|
| **NATO C-M(2002)49** | NATO security policy -- governs NATO-classified data handling |
| **AC/35 series** | NATO CIS security requirements |
| **STANAG 4774 / 4778** | Confidentiality metadata labeling and binding for NATO interoperability |
| **NIS2 Directive** | Applies via EEA membership -- critical infrastructure obligations |
| **GDPR** | Applies via EEA membership -- personal data protection |
| **ENISA guidance** | Informative for security posture |

### Supplementary Standards (Informative, Not Primary)

- **CIS Benchmarks**: Used for technical hardening of Linux, Kubernetes, and database components only where NSM does not provide specific technical guidance. CIS recommendations must be validated against NSM requirements -- NSM takes precedence in all cases of conflict.
- **NIST 800-190** (Container Security): Referenced for container-specific hardening patterns.
- **NIST 800-207** (Zero Trust): Referenced as a conceptual model.
- **DISA STIGs**: Not authoritative in this context. Referenced only where NSM guidance explicitly defers to or aligns with STIG content.

---

## 3. Classification and Security Context

### Norwegian Classification Levels

| Level | Norwegian | Infrastructure Implication |
|---|---|---|
| Unclassified | UGRADERT | Standard IT controls |
| Restricted | BEGRENSET | Encrypted storage/transport, access control |
| Confidential | KONFIDENSIELT | Dedicated infrastructure, accredited systems |
| **Secret** | **HEMMELIG** | **Air-gapped, TEMPEST, accredited facility, cleared personnel only** |
| Top Secret | STRENGT HEMMELIG | Bespoke isolated infrastructure, highest physical security |

This platform operates at **HEMMELIG**. This mandates:

- Physical air gap -- no network connectivity to UGRADERT, BEGRENSET, or KONFIDENSIELT networks
- Security-graded areas (sikkerhetsgraderte omrader) per Virksomhetsikkerhetsforskriften
- All personnel with system access must hold HEMMELIG klarering
- NSM-approved cryptography for all encryption (at rest and in transit within the classified domain)
- TEMPEST considerations (NATO SDIP-27)
- Comprehensive audit trails with long retention
- Supply chain integrity -- tamper-evident packaging, firmware verification, chain-of-custody

### NATO Classification Mapping

For data shared with NATO partners:

| NATO Level | Abbreviation | Norwegian Equivalent | Infrastructure Implication |
|---|---|---|---|
| NATO UNCLASSIFIED | NU | UGRADERT | Standard IT controls |
| NATO RESTRICTED | NR | BEGRENSET | Encrypted storage and transport, access control |
| NATO CONFIDENTIAL | NC | KONFIDENSIELT | Dedicated infrastructure, accredited systems |
| **NATO SECRET** | **NS** | **HEMMELIG** | **Air-gapped or heavily isolated, TEMPEST, accredited facility** |
| COSMIC TOP SECRET | CTS | STRENGT HEMMELIG | Bespoke isolated infrastructure, highest physical security |

Norwegian HEMMELIG maps to NATO SECRET (NS). Data marked HEMMELIG that is shared with NATO must be re-marked as NATO SECRET and protected with NATO-approved cryptographic products (national NSM-approved crypto is not sufficient for NATO-classified data).

---

## 4. Personnel and Clearance Constraints

### Current Team Structure

| Category | Count | Notes |
|---|---|---|
| Total team | 20 | |
| HEMMELIG klarering holders | 8 | Only these individuals can access the classified platform |
| Without required clearance | 12 | Can work on UGRADERT/BEGRENSET systems only |

### Operational Implications

This is the most binding constraint on the architecture. With only 8 cleared personnel:

**On-call rotation**: A sustainable 24/7 on-call rotation requires a minimum of 4-5 people (to avoid burnout). With 8 cleared staff covering all roles (platform engineering, security operations, application support), 24/7 on-call is extremely tight. Recommendation:
- Implement a primary/secondary on-call rotation with 4 people per rotation, cycling weekly.
- Invest heavily in automation to reduce manual intervention. Every alert that pages a human must be justified.
- Design the platform for self-healing -- automated failover, automated remediation via Event-Driven Ansible.

**Role coverage**: The 8 cleared personnel must cover:
- Platform engineering (Kubernetes, Ceph, bare-metal)
- Security operations (monitoring, incident response, audit)
- Application deployment support
- Physical security liaison
- Accreditation and compliance

**Recommendation**: Prioritize additional klarering applications immediately. The Sikkerhetssamtale (security interview) process takes time. Target 12-14 cleared personnel for sustainable operations. Coordinate with NSM or the delegated clearance authority.

**Uncleared staff utilization**: The 12 uncleared team members can work on:
- Development and testing on UGRADERT environments (code that will later be transferred to the classified platform)
- Unclassified documentation and architecture planning
- Supply chain preparation (building offline package bundles, container images -- on UGRADERT systems)
- Training and skills development
- Supporting BEGRENSET systems if they hold appropriate lower-level clearance

**Vendor and contractor access**: Any vendor or contractor requiring access to the classified platform must hold HEMMELIG klarering. This rules out most commercial support contracts. Design for self-sufficiency.

---

## 5. Platform Decision: Kubernetes (RKE2) with KubeVirt

### ADR-001: Platform Selection

**Decision**: RKE2 (Rancher Government) Kubernetes distribution with KubeVirt for VM workloads.

**Alternatives considered**:

| Platform | Pros | Cons | Verdict |
|---|---|---|---|
| **OpenStack (Kolla-Ansible)** | Mature IaaS, strong multi-tenancy, VM-native | High operational complexity, requires larger team (min 4-6 dedicated), many services to maintain (Keystone, Nova, Neutron, Cinder, Glance, Heat, Octavia...), steep learning curve | Rejected -- team too small |
| **OpenStack (Sunbeam/MicroStack)** | Simpler OpenStack deployment | Less mature, Canonical-dependent, still complex for 8 people | Rejected |
| **Proxmox VE** | Simple, FLOSS, good VM management | No native container orchestration, limited API-driven self-service, limited GitOps story | Rejected -- insufficient for cloud-native workloads |
| **RKE2 + KubeVirt** | Government-focused K8s distro, FIPS crypto modules, CIS-hardened by default, converged VM+container platform, GitOps-native, strong CNCF ecosystem, manageable by small team | KubeVirt less mature than traditional hypervisors for complex VM networking | **Selected** |
| **Talos Linux + K8s** | Immutable OS, API-only (no SSH), minimal attack surface | Less flexibility for debugging, smaller community | Strong alternative -- consider for future iterations |

**Rationale**:
1. **Team size**: RKE2 is operationally simpler than OpenStack. A single converged platform (containers + VMs via KubeVirt) reduces the number of distinct systems the 8 cleared personnel must operate.
2. **Security posture**: RKE2 ships CIS-hardened by default, includes FIPS-validated crypto modules, and is purpose-built for government/defense use cases.
3. **GitOps-native**: Declarative configuration via ArgoCD means the platform state is version-controlled, auditable, and reproducible -- critical for accreditation and for enabling the uncleared team members to prepare configurations on UGRADERT systems.
4. **KubeVirt convergence**: Any legacy VM workloads (Windows, legacy Linux applications) run alongside containers on the same platform, reducing operational surface area.
5. **CNCF ecosystem**: Cilium (networking + security), Rook-Ceph (storage), Falco/Tetragon (runtime security), cert-manager, Kyverno -- all production-grade FLOSS components that avoid vendor lock-in.

---

## 6. Physical Architecture

### Data Center Requirements

The platform must be housed in a facility that meets Virksomhetsikkerhetsforskriften requirements for HEMMELIG processing:

- **Security-graded area (Sikkerhetsgradert omrade)**: The server room and all operational areas must be graded to HEMMELIG level.
- **Physical access control**: Multi-factor access (badge + biometric or PIN), mantrap entry, 24/7 guard or monitoring, visitor escort policy.
- **Intrusion detection**: Alarm systems, CCTV (within classified areas per regulations), motion detection.
- **TEMPEST**: Equipment and facility must meet TEMPEST requirements per NATO SDIP-27. For HEMMELIG, Zone B or Zone A TEMPEST rating is likely required (NSM will specify during accreditation). This affects:
  - Equipment selection (TEMPEST-rated servers, switches, KVM)
  - Room shielding (Faraday cage if Zone A)
  - Cable routing (red/black separation -- classified and unclassified cabling must never share conduits)
  - Minimum separation distances from non-classified equipment and external walls
- **Location**: Within Norway. Consider co-location at an existing Forsvaret facility or a LIST-X equivalent Norwegian accredited facility.

### Rack Layout

Minimum deployment for a production HEMMELIG platform:

```
Rack 1: Management + Control Plane
  - 3x Control plane nodes (RKE2 server nodes)
  - 2x Management/bastion servers
  - 1x Hardware security module (HSM) -- NSM-approved
  - 1x Out-of-band management switch (IPMI/iDRAC/iLO -- isolated network)
  - 1x Top-of-rack switch (management network)

Rack 2-3: Compute + Storage (Hyperconverged)
  - 6-9x Worker nodes (RKE2 agent nodes) per rack
    - Each node: dual-socket, 512GB+ RAM, NVMe for Ceph OSD, 10/25GbE NICs
  - 2x Top-of-rack switches per rack (redundant, spine-leaf)

Rack 4: Network + Security
  - 2x Spine switches
  - 1x NSM-approved crypto device (for NATO interop zone, if applicable)
  - 1x Data diode or cross-domain solution (for controlled data transfer)
  - 1x Logging/SIEM server (dedicated, tamper-evident storage)
  - UPS and PDU infrastructure
```

Total: ~18-24 bare-metal servers for initial deployment. Scale by adding worker nodes.

### Hardware Selection

- **Servers**: HPE ProLiant or Dell PowerEdge with TEMPEST rating as required by NSM. Consider Cisco UCS if existing Cisco investment. Must support: Redfish API for out-of-band management, TPM 2.0 for measured boot, UEFI Secure Boot.
- **Switches**: TEMPEST-rated if required. Spine-leaf topology with 25GbE to servers, 100GbE spine uplinks.
- **HSM**: NSM-approved hardware security module for key management and crypto operations.
- **Storage**: NVMe SSDs in each worker node for Ceph OSD (hyperconverged). Minimum 3-way replication. Separate NVMe for OS and Ceph journal/WAL.

### Supply Chain

Per Sikkerhetsgraderte anskaffelser:
- All hardware procured through NSM-approved supply chains.
- Tamper-evident packaging with chain-of-custody documentation from manufacturer to data center.
- Firmware verification against manufacturer-published checksums before deployment.
- BIOS/UEFI configuration hardened per NSM guidance before first boot.
- Consider Norwegian/allied-nation suppliers (Kongsberg ecosystem, Thales Norway) where possible.

---

## 7. Network Architecture

### Network Segmentation

The air-gapped platform requires strict internal segmentation even though it has no external connectivity:

```
+-----------------------------------------------------------+
|  HEMMELIG Air-Gapped Environment                          |
|                                                           |
|  +------------------+  +------------------+               |
|  | Management VLAN  |  | Out-of-Band Mgmt |              |
|  | (Bastion, K8s    |  | (IPMI/Redfish)   |              |
|  |  API, ArgoCD)    |  | Isolated physical |              |
|  +--------+---------+  +------------------+               |
|           |                                               |
|  +--------+---------+                                     |
|  | Control Plane    |                                     |
|  | VLAN (etcd,      |                                     |
|  |  K8s API server) |                                     |
|  +--------+---------+                                     |
|           |                                               |
|  +--------+---------+  +------------------+               |
|  | Workload VLAN    |  | Storage VLAN     |               |
|  | (Pod traffic,    |  | (Ceph cluster    |               |
|  |  KubeVirt VMs)   |  |  replication)    |               |
|  +------------------+  +------------------+               |
|                                                           |
|  +------------------+                                     |
|  | NATO Interop     |                                     |
|  | Zone (isolated   |<--- NSM-approved crypto device      |
|  |  namespace/VLAN) |<--- Data diode / CDS to NATO net   |
|  +------------------+                                     |
+-----------------------------------------------------------+
       |  (Air gap -- physical separation)
       |  Data diode / CDS for controlled transfer
       v
+-----------------------------------------------------------+
|  BEGRENSET / UGRADERT Environment                         |
|  (Code preparation, package staging, unclass work)        |
+-----------------------------------------------------------+
```

### Network Design

- **Spine-leaf topology**: 2 spine switches, 2 leaf (ToR) switches per rack. All links redundant.
- **Cilium CNI**: Selected as the Kubernetes network plugin.
  - eBPF-based -- high performance, kernel-level packet processing.
  - Network policies: **default-deny** across all namespaces with explicit allow rules.
  - Transparent encryption (WireGuard or IPsec) for pod-to-pod traffic within the cluster -- using NSM-approved crypto parameters.
  - Hubble for network flow observability.
  - Replaces kube-proxy (fully eBPF datapath).
- **Multus**: For KubeVirt VMs requiring multiple network interfaces or SR-IOV passthrough.
- **MetalLB**: For LoadBalancer-type services (L2 or BGP mode depending on fabric design).
- **CoreDNS**: Internal cluster DNS. No external DNS resolution (air-gapped).
- **Out-of-band management**: Physically separate network for IPMI/iDRAC/iLO. Never routable to workload or management VLANs.

### Red/Black Separation

All cabling must enforce red/black separation:
- **Red (classified)**: All network cables carrying HEMMELIG data.
- **Black (unclassified)**: Out-of-band management (if not carrying classified data), and any cables connecting to the cross-domain solution's unclassified side.
- Red and black cables must never share conduits, trays, or equipment.
- Color-coded cabling and labeling per NSM physical security guidance.

---

## 8. Compute and Storage Architecture

### Kubernetes Cluster Design

```
RKE2 Cluster: "forsvaret-hemmelig-prod"

Control Plane (3 nodes -- HA):
  - RKE2 server mode
  - etcd (embedded, encrypted at rest with NSM-approved crypto)
  - Kubernetes API server
  - Dedicated hardware -- not shared with workloads
  - Resource: 16 vCPU, 64GB RAM, 500GB NVMe (etcd)

Worker Nodes (12-18 nodes):
  - RKE2 agent mode
  - Container workloads + KubeVirt VMs
  - Ceph OSD (hyperconverged -- storage co-located with compute)
  - Resource per node: 2x CPU (64+ cores), 512GB+ RAM,
    2x 3.84TB NVMe (Ceph OSD), 1x 960GB NVMe (OS + Ceph WAL/DB)
  - NUMA-aware scheduling, hugepages for KubeVirt VMs
  - CPU pinning for latency-sensitive workloads
```

### Storage Architecture (Rook-Ceph)

- **Rook operator** manages Ceph lifecycle within Kubernetes.
- **Ceph cluster**: Deployed on worker nodes (hyperconverged) to avoid dedicated storage nodes (reduces hardware footprint for small team).
- **Storage classes**:
  - `ceph-block` (RBD): Default for PersistentVolumeClaims, KubeVirt VM disks.
  - `ceph-filesystem` (CephFS): Shared filesystems where needed.
  - `ceph-object` (RGW): S3-compatible object storage for application data.
- **Replication**: 3-way replication minimum for HEMMELIG data. No erasure coding for critical data (replication is simpler and faster for recovery).
- **Encryption at rest**: All OSDs encrypted with dm-crypt/LUKS using keys from the HSM. NSM-approved algorithms.
- **Performance**: NVMe-only Ceph cluster. Dedicated NVMe for WAL/DB. Tune for throughput and IOPS per workload requirements.

### KubeVirt for VM Workloads

- Legacy applications or Windows workloads run as VMs managed by KubeVirt.
- VM disk images imported via CDI (Containerized Data Importer) -- all images transferred via air-gap procedures.
- Live migration supported for maintenance windows.
- SR-IOV for VMs requiring direct NIC access (e.g., network appliances).
- GPU passthrough if ML/analytics workloads require it.

---

## 9. Air-Gapped Operations

The platform has zero internet connectivity. All software, updates, and data must be transferred through controlled physical media with chain-of-custody.

### Software Supply Chain

```
UGRADERT Environment                    HEMMELIG Environment
(Internet-connected)                    (Air-gapped)

+-------------------+                   +-------------------+
| Package mirror    |   Physical media  | Local package     |
| (Pulp / Aptly)    |   (approved USB / | mirror (Pulp)     |
|                   |   optical media)  |                   |
| Container registry|   with chain-of-  | Harbor registry   |
| (Harbor - staging)|   custody, malware| (production)      |
|                   |   scan, approval  |                   |
| Helm chart repo   |   workflow, GPG   | Helm chart repo   |
| Git repos (code)  |   sig verification| Git repos (Gitea) |
+-------------------+                   +-------------------+
      |                                        ^
      | Prepared by uncleared staff            | Imported by cleared staff
      | (12 team members)                      | (8 team members)
      v                                        |
+-------------------+                   +-------------------+
| Transfer staging  |   Data diode or   | Import staging    |
| (scan, sign,      | ================> | (verify sigs,     |
|  package, approve)|   sneakernet      |  scan, admit)     |
+-------------------+                   +-------------------+
```

### Transfer Procedures

1. **Package preparation** (UGRADERT, uncleared staff can do this):
   - Mirror required RPM/DEB packages, container images, Helm charts.
   - Generate SBOMs (Syft) for all container images.
   - Scan for vulnerabilities (Trivy, Grype).
   - Sign all artifacts with GPG keys.
   - Bundle into transfer media (encrypted, checksummed).

2. **Transfer approval** (cleared staff):
   - Review manifest of all artifacts being transferred.
   - Verify GPG signatures and checksums.
   - Malware scan on dedicated scanning station.
   - Formal approval per transfer procedure (logged, attributed to individual).

3. **Import to classified environment** (cleared staff):
   - Physical media inserted into dedicated import station within the HEMMELIG area.
   - Re-verify signatures and checksums.
   - Push to Harbor registry and Pulp package mirror.
   - Kyverno admission controller blocks any image not present in the approved Harbor registry.

4. **Data export (downgrade)** (requires formal release review):
   - Any data leaving the HEMMELIG environment must undergo sanitization review.
   - Formal release procedure per Sikkerhetsloven.
   - Approved by security officer.
   - Data diode preferred for one-way flows (e.g., logs to lower-classification SIEM if approved).

### Offline Tooling

| Tool | Purpose | Air-gap Strategy |
|---|---|---|
| **Harbor** | Container registry | Offline mirror sync via `skopeo copy` on transfer media |
| **Pulp** | RPM/DEB package repository | Export/import functionality with signed repos |
| **Gitea** | Git hosting (code, IaC, GitOps) | Bundle repos, transfer via media |
| **Hauler** | Kubernetes air-gap artifact bundling | Purpose-built for collecting and transferring K8s artifacts |
| **ArgoCD** | GitOps deployment | Points to local Gitea repos |
| **Helm** | Chart management | Charts stored in Harbor OCI registry |

---

## 10. NATO Interoperability

### Requirements

Some data processed on the platform must be shared with NATO partners. This requires:

1. **Data marking**: All shareable data must be marked per STANAG 4774 (confidentiality metadata labeling) and STANAG 4778 (metadata binding).
2. **NATO-approved crypto**: Data shared at NATO SECRET (NS) level must be encrypted with NATO-approved cryptographic products. NSM-approved national crypto is **not** acceptable for NATO-classified data.
3. **Interoperability zone**: A dedicated zone within (or adjacent to) the platform for preparing and staging NATO-bound data.
4. **Security agreements**: Bilateral/multilateral security agreements must be in place with receiving NATO nations.

### NATO Interoperability Architecture

```
+--------------------------------------+
| HEMMELIG Platform                    |
|                                      |
| +------------------+                 |
| | Main workload    |                 |
| | namespaces       |                 |
| +--------+---------+                 |
|          | (Cilium network policy:   |
|          |  strict egress control)   |
| +--------v---------+                 |
| | NATO Interop     |                 |
| | Namespace        |                 |
| | - Data sanitizer |                 |
| | - STANAG 4774    |                 |
| |   labeler        |                 |
| | - Format         |                 |
| |   converter      |                 |
| | - Approval gate  |                 |
| +--------+---------+                 |
+----------|---------------------------+
           |
   +-------v-------+
   | NATO-approved  |
   | crypto device  |
   | (hardware)     |
   +-------+-------+
           |
   +-------v-------+
   | NATO SECRET    |
   | network link   |
   | (to NCIA /     |
   |  NATO CIS)     |
   +---------------+
```

### Implementation Details

- **Dedicated Kubernetes namespace** (`nato-interop`) with strict Cilium network policies:
  - Ingress only from approved source namespaces.
  - Egress only to the NATO crypto device interface.
  - No lateral movement to other namespaces.
- **Data pipeline**: Application submits data for NATO sharing -> sanitization service strips national-only content -> STANAG 4774/4778 labeler applies NATO metadata -> format conversion to NATO standard -> approval gate (human-in-the-loop or automated per policy) -> encrypted via NATO crypto device -> transmitted.
- **NATO crypto device**: Hardware appliance, NSM and NCIA approved for NATO SECRET. This is a physical device in the network path, not software encryption.
- **Accreditation**: The NATO interop zone will require separate accreditation by both NSM (national) and NCIA (NATO). Plan for this as a distinct accreditation workstream.

---

## 11. Security Architecture

### Mapping to NSM Grunnprinsipper for IKT-sikkerhet

Every security control maps to the four categories of NSM's framework:

#### Identifisere (Identify)

| Principle | Implementation |
|---|---|
| Asset inventory | NetBox for DCIM/IPAM -- all hardware, IPs, VLANs documented. Kubernetes API as source of truth for workloads. |
| Risk assessment | Formal risk assessment per Sikkerhetsloven, updated annually or on significant change. |
| Threat intelligence | NorCERT (NSM) feeds consumed (via approved transfer to classified environment). |
| Security governance | Dedicated security officer among the 8 cleared staff. Formal security plan (sikkerhetsplan). |

#### Beskytte (Protect)

| Principle | Implementation |
|---|---|
| Access control | RBAC in Kubernetes, FreeIPA or Keycloak for identity, MFA on all admin access, need-to-know enforced via namespace-level RBAC. |
| Network segmentation | Cilium default-deny policies, VLAN segmentation, physical separation for management/OOB. |
| Secure configuration | RKE2 CIS-hardened by default. Kyverno policies enforce pod security standards. OpenSCAP scans against NSM baselines. |
| Patch management | Monthly patch cycle via air-gap transfer. Emergency patches via expedited transfer procedure. |
| Data security | Encryption at rest (LUKS/dm-crypt with HSM-managed keys), encryption in transit (Cilium WireGuard/IPsec, mTLS via service mesh), NSM-approved crypto. |
| Protective technology | Kyverno admission controller (block unsigned images, enforce policies), Falco and Tetragon for runtime enforcement. |

#### Oppdage (Detect)

| Principle | Implementation |
|---|---|
| Continuous monitoring | Prometheus + Grafana for infrastructure and application metrics. Alertmanager for alerting. |
| Security event logging | All K8s audit logs, OS auth logs, Ceph logs, network flow logs (Hubble) centralized to SIEM. |
| Anomaly detection | Falco for runtime anomaly detection (unexpected syscalls, container escapes, privilege escalation). Tetragon for kernel-level enforcement. |
| Intrusion detection | Cilium network flow analysis. Suricata on mirrored traffic if required by NSM. |

#### Handtere (Respond)

| Principle | Implementation |
|---|---|
| Incident response plan | Documented IR plan, tested quarterly. NorCERT notification procedures. |
| Automated response | Event-Driven Ansible for automated remediation (isolate compromised node, restart failed services). Kyverno and Cilium can auto-block non-compliant workloads. |
| Forensics capability | Immutable audit logs on dedicated SIEM server with tamper-evident storage. Node-level forensic imaging capability. |
| Post-incident improvement | Blameless post-mortems documented as ADRs. |

### Zero Trust

- No implicit trust between any components, even within the air-gapped network.
- All service-to-service communication authenticated via mTLS (Cilium service mesh or Istio ambient mesh).
- Kubernetes API access via short-lived certificates, not long-lived tokens.
- All human access via bastion with MFA, session recording, and time-limited credentials.

### Secrets Management

- **HashiCorp Vault** (deployed within the cluster) for dynamic secrets, PKI, and encryption as a service.
- Vault auto-unseal via the HSM (NSM-approved).
- External Secrets Operator to sync Vault secrets into Kubernetes Secrets.
- No secrets in Git -- Sealed Secrets or External Secrets for GitOps compatibility.

### Container Supply Chain Security

- **Harbor** registry with Trivy scanning -- all images scanned on import.
- **Cosign/Sigstore** signing -- all images must be signed before deployment.
- **Kyverno** admission policies:
  - Block images not from the local Harbor registry.
  - Block unsigned images.
  - Block images with critical/high CVEs.
  - Enforce pod security standards (restricted profile).
  - Block privileged containers (except explicit exemptions for Ceph, Cilium).
- **SBOM**: Every image must have an SBOM (Syft-generated) attached and stored in Harbor.

---

## 12. Monitoring and Observability

### Stack

| Component | Tool | Purpose |
|---|---|---|
| Metrics | **Prometheus** (+ Thanos for long-term) | Infrastructure and application metrics |
| Dashboards | **Grafana** | Visualization, alerting dashboards |
| Alerting | **Alertmanager** | Alert routing and deduplication |
| Logs | **Loki** | Log aggregation (lightweight, fits small team) |
| Network flows | **Hubble** (Cilium) | Network observability, flow logs, policy visualization |
| Tracing | **Jaeger** or **Tempo** | Distributed tracing (if applications support OpenTelemetry) |
| SIEM | **Wazuh** | Security event correlation, compliance reporting |
| Hardware | **Redfish/IPMI exporters** | Hardware health, temperature, disk status |
| Kubernetes | **kube-state-metrics, node-exporter** | Cluster state and node metrics |
| Ceph | **Ceph Prometheus exporter** | Storage cluster health, performance, capacity |

### Audit Logging

- Kubernetes audit logging enabled at `RequestResponse` level for sensitive resources, `Metadata` for others.
- All audit logs shipped to Wazuh SIEM on dedicated logging server.
- Log retention: Per NSM requirements (typically 1-5 years for classified systems -- confirm with NSM during accreditation).
- Tamper-evident log storage: WORM (Write Once Read Many) storage or cryptographic chaining for audit logs.
- Every action attributable to an individual (no shared accounts).

---

## 13. Disaster Recovery

### RTO/RPO Targets

| Tier | RTO | RPO | Examples |
|---|---|---|---|
| Tier 1 (Critical) | 4 hours | 1 hour | Core platform services (K8s control plane, Ceph) |
| Tier 2 (Important) | 8 hours | 4 hours | Application workloads, databases |
| Tier 3 (Standard) | 24 hours | 24 hours | Development/test namespaces, non-critical services |

### Backup Strategy

- **Velero** for Kubernetes resource and PV backup.
- **Ceph RBD snapshots** for block storage.
- **Ceph pool mirroring** to a secondary site (if a DR site exists at HEMMELIG level).
- Backups encrypted with HSM-managed keys.
- Backup media stored in a separate security-graded area (physical separation from primary).
- Regular restore testing (monthly) -- documented and signed off.

### Multi-Site Resilience

If budget and facilities allow, deploy a secondary HEMMELIG site:
- Ceph RBD mirroring for asynchronous storage replication.
- RKE2 cluster at DR site with Cluster API or Rancher for management.
- Dark fiber between sites (encrypted with NSM-approved crypto).
- Failover runbooks automated where possible, tested quarterly.

If single-site only: ensure local redundancy (N+2 for control plane, N+1 for compute, 3-way Ceph replication) and comprehensive backup to offline media.

---

## 14. Automation and Infrastructure as Code

### GitOps Workflow

```
Developer (UGRADERT)        Air-gap Transfer        Classified Platform (HEMMELIG)

Code/config changes  --->   Transfer media   --->   Gitea (local Git)
in Git repos                (approved,signed)             |
                                                          v
                                                    ArgoCD watches repos
                                                          |
                                                          v
                                                    Auto-sync to cluster
                                                    (with approval gates
                                                     for production)
```

### Tooling

| Tool | Purpose |
|---|---|
| **ArgoCD** | GitOps continuous deployment -- watches Gitea repos, syncs to cluster |
| **Ansible** (AWX) | Day-2 operations, bare-metal provisioning, OS configuration, patching |
| **OpenTofu** | Infrastructure provisioning (VMs via KubeVirt, Ceph pools, network config) |
| **Kustomize** | Kubernetes manifest customization (used by ArgoCD) |
| **Helm** | Package management for complex applications |
| **Packer** | Golden image building (on UGRADERT, transferred to classified) |

### Bare-Metal Provisioning

- **MAAS** (Metal as a Service) or **Foreman** for initial bare-metal provisioning.
- PXE boot from local server (no internet).
- Ansible for post-provision hardening and RKE2 installation.
- Immutable base OS (consider Talos Linux in future iterations for even stronger security posture).

---

## 15. Accreditation Approach

### Accreditation-Driven Design

Per the principle "classification drives architecture" -- this platform is designed for HEMMELIG accreditation from inception, not retrofitted.

### NSM Accreditation Lifecycle

| Phase | Activities | Timeline Estimate |
|---|---|---|
| 1. Categorize | Confirm HEMMELIG classification, define data types, identify system boundary | Month 1-2 |
| 2. Select controls | Map NSM Grunnprinsipper to architecture, identify gaps, document control selections | Month 2-4 |
| 3. Implement | Build platform with controls in place, automate compliance scanning | Month 4-10 |
| 4. Assess | NSM security audit, penetration testing, TEMPEST inspection | Month 10-13 |
| 5. Authorize | NSM grants approval to process HEMMELIG data | Month 13-14 |
| 6. Monitor | Continuous compliance, periodic reassessment, change management triggers reaccreditation | Ongoing |

### Continuous Compliance Automation

- **OpenSCAP** with NSM-aligned profiles for OS hardening verification.
- **kube-bench** for Kubernetes CIS benchmark scanning (supplementary to NSM requirements).
- **Kyverno policy reports** for Kubernetes policy compliance dashboards.
- **Wazuh** compliance module for continuous control monitoring.
- **Machine-readable security documentation**: Maintain security architecture in structured format (consider OSCAL if NSM accepts it, otherwise NSM's prescribed format).

### Separate Accreditation Workstreams

1. **Main platform**: HEMMELIG accreditation by NSM.
2. **NATO interop zone**: Dual accreditation by NSM (national) and NCIA (NATO) for NATO SECRET.
3. **Cross-domain solution**: Separate accreditation for the data diode/CDS connecting to lower-classification environments.

---

## 16. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Only 8 cleared personnel -- burnout, single points of failure | High | Critical | Immediately sponsor additional klarering applications. Cross-train all 8 on all platform components. Automate aggressively. |
| R2 | NSM accreditation delays | Medium | High | Engage NSM early (pre-design consultation). Use accreditation-driven design. Maintain continuous dialogue. |
| R3 | Air-gap transfer bottleneck slows patching | Medium | High | Establish weekly transfer cadence. Pre-stage quarterly patch bundles. Expedited procedure for critical vulnerabilities. |
| R4 | TEMPEST compliance adds cost and delays | Medium | Medium | Engage TEMPEST specialists early. Budget for TEMPEST-rated equipment. Consider Zone B to reduce cost vs. Zone A if NSM approves. |
| R5 | KubeVirt immaturity for complex VM networking | Low | Medium | Evaluate SR-IOV and Multus early. Maintain Proxmox as fallback for complex VM workloads if KubeVirt proves insufficient. |
| R6 | NATO crypto device procurement delays | Medium | High | Begin procurement under Sikkerhetsgraderte anskaffelser immediately. Long lead times for NATO-approved crypto. |
| R7 | Vendor lock-in | Low | Medium | FLOSS-first strategy (RKE2, Ceph, Cilium, ArgoCD) minimizes lock-in. All components replaceable. |
| R8 | Supply chain compromise | Low | Critical | Tamper-evident packaging, firmware verification, SBOM analysis, trusted supplier relationships (Kongsberg, Thales Norway). |

---

## 17. Architectural Decision Records

### ADR-001: Platform Selection
- **Decision**: RKE2 + KubeVirt over OpenStack or Proxmox.
- **Rationale**: See Section 5.
- **Status**: Accepted.

### ADR-002: Storage Backend
- **Decision**: Rook-Ceph hyperconverged on worker nodes.
- **Rationale**: Avoids dedicated storage hardware (reduces footprint and cost). Ceph provides block, file, and object storage from a single system. Proven at scale. FLOSS.
- **Status**: Accepted.

### ADR-003: CNI Selection
- **Decision**: Cilium.
- **Rationale**: eBPF-based performance, integrated network policies (default-deny), WireGuard encryption, Hubble observability, service mesh capability. Reduces need for separate tools.
- **Status**: Accepted.

### ADR-004: GitOps over Imperative Deployment
- **Decision**: ArgoCD + Gitea for all deployments.
- **Rationale**: Declarative, auditable, reproducible. Enables uncleared staff to prepare configurations on UGRADERT. Aligns with continuous compliance requirements.
- **Status**: Accepted.

### ADR-005: FLOSS-First Strategy
- **Decision**: Use FLOSS for all software components except where NSM mandates specific commercial products (crypto, HSM).
- **Rationale**: Avoids vendor lock-in, enables code audit, reduces licensing costs, aligns with Norwegian government digital sovereignty goals. Commercial support available from SUSE/Rancher (RKE2), Red Hat (Ceph), Isovalent (Cilium) if needed.
- **Status**: Accepted.

### ADR-006: Hyperconverged over Disaggregated
- **Decision**: Compute and storage co-located on worker nodes (hyperconverged).
- **Rationale**: Reduces hardware footprint and operational complexity for a small team. Ceph's distributed architecture handles node failures gracefully. Dedicated storage nodes would require additional hardware and rack space.
- **Status**: Accepted.

### ADR-007: FreeIPA for Identity Management
- **Decision**: FreeIPA for internal identity, Kerberos, and certificate authority.
- **Rationale**: FLOSS, integrates with Linux/Kubernetes RBAC, provides internal CA for mTLS certificates, avoids Active Directory dependency. If Forsvaret mandates AD integration, FreeIPA can establish a trust relationship.
- **Status**: Proposed -- confirm with Forsvaret identity architecture team.

---

## Summary of Key Recommendations

1. **Start klarering applications immediately** for additional team members. 8 cleared staff is the critical bottleneck.
2. **Engage NSM early** -- pre-accreditation consultation to validate architecture decisions before build.
3. **Procure NATO crypto hardware now** -- long lead times under Sikkerhetsgraderte anskaffelser.
4. **Invest in automation** -- with 8 people, every manual process is a risk. GitOps, Event-Driven Ansible, and self-healing infrastructure are not luxuries; they are necessities.
5. **TEMPEST assessment early** -- the TEMPEST zone classification (A, B, or C) materially affects hardware selection, facility design, and budget. Get NSM guidance on this before procurement.
6. **Separate the NATO interop workstream** -- it has its own accreditation path (NSM + NCIA) and its own crypto requirements. Plan and resource it independently.
7. **Leverage uncleared staff** -- the 12 uncleared team members can do significant preparatory work on UGRADERT systems (code development, package preparation, documentation, testing). Design workflows that maximize their contribution.
