# Private Cloud Architecture for HEMMELIG Classified Data Processing

## Norwegian Defense Platform — Architecture Document

**Classification**: This document itself is UGRADERT (Unclassified) — it describes architecture patterns without revealing specific system details.

**Customer**: Norwegian defense company under contract for Forsvaret (Norwegian Armed Forces)

**Data Classification**: HEMMELIG (Secret) per Norwegian classification scheme

**Accreditation Authority**: NSM (Nasjonal Sikkerhetsmyndighet)

**Date**: 2026-03-20

**Version**: 1.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Compliance Framework](#2-regulatory-and-compliance-framework)
3. [Platform Decision: OpenStack vs. Kubernetes](#3-platform-decision-openstack-vs-kubernetes)
4. [Physical Architecture](#4-physical-architecture)
5. [Network Architecture](#5-network-architecture)
6. [Compute Architecture](#6-compute-architecture)
7. [Storage Architecture](#7-storage-architecture)
8. [Identity and Access Management](#8-identity-and-access-management)
9. [Security Architecture](#9-security-architecture)
10. [Air-Gap Strategy and Data Transfer](#10-air-gap-strategy-and-data-transfer)
11. [NATO Interoperability](#11-nato-interoperability)
12. [Personnel and Clearance Constraints](#12-personnel-and-clearance-constraints)
13. [Automation and Infrastructure as Code](#13-automation-and-infrastructure-as-code)
14. [Monitoring and Observability](#14-monitoring-and-observability)
15. [Disaster Recovery and Business Continuity](#15-disaster-recovery-and-business-continuity)
16. [Accreditation Approach](#16-accreditation-approach)
17. [Supply Chain Security](#17-supply-chain-security)
18. [Implementation Roadmap](#18-implementation-roadmap)
19. [Risk Register](#19-risk-register)
20. [Architectural Decision Records](#20-architectural-decision-records)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform that will process HEMMELIG (Secret) classified data under contract for Forsvaret. The platform must be fully air-gapped, physically located in Norway, accredited by NSM under Sikkerhetsloven, and capable of sharing data with NATO partners through controlled mechanisms.

### Key Architectural Decisions

- **Platform**: RKE2-based Kubernetes with KubeVirt for VM workloads, deployed on hardened bare-metal infrastructure. OpenStack is recommended as an alternative only if the workload profile is predominantly VM-based and the team has existing OpenStack competence.
- **Air gap**: Physical air gap — no logical network separation, no internet connectivity, separate physical infrastructure from any lower-classification environment.
- **Data transfer**: Hardware data diodes for ingress; NSM-approved cross-domain solution for controlled bidirectional transfer where required; manual sneakernet with chain-of-custody for media.
- **NATO interoperability**: STANAG-compliant data formats, NATO-approved cryptographic products for data destined for NATO networks, and defined interconnection procedures aligned with NATO security policy C-M(2002)49.
- **Hardening baseline**: NSM Grunnprinsipper for IKT-sikkerhet as the primary framework, supplemented by CIS Benchmarks where NSM does not provide specific technical guidance.
- **Cryptography**: NSM-approved cryptographic products exclusively. No FIPS or other national crypto standards as primary — NSM approval is the only valid baseline for HEMMELIG.

### Constraints

- 20-person team, only 8 hold HEMMELIG klarering
- All infrastructure and data must remain physically within Norway
- Full air gap — no internet, no connectivity to lower-classification networks
- NSM accreditation required before operational use
- NATO data sharing requirement adds interoperability obligations

---

## 2. Regulatory and Compliance Framework

### Primary Framework: Sikkerhetsloven (Norwegian Security Act, 2018)

Sikkerhetsloven is the governing legislation. All design decisions must trace back to its requirements and subordinate regulations:

- **Sikkerhetsloven (Lov om nasjonal sikkerhet)** — establishes the overarching obligations for protecting national security information
- **Virksomhetsikkerhetsforskriften** — enterprise security regulation; defines organizational security requirements, security graded areas, physical access controls
- **Klareringsforskriften** — clearance regulation; governs personnel security clearance process
- **Sikkerhetsgraderte anskaffelser** — regulation on security-graded procurements; governs how classified contracts are managed, supplier security obligations

### NSM Guidance

- **NSM Grunnprinsipper for IKT-sikkerhet** — the primary ICT security baseline. Structured around four categories:
  - **Identifisere** (Identify): asset management, risk assessment, governance
  - **Beskytte** (Protect): access control, awareness training, data security, protective technology
  - **Oppdage** (Detect): continuous monitoring, detection processes, anomaly identification
  - **Respondere** (Respond): response planning, communications, analysis, mitigation, improvements
- **NSM technical guidelines** — specific technical advisories and configuration guidance
- **NSM-approved cryptographic products list** — mandatory reference for all encryption decisions

### International Context

Norway's position creates specific international obligations:

- **NATO member (founding)**: NATO security policy C-M(2002)49 applies to all NATO-classified information. Systems handling data shared with NATO must meet NATO security standards. NCIA accreditation may be required for NATO interconnection.
- **EEA member**: NIS2 Directive obligations apply (transposed into Norwegian law). GDPR applies through EEA agreement. While HEMMELIG data is exempted from GDPR (national security exemption under Article 2(2)), any personal data processed alongside classified data must still be handled with awareness of GDPR principles.
- **Bilateral agreements**: Norway has bilateral security agreements with NATO allies. Data sharing with specific nations may invoke additional bilateral requirements beyond NATO baseline.

### Classification Level Implications for HEMMELIG

HEMMELIG is the fourth level in the Norwegian five-tier classification scheme (UGRADERT, BEGRENSET, KONFIDENSIELT, HEMMELIG, STRENGT HEMMELIG). At this level:

- **Physical separation**: Mandatory physical air gap from lower-classification networks and the internet
- **Personnel**: All personnel with access must hold HEMMELIG klarering (or higher)
- **Physical areas**: Processing must occur in sikkerhetsgradert omrade (security-graded area) approved for HEMMELIG
- **Cryptography**: Only NSM-approved products for any encryption
- **TEMPEST**: Emanation security requirements apply — SDIP-27 Zone B or Zone A depending on facility and threat assessment
- **Audit**: Comprehensive audit logging with long retention, attributable to individual users
- **Access**: Klarering alone is not sufficient — need-to-know must also be established

---

## 3. Platform Decision: OpenStack vs. Kubernetes

### Recommendation: Kubernetes (RKE2) with KubeVirt

After evaluating both approaches against the specific constraints of this project, we recommend a Kubernetes-based platform using RKE2 as the distribution, augmented with KubeVirt for any VM workloads.

### Decision Rationale

| Criterion | OpenStack | Kubernetes (RKE2 + KubeVirt) | Assessment |
|-----------|-----------|------------------------------|------------|
| Operational complexity | High — many services (Keystone, Nova, Neutron, Cinder, Glance, Heat, etc.) each requiring configuration, upgrade coordination | Moderate — single control plane, declarative API, GitOps-native | **Kubernetes wins** — with only 8 cleared operators, minimizing operational overhead is critical |
| Air-gapped deployment | Possible but complex — many Python dependencies, OpenStack release upgrades are involved | Strong tooling — RKE2 has explicit air-gap installation support, Helm chart bundles, Hauler for artifact management | **Kubernetes wins** — better air-gap tooling ecosystem |
| VM support | Native (Nova + KVM) | Via KubeVirt — mature CNCF incubating project, production-grade | **OpenStack wins** for pure VM workloads, but KubeVirt is sufficient for mixed workloads |
| Container-native workloads | Possible via Magnum/Zun but not first-class | Native — this is what Kubernetes does | **Kubernetes wins** — modern defense applications are increasingly containerized |
| Government/defense adoption | Used in some defense contexts but less momentum | RKE2 Government (FIPS-validated), Platform One Big Bang, strong defense sector adoption globally | **Kubernetes wins** — stronger defense ecosystem and reference architectures |
| Team size impact | Requires dedicated OpenStack operators — 8 cleared people may not be enough for both platform ops and application delivery | Smaller operational footprint, more automation possible with fewer people | **Kubernetes wins** — critical given the 8-person cleared team constraint |
| NATO interoperability | No specific NATO tooling | STANAG-compliant workloads deploy as containers; easier to align with NATO partner platforms that are also Kubernetes-based | **Kubernetes wins** — NATO partners are converging on Kubernetes |
| Hardening | Manual — no equivalent of Big Bang or hardened distributions | RKE2 ships hardened by default (CIS compliant out of box), Big Bang provides security stack | **Kubernetes wins** |

### When OpenStack Would Be Preferred

OpenStack should be reconsidered if:
- The workload profile is 90%+ traditional VMs with no containerization path
- The team has deep existing OpenStack competence (Kolla-Ansible, Ceph integration)
- There is a requirement to provide IaaS self-service to multiple tenant organizations within the classified environment

### Hybrid Option

If both VM-heavy and container workloads must coexist at scale, consider OpenStack for IaaS with Kubernetes deployed on top of OpenStack VMs. However, this adds operational complexity that is challenging with 8 cleared operators.

### Selected Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| OS | Talos Linux or RHEL 9 (hardened) | Talos: immutable, API-only, minimal attack surface. RHEL: if NSM or Forsvaret requires a commercially supported OS with Norwegian language support contracts |
| Kubernetes distribution | RKE2 | Hardened by default, CIS compliant, explicit air-gap support, FIPS mode available, strong government adoption |
| VM workloads | KubeVirt | Run legacy VMs alongside containers in same cluster, CNCF incubating, production-grade |
| Container runtime | containerd (bundled with RKE2) | Industry standard, CRI-compliant |
| Networking | Cilium | eBPF-based, high performance, network policy enforcement, transparent encryption, observability (Hubble) |
| Storage | Rook-Ceph | Unified block/file/object storage, mature, well-integrated with Kubernetes |
| Service mesh | Cilium service mesh or Istio | Mutual TLS for all service-to-service communication, traffic observability |
| GitOps | ArgoCD | Declarative deployment, drift detection, audit trail |
| Registry | Harbor | Air-gap mirror support, image signing, vulnerability scanning, RBAC |
| Security | Kyverno + Falco + Trivy | Policy enforcement, runtime detection, vulnerability scanning |
| Secrets | HashiCorp Vault (self-hosted) | Centralized secrets management, audit logging, dynamic secrets |
| Monitoring | Prometheus + Grafana + Loki + Alertmanager | Full observability stack, all FLOSS |

---

## 4. Physical Architecture

### Data Center Requirements

The platform must be hosted in a facility that meets the physical security requirements for HEMMELIG under Virksomhetsikkerhetsforskriften:

#### Security-Graded Area (Sikkerhetsgradert omrade)

- **Sperret omrade (restricted area)** at minimum, with access control and logging for all entry/exit
- All personnel entering must hold appropriate klarering
- Intrusion detection systems on all access points
- 24/7 monitoring of the physical space
- Visitor escort requirements for any uncleared personnel (e.g., facility maintenance)
- The facility must be approved by the organization's sikkerhetsansvarlig (security officer) and may require NSM inspection

#### TEMPEST/Emanation Security

For HEMMELIG processing, TEMPEST requirements per SDIP-27 apply:

- **Zone classification**: Conduct a TEMPEST threat assessment with NSM guidance. HEMMELIG typically requires Zone B (inspectable space of 20 meters) or potentially Zone A depending on the facility's proximity to public areas or foreign entities
- **TEMPEST-rated equipment**: Servers, switches, KVM systems, and cabling must meet the required SDIP-27 zone rating, OR the facility must provide equivalent shielding (e.g., a shielded server room / Faraday cage)
- **Red/black separation**: Classified (red) and unclassified (black) signals must never share conductors, conduits, or equipment. Since this is a fully air-gapped HEMMELIG environment, ALL cabling in the secure area is "red" — but separation from any building infrastructure cabling that exits the secure zone must be maintained
- **Cable routing**: Fiber optic preferred over copper for inter-rack and inter-room links (fiber does not emanate)

#### Rack Layout

Recommended minimum deployment for production:

```
Rack 1: Management / Control Plane
  - 3x management/control plane nodes (RKE2 server nodes)
  - 2x infrastructure services (Harbor registry, Vault, monitoring)
  - 1x out-of-band management switch (dedicated IPMI/iDRAC/iLO network)
  - 1x top-of-rack switch (management network)
  - UPS

Rack 2-3: Compute / Worker Nodes
  - 8-10x worker nodes per rack (RKE2 agent nodes)
  - Each node: dual-socket, 512GB+ RAM, NVMe local storage for Ceph OSDs
  - 2x top-of-rack switches (spine-leaf to Rack 1)
  - UPS

Rack 4: Storage-Heavy Nodes + Network
  - 3-5x storage-dense nodes (high NVMe/SSD count for Ceph)
  - Core/spine switches
  - Data diode hardware
  - Cross-domain solution hardware (if bidirectional transfer required)
  - Patch panels, fiber management
  - UPS
```

This is a minimum viable deployment. Scale racks 2-3 pattern for additional compute capacity.

#### Power and Cooling

- Dual-feed power from independent sources where possible
- UPS per rack with minimum 15-minute runtime
- Generator backup for the secure facility
- N+1 cooling redundancy
- Power Usage Effectiveness (PUE) target: < 1.5
- Monitor power draw per rack to stay within facility capacity

### Hardware Selection Criteria

All hardware must be procured through trusted supply chains per Sikkerhetsgraderte anskaffelser:

- **Server vendor**: Prefer vendors with Norwegian or NATO-nation supply chains. Dell, HPE, or Lenovo (with supply chain verification). Cisco UCS is an option if the organization has existing Cisco competence
- **No Chinese-manufactured network equipment**: Given the classification level, avoid network hardware from vendors flagged by NSM or NATO
- **Firmware verification**: All hardware must have firmware signatures verified before deployment. Establish firmware baselines and monitor for unauthorized changes
- **Tamper-evident shipping**: Hardware must be shipped with tamper-evident seals, and chain-of-custody documentation must be maintained from vendor to rack
- **Lifecycle**: Plan for 5-year hardware lifecycle with spares held on-site (air-gapped environment means no just-in-time delivery for urgent replacements)

---

## 5. Network Architecture

### Design Principles

- **Physical air gap**: No network path exists between this environment and any lower-classification network or the internet. This is enforced physically, not logically.
- **Defense in depth within the air gap**: Even within the HEMMELIG boundary, apply network segmentation to limit blast radius of any compromise.
- **Spine-leaf topology**: Modern, scalable, predictable latency.

### Network Topology

```
                    ┌─────────────────────────────────────────────┐
                    │        HEMMELIG Air-Gapped Boundary         │
                    │                                             │
                    │  ┌──────────┐          ┌──────────┐        │
                    │  │ Spine-1  │──────────│ Spine-2  │        │
                    │  └────┬─────┘          └─────┬────┘        │
                    │       │  ╲              ╱    │             │
                    │       │    ╲          ╱      │             │
                    │       │      ╲      ╱        │             │
                    │       │        ╲  ╱          │             │
                    │       │        ╱  ╲          │             │
                    │       │      ╱      ╲        │             │
                    │  ┌────┴──┐ ╱    ┌─────┴──┐   │             │
                    │  │Leaf-1 │╱     │ Leaf-2 │   │             │
                    │  │(Mgmt) │      │(Compute│   │             │
                    │  └───┬───┘      └───┬────┘   │             │
                    │      │              │        │             │
                    │  ┌───┴───┐     ┌────┴────┐   │             │
                    │  │Rack 1 │     │Rack 2-3 │   │             │
                    │  │Ctrl   │     │Workers  │   │             │
                    │  │Plane  │     │         │   │             │
                    │  └───────┘     └─────────┘   │             │
                    │                              │             │
                    │  ┌───────┐     ┌─────────┐   │             │
                    │  │Leaf-3 │     │Leaf-4   │   │             │
                    │  │(Stor) │     │(OOB/Xfr)│   │             │
                    │  └───┬───┘     └────┬────┘   │             │
                    │      │              │        │             │
                    │  ┌───┴───┐     ┌────┴────┐   │             │
                    │  │Rack 4 │     │Data     │   │             │
                    │  │Storage│     │Transfer │   │             │
                    │  │       │     │Zone     │   │             │
                    │  └───────┘     └─────────┘   │             │
                    │                                             │
                    └─────────────────────────────────────────────┘
                                      │
                              ┌───────┴────────┐
                              │  Data Diode /  │
                              │  Cross-Domain  │
                              │  Solution      │
                              └───────┬────────┘
                                      │
                              ┌───────┴────────┐
                              │  Transfer Zone │
                              │  (Lower Class.)│
                              └────────────────┘
```

### Network Segments

| Segment | Purpose | VLAN/Subnet | Access |
|---------|---------|-------------|--------|
| Management | Control plane, Kubernetes API, IPMI/BMC | Dedicated VLAN, e.g., 10.10.0.0/24 | Control plane nodes only |
| Workload - Data | Pod-to-pod and service communication | Cilium overlay (VXLAN or Geneve) | Worker nodes |
| Storage | Ceph cluster network (replication traffic) | Dedicated VLAN, e.g., 10.20.0.0/24 | Storage nodes + OSD nodes |
| Storage - Public | Ceph client access network | Dedicated VLAN, e.g., 10.21.0.0/24 | All nodes needing storage |
| Out-of-Band | IPMI, iDRAC, iLO, Redfish | Physically separate switch, 10.30.0.0/24 | Admin workstations only |
| Data Transfer | Controlled ingress/egress zone | Isolated VLAN, 10.40.0.0/24 | Data diode, CDS hardware, transfer workstations |

### Cilium Network Policy

Cilium is selected as the CNI for its eBPF-based performance, transparent encryption capabilities, and granular network policy enforcement:

- **Default deny**: All namespaces start with default-deny ingress and egress policies
- **Explicit allow**: Each application team must define CiliumNetworkPolicy resources specifying exact allowed flows
- **Transparent encryption**: WireGuard-based node-to-node encryption for all pod traffic (using NSM-approved configuration — verify WireGuard's crypto primitives against NSM requirements; if ChaCha20-Poly1305 is not NSM-approved, use IPsec with NSM-approved algorithms instead)
- **Hubble**: Network flow observability for audit and troubleshooting — all flow logs shipped to central log aggregation

### DNS

- **CoreDNS**: Deployed as part of RKE2 for cluster-internal DNS
- **No external DNS**: Air-gapped — all DNS is internal
- **Split zones**: Separate DNS zones for management, workload, and storage networks

### Load Balancing

- **kube-vip**: For Kubernetes API server high availability (virtual IP for control plane)
- **MetalLB**: For LoadBalancer-type services within the cluster (L2 or BGP mode depending on switch support)
- **No external load balancers needed**: Air-gapped environment with no external ingress

### Switch Selection

- 25/100GbE spine-leaf fabric
- If the organization has Cisco competence: Cisco Nexus 9000 series with NX-OS or ACI mode
- If FLOSS-preferred: Switches supporting Cumulus Linux, SONiC, or similar open network OS
- Redundant control plane connections to both spine switches
- All inter-switch links are fiber (TEMPEST consideration — no copper between racks)

---

## 6. Compute Architecture

### Node Specifications

#### Control Plane Nodes (3x)

| Component | Specification |
|-----------|--------------|
| CPU | 2x Intel Xeon Gold or AMD EPYC (16+ cores per socket) |
| RAM | 256 GB DDR5 ECC |
| Storage | 2x 1TB NVMe (RAID 1 for OS), 2x 2TB NVMe (etcd) |
| Network | 2x 25GbE (bonded, management), 2x 25GbE (workload) |
| BMC | iDRAC/iLO/CIMC on OOB network |

#### Worker Nodes (16-20x)

| Component | Specification |
|-----------|--------------|
| CPU | 2x Intel Xeon Gold or AMD EPYC (32+ cores per socket) |
| RAM | 512 GB DDR5 ECC |
| Storage | 2x 1TB NVMe (OS, RAID 1), 4x 4TB NVMe (Ceph OSD) |
| Network | 2x 25GbE (workload), 2x 25GbE (storage), 1x 25GbE (management) |
| BMC | iDRAC/iLO/CIMC on OOB network |
| GPU (optional) | NVIDIA A-series if ML/AI workloads required |

#### Storage-Dense Nodes (3-5x, if Ceph is separated)

| Component | Specification |
|-----------|--------------|
| CPU | 2x Intel Xeon Silver or AMD EPYC (16+ cores per socket) |
| RAM | 256 GB DDR5 ECC |
| Storage | 2x 1TB NVMe (OS), 12x 8TB NVMe or SSD (Ceph OSD) |
| Network | 2x 25GbE (storage-public), 2x 25GbE (storage-cluster), 1x 25GbE (management) |

### Operating System

**Primary recommendation: Talos Linux**

- Immutable, API-only operating system purpose-built for Kubernetes
- No SSH, no shell — reduces attack surface to near zero
- All configuration via API (talosctl) — fully auditable
- Automatic updates via machine configuration
- Ideal for high-security environments where minimal OS attack surface is paramount

**Alternative: RHEL 9 (hardened)**

- If Forsvaret or NSM requires a commercially supported, certified OS
- Hardened per NSM Grunnprinsipper for IKT-sikkerhet
- OpenSCAP profiles applied (Norwegian defense profile if available, otherwise CIS RHEL 9)
- SELinux enforcing mode mandatory
- Unnecessary services disabled, minimal package installation
- FIPS mode enabled if required by NSM for the OS-level crypto (verify with NSM — FIPS is a US standard, NSM may have different requirements)

### NUMA and Performance

- NUMA-aware scheduling enabled in Kubernetes (Topology Manager)
- CPU Manager policy: static (for guaranteed QoS pods needing dedicated CPUs)
- Hugepages configured for workloads requiring them (databases, KubeVirt VMs)
- IRQ affinity tuned for network-intensive workloads

---

## 7. Storage Architecture

### Rook-Ceph Deployment

Ceph is deployed via Rook operator within the Kubernetes cluster, providing unified storage:

| Storage Type | Ceph Component | Kubernetes Interface | Use Case |
|-------------|----------------|---------------------|----------|
| Block (RBD) | RADOS Block Device | CSI PersistentVolume | Databases, stateful workloads, KubeVirt VM disks |
| Filesystem (CephFS) | Ceph Filesystem | CSI PersistentVolume (ReadWriteMany) | Shared file access across pods |
| Object (RGW) | RADOS Gateway | S3-compatible API | Artifact storage, backups, unstructured data |

### Ceph Configuration

- **Minimum 3 monitors (MON)**: Deployed on control plane nodes or dedicated infra nodes
- **Minimum 3 managers (MGR)**: Co-located with MONs
- **OSDs**: One per NVMe device on worker or storage-dense nodes
- **Replication factor**: 3 (size=3, min_size=2) — standard for HEMMELIG availability requirements
- **Erasure coding**: Optional for cold/archive data (higher storage efficiency, lower performance)
- **Encryption at rest**: Ceph OSD encryption using dm-crypt with LUKS — keys managed by Vault
- **Separate cluster network**: Ceph replication traffic on dedicated storage-cluster VLAN (keeps replication traffic off the workload network, improving performance and security)

### Storage Classes

```yaml
# High-performance block storage (NVMe-backed)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-block-fast
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  pool: fast-pool
  encrypted: "true"
reclaimPolicy: Retain

# Standard block storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-block-standard
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  pool: standard-pool
  encrypted: "true"
reclaimPolicy: Retain

# Shared filesystem
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-filesystem
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  fsName: cephfs
  encrypted: "true"
reclaimPolicy: Retain
```

### Backup

- **Velero**: Kubernetes-native backup of resources and persistent volumes
- **Ceph RBD snapshots**: Point-in-time snapshots for rapid recovery
- **Backup target**: Separate Ceph pool or dedicated backup nodes within the air-gapped environment
- **Backup schedule**: Daily incremental, weekly full, per RPO requirements (see Section 15)
- **Backup encryption**: All backups encrypted at rest
- **Backup testing**: Monthly restore test of randomly selected workloads

---

## 8. Identity and Access Management

### Authentication Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Keycloak   │────│   FreeIPA    │────│ LDAP/Kerberos│
│  (SSO/OIDC)  │     │ (Identity)   │     │  (Backend)   │
└──────┬───────┘     └──────────────┘     └──────────────┘
       │
       ├── Kubernetes OIDC authentication
       ├── ArgoCD SSO
       ├── Grafana SSO
       ├── Harbor SSO
       ├── Vault SSO
       └── Application SSO
```

### Components

- **FreeIPA**: Central identity store. Manages users, groups, host-based access control, sudo rules, and certificate authority for internal PKI. Preferred over Active Directory for this environment (FLOSS, no Microsoft licensing dependency, well-suited to Linux-only environments).
- **Keycloak**: OIDC/SAML identity broker. Provides SSO for all web-based services. Federates identity from FreeIPA.
- **Kubernetes RBAC**: Integrated with Keycloak OIDC. Users authenticate via Keycloak, Kubernetes maps OIDC groups to ClusterRoles/Roles.
- **Vault**: Application secrets management. Kubernetes auth backend for pod identity, OIDC auth for human operators.

### RBAC Model

Design around the principle of least privilege AND need-to-know:

| Role | Scope | Clearance Required | Personnel |
|------|-------|-------------------|-----------|
| Platform Admin | Full cluster admin | HEMMELIG | 2-3 people |
| Platform Operator | Namespace admin for infra namespaces, read-only cluster | HEMMELIG | 2-3 people |
| Application Developer | Namespace admin for assigned app namespaces only | HEMMELIG | Remaining cleared personnel |
| Security Auditor | Read-only across all namespaces, full access to audit logs | HEMMELIG | 1-2 people (can overlap with other roles) |
| Monitoring Operator | Access to monitoring stack, read-only workload metrics | HEMMELIG | 1-2 people (can overlap) |

### Multi-Factor Authentication

- MFA mandatory for all human access (Keycloak enforces)
- Hardware tokens (YubiKey or similar) preferred over TOTP for HEMMELIG environments
- Certificate-based authentication (from FreeIPA CA) as primary factor for SSH (if RHEL, not applicable for Talos)
- Service accounts use Kubernetes ServiceAccount tokens or Vault-issued short-lived credentials — never long-lived shared secrets

---

## 9. Security Architecture

### Defense-in-Depth Layers

```
Layer 1: Physical Security
  └─ Sikkerhetsgradert omrade, access control, TEMPEST, guards

Layer 2: Network Security
  └─ Air gap, spine-leaf segmentation, Cilium network policies, encrypted overlay

Layer 3: Host Security
  └─ Immutable OS (Talos) or hardened RHEL, SELinux, minimal packages

Layer 4: Container/Runtime Security
  └─ Hardened RKE2, Falco runtime detection, Tetragon eBPF enforcement

Layer 5: Application Security
  └─ Image signing (cosign), vulnerability scanning (Trivy), admission control (Kyverno)

Layer 6: Data Security
  └─ Encryption at rest (LUKS/Ceph), encryption in transit (mTLS/WireGuard), Vault for secrets

Layer 7: Identity & Access
  └─ FreeIPA + Keycloak, RBAC, MFA, least privilege, need-to-know

Layer 8: Audit & Monitoring
  └─ Comprehensive audit logs, Falco alerts, Prometheus metrics, Loki logs
```

### Hardening Baseline: NSM Grunnprinsipper for IKT-sikkerhet

Every control in the platform maps to NSM Grunnprinsipper categories. The following table shows key mappings:

| NSM Category | NSM Principle | Platform Implementation |
|-------------|--------------|------------------------|
| Identifisere | Kartlegg enheter og programvare | NetBox DCIM/IPAM for asset inventory, Kubernetes resource discovery |
| Identifisere | Kartlegg saarbarheter | Trivy vulnerability scanning of all images and nodes |
| Beskytte | Etabler tilgangskontroll | FreeIPA + Keycloak + K8s RBAC + Cilium network policies |
| Beskytte | Beskytt data i transitt | Cilium WireGuard/IPsec, mTLS via service mesh |
| Beskytte | Beskytt data i ro | LUKS on OS disks, Ceph OSD encryption, etcd encryption |
| Beskytte | Sikre IKT-konfigurasjon | GitOps (ArgoCD), immutable OS (Talos), Kyverno policies |
| Oppdage | Etabler sentralisert logging | Loki + Promtail for all logs, shipped to central store |
| Oppdage | Gjennomfor kontinuerlig overvaking | Prometheus + Alertmanager + Falco + Hubble |
| Respondere | Etabler hendelseshandtering | Incident response runbooks, Falco alert routing, defined escalation |

### Admission Control (Kyverno Policies)

Mandatory policies enforced on all workloads:

```yaml
# Example: Require all images to be signed and from Harbor registry
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-image-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "harbor.hemmelig.local/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <platform-signing-key>
              -----END PUBLIC KEY-----
```

Additional mandatory policies:
- No privileged containers (except explicitly approved system workloads)
- No host network/PID/IPC namespace access (except approved)
- Read-only root filesystem enforced
- Resource limits required on all containers
- No latest tag — all images must use digest or semver
- All images must originate from the internal Harbor registry
- Disallow NodePort services
- Require security context with non-root user
- Require network policies in every namespace

### Runtime Security (Falco + Tetragon)

- **Falco**: Detects anomalous system calls and container behavior (file access to sensitive paths, unexpected network connections, shell spawning in containers, privilege escalation attempts)
- **Tetragon**: eBPF-based enforcement — can block (not just detect) unauthorized behavior at the kernel level
- All security events shipped to central logging and alerting pipeline
- Critical alerts trigger immediate incident response

### Encryption Requirements

| Data State | Mechanism | Key Management |
|-----------|-----------|---------------|
| Data at rest (OS) | LUKS2 with dm-crypt | TPM-sealed keys or Vault |
| Data at rest (Ceph) | Ceph OSD encryption (dm-crypt) | Vault KMS integration |
| Data at rest (etcd) | Kubernetes EncryptionConfiguration | KMS provider (Vault) |
| Data in transit (pod-to-pod) | Cilium WireGuard or IPsec | Automatically rotated |
| Data in transit (service-to-service) | mTLS via Istio/Cilium | cert-manager with FreeIPA CA |
| Data in transit (user-to-cluster) | TLS 1.3 | cert-manager certificates |
| Backup data | Encrypted Velero backups | Vault-managed keys |

**Critical**: All cryptographic algorithms and implementations must be verified against NSM's approved list. If a specific algorithm (e.g., ChaCha20-Poly1305 in WireGuard) is not NSM-approved for HEMMELIG, substitute with an NSM-approved alternative (e.g., AES-256-GCM via IPsec).

---

## 10. Air-Gap Strategy and Data Transfer

### Air-Gap Enforcement

The air gap is **physical, not logical**:

- No network cable, wireless link, or any other electromagnetic communication path connects the HEMMELIG environment to any lower-classification network or the internet
- Separate physical switches, separate cabling, separate server hardware
- No shared infrastructure of any kind with lower-classification environments
- WiFi and Bluetooth disabled in BIOS/UEFI on all hardware, physically verified
- USB ports disabled in BIOS/UEFI on worker/storage nodes (enabled only on designated transfer workstations with hardware write-blockers)

### Data Ingress (Into HEMMELIG Environment)

```
External Source (Lower Classification)
        │
        ▼
┌───────────────────┐
│  Staging Area     │  (Lower-classification workstation)
│  - Prepare media  │
│  - Verify content │
│  - Calculate hash │
└───────┬───────────┘
        │ Physical media (USB, encrypted removable drive)
        │ OR data diode (hardware one-way)
        ▼
┌───────────────────┐
│  Data Diode       │  (Hardware-enforced one-way transfer)
│  (e.g., Advenica  │
│   SecuriCDS)      │
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│  HEMMELIG Ingest  │  (Quarantine zone within air-gapped network)
│  - Hash verify    │
│  - Malware scan   │
│  - Content inspect│
│  - Approval gate  │
└───────┬───────────┘
        │ (After approval)
        ▼
┌───────────────────┐
│  Production       │  (Main HEMMELIG cluster)
│  Environment      │
└───────────────────┘
```

**Data diode selection**: Use an NSM-approved or NATO-evaluated data diode product. Vendors to evaluate:
- Advenica SecuriCDS (Swedish, NATO-evaluated, used in Scandinavian defense)
- Owl Cyber Defense (US, widely deployed in NATO)
- Fox-IT DataDiode (Dutch, European supply chain)

Preference for Advenica given Scandinavian proximity and existing Nordic defense adoption.

### Data Egress (Out of HEMMELIG Environment)

Data leaving the HEMMELIG environment is the highest-risk operation and must follow strict procedures:

1. **Sanitization review**: All outbound data must be reviewed by an authorized person to verify it does not contain HEMMELIG information that should not leave the environment (downgrade/release review)
2. **Formal approval**: Documented approval by the information owner and security officer before any data leaves
3. **Transfer mechanism**:
   - **For NATO sharing**: Via NSM-approved cross-domain solution or through established NATO communication channels (see Section 11)
   - **For downgrade to KONFIDENSIELT or lower**: Manual transfer via approved removable media with chain-of-custody logging
4. **Audit trail**: Every data egress event logged with: who approved, what data, when, why, destination, and method

### Software and Patch Ingress

Since the environment is air-gapped, all software updates, container images, OS patches, and configuration changes must be physically transferred in:

1. **On the connected side** (lower-classification network with internet access):
   - Pull updated container images, Helm charts, OS packages
   - Scan all artifacts for vulnerabilities (Trivy, Grype)
   - Verify signatures and checksums (GPG, cosign)
   - Generate SBOMs for all new artifacts (Syft)
   - Bundle everything using Hauler or equivalent air-gap transfer tooling
   - Write to approved transfer media

2. **Transfer through the diode or approved media**

3. **On the HEMMELIG side**:
   - Verify checksums match manifests from the connected side
   - Re-scan all artifacts in the quarantine zone
   - Import into Harbor registry (container images)
   - Import into local package repository (Pulp for RPMs, or Aptly for Debian)
   - Run integration tests in a staging namespace before production rollout

**Patch cadence**: Monthly patch cycle minimum. Critical security patches follow expedited transfer process (still through diode/media, but with accelerated approval).

---

## 11. NATO Interoperability

### Requirements

Some data processed on this platform will be shared with NATO partners. This requires:

1. **NATO security policy compliance**: Systems handling NATO-classified information must comply with C-M(2002)49
2. **NATO classification marking**: Data destined for NATO sharing must carry appropriate NATO classification markings (e.g., NATO SECRET)
3. **Cryptographic requirements**: Data transmitted to NATO partners must be encrypted with NATO-approved cryptographic products
4. **Interoperability standards**: Data formats must align with applicable STANAGs

### Architectural Implications

#### Data Classification Dual-Marking

Data that is both nationally classified (HEMMELIG) and intended for NATO sharing must be dual-marked:

- National marking: HEMMELIG
- NATO marking: NATO SECRET (or as appropriate per the bilateral/multilateral agreement)

The platform must support metadata tagging that carries both classification markings and controls access accordingly.

#### NATO Communication Channel

Data sharing with NATO partners does NOT happen through direct network interconnection with the HEMMELIG platform. Instead:

```
HEMMELIG Platform
       │
       │ (Approved egress procedure - see Section 10)
       ▼
┌──────────────────┐
│ Transfer/Release │  (Data sanitized, approved for NATO release)
│ Zone             │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│ NATO Network     │  (Separate infrastructure, NATO-accredited)
│ Interface        │  (e.g., connection to NATO SECRET WAN)
│                  │  (Using NATO-approved crypto)
└──────────────────┘
```

The NATO network interface is a **separate system** — it is NOT part of the HEMMELIG platform being designed here. The HEMMELIG platform's responsibility is to:
1. Support proper dual-classification marking
2. Implement release/downgrade workflows for NATO-destined data
3. Produce data in STANAG-compliant formats
4. Maintain audit trails for all NATO data releases

#### STANAG Compliance

Relevant STANAGs to consider for data format interoperability:

| STANAG | Purpose |
|--------|---------|
| STANAG 4559 | NATO Standard ISR Library Interface (for intelligence data) |
| STANAG 4609 | NATO Digital Motion Imagery Standard |
| STANAG 4607 | NATO Ground Moving Target Indicator Format |
| STANAG 5516 | Link 16 (tactical data link) |
| STANAG 4774/4778 | Metadata binding and labeling for information sharing |

The specific STANAGs applicable depend on the nature of the data being processed. The platform must be designed to support the relevant format transformations and metadata enrichment as part of the data processing pipeline.

#### Federated Identity for NATO Sharing

If NATO partners need to access shared services (unlikely for HEMMELIG, but possible for specific collaboration scenarios):

- SAML federation via Keycloak
- NATO PKI certificate-based authentication
- Separate namespace/tenant with NATO-specific access controls
- All access logged and auditable

---

## 12. Personnel and Clearance Constraints

### Current State

| Category | Count | Cleared (HEMMELIG) |
|----------|-------|-------------------|
| Total team | 20 | 8 |
| Uncleared | 12 | 0 |

### Impact on Architecture

The 8-person cleared team constraint is one of the most significant drivers of architectural decisions:

1. **Automation is mandatory, not optional**: With only 8 people who can touch the system, every manual operation is expensive. The platform must be automated to the maximum extent possible.

2. **Operational simplicity over feature richness**: This is why Kubernetes (RKE2) is recommended over OpenStack. RKE2 has a smaller operational surface area, and a team of 8 can realistically operate it while also doing application development.

3. **On-call burden**: 8 people for 24/7 on-call is thin. With a typical rotation:
   - 4-person rotation = each person on call every 4th week (sustainable minimum)
   - Leaves 4 people for day-shift development/operations
   - This is tight — consider whether 24/7 is actually required or if extended business hours (e.g., 06:00-22:00) suffice

4. **Bus factor**: With 8 people, losing even one is a 12.5% reduction in capacity. Cross-training is essential. No single person should be the only one who understands any subsystem.

### Role Allocation (Recommended)

| Role | People | Notes |
|------|--------|-------|
| Platform Lead / Architect | 1 | Overall technical decisions, NSM accreditation liaison |
| Platform Operations | 3 | Day-to-day cluster operations, patching, monitoring, on-call |
| Application Development | 3 | Building the classified applications that run on the platform |
| Security / Compliance | 1 | Security monitoring, audit log review, incident response, accreditation documentation |

All 8 should be cross-trained on at least one other role. The Platform Lead should be able to perform Platform Operations tasks. Application Developers should understand the deployment pipeline (GitOps).

### What the 12 Uncleared People Can Do

The uncleared team members cannot access the HEMMELIG environment but can contribute to:

- **Development on unclassified look-alike environments**: Build and test on an UGRADERT development cluster with identical architecture (minus the classified data)
- **Tooling and automation development**: Write Ansible playbooks, Helm charts, Terraform modules, CI/CD pipelines — all tested on the unclassified environment, then transferred via approved media to the classified environment
- **Documentation**: Architecture docs, runbooks, training materials (at UGRADERT level)
- **Vendor management and procurement**: Hardware ordering, contract management
- **Training**: Develop and deliver training for the cleared team

### Clearance Pipeline

Given the constraint, actively work to get more team members cleared:

- Initiate klarering applications for additional team members through the organization's sikkerhetsansvarlig
- The HEMMELIG clearance process (managed by NSM or delegated clearance authority) takes months — start immediately
- Target: At least 12 cleared personnel within 12 months to provide sustainable operational capacity

---

## 13. Automation and Infrastructure as Code

### Automation Stack

Given the personnel constraints, automation is the most critical enabling capability:

```
┌─────────────────────────────────────────────────┐
│                  GitOps Layer                    │
│  ArgoCD manages all Kubernetes resources         │
│  from Git repositories (air-gapped Git server)   │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────┐
│              Infrastructure as Code              │
│  OpenTofu: Infrastructure provisioning           │
│  Ansible: OS configuration, bare-metal setup     │
│  Helm: Kubernetes application packaging          │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────┐
│              Policy as Code                      │
│  Kyverno: Kubernetes admission policies          │
│  OPA/Conftest: Infrastructure policy testing     │
└─────────────────────────────────────────────────┘
```

### Git Server (Air-Gapped)

Since there is no internet access, a self-hosted Git server is required within the HEMMELIG environment:

- **Gitea**: Lightweight, self-hosted Git server (FLOSS, single binary, low resource footprint)
- All infrastructure code, Helm charts, application code, and ArgoCD ApplicationSets stored here
- Code is developed on the unclassified side and transferred via approved media, or developed directly on classified workstations by cleared developers
- Branch protection rules enforce code review by at least one other cleared person before merge

### ArgoCD Configuration

- ArgoCD deployed in a dedicated `argocd` namespace
- SSO via Keycloak (OIDC)
- RBAC: Platform operators can sync all apps; application developers can sync only their namespaces
- App-of-apps pattern for managing the entire platform stack
- Automated sync enabled with self-heal for infrastructure components
- Manual sync required for application deployments (human approval gate)
- All sync events logged for audit

### Ansible for Bare-Metal and OS

Ansible is used for everything below Kubernetes:

- **Bare-metal provisioning**: PXE boot configuration, BIOS/UEFI settings, RAID configuration
- **OS hardening**: NSM Grunnprinsipper-aligned hardening playbooks (if RHEL)
- **Talos machine config**: Generate and apply Talos Linux machine configurations (if Talos)
- **Network switch configuration**: If using Cisco NX-OS, ansible cisco.nxos collection
- **Ceph pre-configuration**: Disk preparation, network configuration before Rook takes over

Ansible is run from a dedicated admin workstation within the HEMMELIG environment. AWX (FLOSS upstream of Ansible Automation Platform) can be deployed in the cluster for scheduled and audited Ansible execution.

### OpenTofu for Infrastructure

OpenTofu manages infrastructure that sits outside Kubernetes but is still within the air-gapped environment:

- FreeIPA configuration
- DNS zone management
- Network infrastructure (if switches support Terraform/OpenTofu providers)
- State stored in an S3-compatible backend (Ceph RGW via MinIO gateway, within the air-gapped environment)

### CI/CD Pipeline (Air-Gapped)

```
Developer workstation (HEMMELIG)
       │
       ▼
Gitea (push code)
       │
       ▼
Tekton Pipeline (triggered by webhook)
  ├── Run unit tests
  ├── Build container image (Kaniko - no Docker daemon)
  ├── Scan image (Trivy)
  ├── Sign image (cosign with internal PKI)
  ├── Push to Harbor
  └── Update Helm chart version in GitOps repo
       │
       ▼
ArgoCD (detects change in GitOps repo)
  ├── Sync to staging namespace
  ├── Run integration tests
  └── (After manual approval) Sync to production namespace
```

---

## 14. Monitoring and Observability

### Stack

| Component | Purpose | Deployment |
|-----------|---------|-----------|
| Prometheus | Metrics collection and alerting | Prometheus Operator (kube-prometheus-stack) |
| Alertmanager | Alert routing and deduplication | Part of kube-prometheus-stack |
| Grafana | Dashboards and visualization | SSO via Keycloak |
| Loki | Log aggregation | Deployed via Helm, backed by Ceph object storage |
| Promtail | Log shipping from nodes | DaemonSet on all nodes |
| Falco | Runtime security events | DaemonSet on all worker nodes |
| Hubble | Network flow observability | Part of Cilium deployment |
| Ceph Dashboard | Storage health and performance | Built into Rook-Ceph |
| NetBox | Asset inventory, IPAM, DCIM | Deployed in cluster |

### Dashboards (Grafana)

Mandatory dashboards:

1. **Cluster Overview**: Node health, pod counts, resource utilization, Kubernetes API server latency
2. **Ceph Storage**: OSD status, pool utilization, IOPS, latency, recovery progress
3. **Network**: Hubble flow metrics, dropped packets, network policy violations, DNS query rates
4. **Security**: Falco alert count by severity, Kyverno policy violation count, failed authentication attempts
5. **Application**: Per-namespace resource usage, pod restart counts, request latency (if instrumented)
6. **Audit**: Authentication events, RBAC authorization failures, API server audit log summary
7. **Hardware**: Node temperatures (via IPMI exporter), disk health (SMART), power draw

### Alerting Rules

Critical alerts (immediate response required):
- Node down
- Ceph OSD down (if reduces below min_size for any pool)
- Ceph health WARN or ERR
- Etcd leader election or quorum loss
- Kubernetes API server unreachable
- Falco critical severity event
- Certificate expiry < 7 days
- Disk utilization > 85%
- Memory pressure or OOM kills

Warning alerts (next business day):
- Pod restart loop (CrashLoopBackOff)
- Disk utilization > 70%
- CPU sustained > 80% for 30 minutes
- Kyverno policy violations (should not happen if enforce mode, but monitor audit mode policies)
- Ceph recovery/rebalancing in progress

### Log Retention

For HEMMELIG environments, NSM and Sikkerhetsloven require comprehensive audit trails:

- **Security-relevant logs**: Minimum 12 months online, 5 years archived (verify specific requirement with NSM)
- **Application logs**: Minimum 6 months online
- **All logs**: Immutable storage (write-once Ceph pool or equivalent)
- **Log integrity**: Cryptographic chaining or signing to detect tampering

---

## 15. Disaster Recovery and Business Continuity

### RPO/RTO Targets

| Tier | Workload Type | RPO | RTO | Strategy |
|------|-------------|-----|-----|----------|
| Tier 1 | Core platform (K8s control plane, etcd) | 1 hour | 4 hours | Etcd snapshots every hour, 3-node HA |
| Tier 2 | Critical applications | 4 hours | 8 hours | Velero scheduled backups, Ceph snapshots |
| Tier 3 | Supporting services (monitoring, CI/CD) | 24 hours | 24 hours | Daily Velero backups, re-deployable from Git |

### Backup Strategy

- **Etcd**: Automated snapshots every hour, stored on separate Ceph pool, retained for 30 days
- **Kubernetes resources**: Velero backup every 4 hours for Tier 2, daily for Tier 3
- **Persistent volumes**: Ceph RBD snapshots aligned with Velero schedule
- **Configuration**: All configuration is in Git (Gitea) — Git is the backup for configuration
- **Gitea**: Database backup (PostgreSQL dump) daily, Git repositories stored on Ceph

### Single-Site Risk

This architecture describes a single-site deployment. A single site is a significant risk for HEMMELIG operations:

**Recommendation**: If budget and facility allow, deploy a second site:
- Geographically separated within Norway (minimum 50 km)
- Connected via dark fiber or NSM-approved encrypted link (dedicated, not shared)
- Ceph RBD mirroring between sites for asynchronous replication
- Kubernetes cluster federation or active-passive failover
- Both sites must meet identical physical security requirements

If a second site is not feasible in the initial deployment, mitigate with:
- Comprehensive on-site spares inventory (spare nodes, switches, drives)
- Documented rebuild procedures tested quarterly
- Off-site backup of critical data on encrypted media stored in NSM-approved secure storage at a separate location

### DR Testing

- Quarterly: Full restore test of a Tier 2 application from Velero backup
- Semi-annually: Etcd restore test (restore cluster from snapshot)
- Annually: Full platform rebuild test (bare-metal to operational)
- All DR tests documented with results and lessons learned

---

## 16. Accreditation Approach

### NSM Accreditation Lifecycle

Following the accreditation process aligned with Sikkerhetsloven:

#### Phase 1: Categorization and Planning

- **Classify the system**: HEMMELIG processing system
- **Define system boundary**: All hardware, software, networks, and facilities described in this architecture document
- **Identify applicable controls**: Map NSM Grunnprinsipper for IKT-sikkerhet measures to the system
- **Develop Security Target (Sikkerhetsmaal)**: Document the security objectives, threats, and countermeasures
- **Engage NSM early**: Notify NSM of the planned system and request guidance on specific requirements. NSM may have additional requirements not covered in published guidance

#### Phase 2: Control Implementation

- Build the platform per this architecture
- Implement all NSM Grunnprinsipper measures
- Document every control implementation with evidence
- Develop system-specific security documentation:
  - System security plan (Sikkerhetsinstruks)
  - Operating procedures
  - Incident response procedures
  - Personnel security procedures
  - Physical security documentation

#### Phase 3: Assessment

- **Internal assessment**: The organization's own security team verifies all controls before engaging NSM
- **NSM assessment**: NSM conducts security audit (sikkerhetsgodkjenning). This includes:
  - Review of security documentation
  - Technical testing (penetration testing, configuration review)
  - Physical security inspection
  - Personnel security verification
  - Process and procedure evaluation
- **Findings remediation**: Address any findings from NSM assessment

#### Phase 4: Authorization

- NSM grants approval to operate (driftsgodkjenning) for HEMMELIG processing
- Conditions of approval documented (any restrictions, time limits, required reviews)
- System enters operational status

#### Phase 5: Continuous Monitoring

- Automated compliance scanning (OpenSCAP if RHEL, or custom scripts for Talos)
- Continuous monitoring dashboard showing security posture
- Regular vulnerability assessments
- Incident reporting to NSM per Sikkerhetsloven requirements
- Annual security review
- Significant changes trigger reassessment (new hardware, major software updates, architectural changes)

### Accreditation Documentation Deliverables

| Document | Description |
|----------|-------------|
| System Security Plan | Overall security posture, controls, and rationale |
| Architecture Document | This document |
| Risk Assessment | Threat analysis, risk evaluation, residual risk acceptance |
| Physical Security Plan | Facility security measures for the data center |
| Personnel Security Plan | Clearance requirements, access management procedures |
| Incident Response Plan | Procedures for security incidents |
| Configuration Management Plan | How changes are controlled and tracked |
| Audit and Monitoring Plan | What is logged, how it is monitored, retention policies |
| Business Continuity Plan | DR procedures, backup strategy, recovery testing |
| Interconnection Security Agreement | If connecting to NATO or other classified networks |

---

## 17. Supply Chain Security

### Hardware Supply Chain

- All server and network hardware procured from vendors with documented supply chain integrity practices
- Procurement through channels compliant with Sikkerhetsgraderte anskaffelser
- Tamper-evident packaging required for all shipments
- Chain-of-custody documentation from vendor warehouse to secure facility
- Firmware verification against vendor-published hashes upon receipt
- Hardware stored in secure facility from receipt until rack deployment

### Software Supply Chain

- All software artifacts (container images, OS packages, Helm charts) verified before ingress to HEMMELIG environment
- **Container images**: Pulled from upstream on connected network, rebuilt from source where possible, scanned with Trivy/Grype, SBOM generated with Syft, signed with cosign using organizational key
- **OS packages**: Mirrored from vendor repositories (Red Hat for RHEL, Talos release artifacts for Talos), GPG signature verified
- **Helm charts**: Version-pinned, stored in internal chart repository, reviewed before deployment
- **No arbitrary internet content enters the classified environment**: Everything is curated, scanned, and approved

### SBOM Management

- SBOMs generated for every container image deployed in the environment
- SBOMs stored in Harbor alongside the images
- Vulnerability correlation: New CVE disclosures checked against stored SBOMs to identify affected components
- SBOM format: SPDX or CycloneDX (both are industry standards)

---

## 18. Implementation Roadmap

### Phase 0: Preparation (Months 1-2)

- Finalize facility security assessment and any required upgrades to meet HEMMELIG physical security requirements
- Procure hardware with proper supply chain documentation
- Engage NSM for initial consultation and guidance
- Build unclassified development/test environment (mirror architecture) for the 12 uncleared team members
- Begin klarering applications for additional team members
- Develop automation playbooks and Helm charts on unclassified environment

### Phase 1: Foundation (Months 3-4)

- Receive and verify hardware (tamper checks, firmware verification)
- Rack and cable infrastructure in secure facility
- Deploy network fabric (spine-leaf switches)
- Install base OS on all nodes (Talos or hardened RHEL)
- Deploy RKE2 control plane (3 server nodes)
- Deploy Rook-Ceph storage cluster
- Validate basic cluster operations

### Phase 2: Platform Services (Months 5-6)

- Deploy FreeIPA and Keycloak
- Deploy Harbor registry and import initial container images via approved transfer
- Deploy ArgoCD and configure GitOps pipeline
- Deploy Gitea and import infrastructure code
- Deploy Vault for secrets management
- Deploy monitoring stack (Prometheus, Grafana, Loki, Alertmanager)
- Deploy Cilium with network policies and transparent encryption
- Deploy Kyverno admission policies
- Deploy Falco and Tetragon

### Phase 3: Security Hardening and Testing (Months 7-8)

- Comprehensive security hardening pass
- Penetration testing (by cleared security personnel)
- Compliance scanning against NSM Grunnprinsipper
- DR testing (backup, restore, failover)
- Performance testing and tuning
- Documentation completion (all accreditation documents)
- Internal security assessment

### Phase 4: Accreditation (Months 9-12)

- Submit documentation to NSM
- Support NSM security audit
- Remediate any findings
- Obtain driftsgodkjenning (operational approval)

### Phase 5: Application Onboarding (Month 12+)

- Deploy first classified workloads
- Establish NATO data sharing procedures
- Begin operational monitoring and continuous compliance
- Initiate continuous improvement cycle

**Total estimated timeline: 12-15 months to operational classified workloads**

This timeline assumes facility readiness and timely hardware procurement. The HEMMELIG facility approval and NSM accreditation are on the critical path and may extend the timeline.

---

## 19. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|-----------|--------|------------|
| R1 | NSM accreditation delayed due to findings | Medium | High | Engage NSM early; conduct thorough internal assessment before formal audit |
| R2 | Cleared personnel attrition (losing people from the 8) | Medium | Critical | Cross-train all roles; accelerate additional klarering applications; competitive retention packages |
| R3 | Hardware failure with extended replacement time (air-gapped, no just-in-time) | Medium | High | Maintain on-site spares inventory: 2 spare nodes, spare switches, spare drives |
| R4 | Software vulnerability discovered in deployed component | High | Medium | Monthly patch cycle via air-gap transfer; expedited process for critical CVEs; monitor CVE databases on connected side |
| R5 | TEMPEST assessment reveals inadequate shielding | Low | High | Commission TEMPEST survey early in Phase 0; budget for facility upgrades |
| R6 | Ceph cluster degradation or data loss | Low | Critical | Replication factor 3; regular scrubbing; monitoring with immediate alerts; tested restore procedures |
| R7 | Supply chain compromise of hardware or software | Low | Critical | Tamper-evident shipping; firmware verification; image scanning; SBOM tracking; procure from trusted vendors |
| R8 | Insufficient team capacity (8 people for platform + applications) | High | High | Maximize automation; simplify architecture; accelerate clearances; consider split between platform and application teams |
| R9 | NATO interoperability requirements change | Low | Medium | Monitor STANAG updates; maintain modular data format transformation pipeline |
| R10 | Data diode or CDS procurement delay | Medium | High | Begin procurement in Phase 0; these are specialized products with long lead times |

---

## 20. Architectural Decision Records

### ADR-001: Kubernetes over OpenStack

**Status**: Accepted

**Context**: The platform must support classified workloads in an air-gapped environment with only 8 cleared operators.

**Decision**: Use Kubernetes (RKE2) with KubeVirt instead of OpenStack.

**Rationale**: Smaller operational footprint, better air-gap tooling, defense sector momentum, and feasible operation by a team of 8. KubeVirt provides VM support where needed. See Section 3 for detailed analysis.

**Consequences**: Teams familiar with OpenStack will need Kubernetes training. Some VM-centric workloads may need containerization or KubeVirt migration.

---

### ADR-002: RKE2 as Kubernetes Distribution

**Status**: Accepted

**Context**: Need a Kubernetes distribution suitable for classified environments with air-gap support.

**Decision**: Use RKE2 (Rancher Government).

**Rationale**: Ships hardened (CIS compliant by default), explicit air-gap installation support, FIPS mode available, active government/defense adoption. Alternatives considered: Talos (strong but less defense track record), OpenShift (heavy, expensive licensing), kubeadm (too manual for this context).

**Consequences**: Depends on SUSE/Rancher for distribution support. Mitigated by RKE2 being open source.

---

### ADR-003: Cilium as CNI

**Status**: Accepted

**Context**: Need a CNI that provides network policy enforcement, encryption, and observability.

**Decision**: Use Cilium.

**Rationale**: eBPF-based performance, granular network policies (L3-L7), transparent encryption (WireGuard or IPsec), Hubble for network observability, active CNCF development. Alternative considered: Calico (mature but less observability, no native transparent encryption).

**Consequences**: Requires Linux kernel 5.10+ (standard on RHEL 9 or Talos). Team needs Cilium-specific training. Must verify WireGuard crypto primitives with NSM; fall back to IPsec if needed.

---

### ADR-004: Rook-Ceph for Storage

**Status**: Accepted

**Context**: Need unified block, file, and object storage within the Kubernetes cluster.

**Decision**: Use Rook-Ceph.

**Rationale**: Provides all three storage types (RBD, CephFS, RGW), mature integration with Kubernetes via CSI, encryption at rest support, self-healing, well-understood operational model. Alternative considered: Longhorn (simpler but block-only, no file/object).

**Consequences**: Ceph has operational complexity. Mitigated by Rook operator automation and dedicated monitoring.

---

### ADR-005: Talos Linux as Node OS (Primary) with RHEL as Alternative

**Status**: Proposed (pending NSM guidance)

**Context**: Need a minimal, secure operating system for cluster nodes.

**Decision**: Prefer Talos Linux; fall back to hardened RHEL 9 if NSM requires a commercially supported OS.

**Rationale**: Talos is immutable and API-only, eliminating entire classes of attack (no SSH, no shell). RHEL is the safe choice if NSM or Forsvaret has specific OS certification requirements.

**Consequences**: Talos requires all management through talosctl API — no traditional Linux troubleshooting. Ops team needs training. If RHEL is selected, additional hardening effort required.

---

### ADR-006: NSM Grunnprinsipper as Primary Hardening Baseline

**Status**: Accepted

**Context**: Need to select the primary security hardening baseline for the platform.

**Decision**: Use NSM Grunnprinsipper for IKT-sikkerhet as the primary baseline. CIS Benchmarks as supplementary guidance where NSM does not provide specific technical controls. Do NOT use DISA STIGs as primary reference.

**Rationale**: This is a Norwegian defense system accredited by NSM. NSM's own guidelines are the authoritative baseline. Using US standards (DISA STIGs, NIST) as primary would be inappropriate and could create friction during accreditation. CIS Benchmarks are internationally recognized and complementary.

**Consequences**: Some technical hardening details may need to be derived from CIS or other sources where NSM guidance is high-level. All such derivations must be documented and justified.

---

### ADR-007: Physical Air Gap with Data Diode for Ingress

**Status**: Accepted

**Context**: HEMMELIG classification requires physical separation from lower-classification networks.

**Decision**: Fully physical air gap. Data ingress via hardware data diode (Advenica or equivalent). Manual sneakernet for media transfer where diode is not suitable.

**Rationale**: Mandatory for HEMMELIG per Sikkerhetsloven. Logical separation (firewalls, VLANs) is not sufficient. Data diodes provide hardware-enforced one-way transfer for operational efficiency while maintaining the air gap.

**Consequences**: All software updates, patches, and data must be physically transferred. This adds latency to operations but is non-negotiable for the classification level.

---

## Appendix A: Technology Stack Summary

| Category | Technology | License | Version Policy |
|----------|-----------|---------|---------------|
| Kubernetes | RKE2 | Apache 2.0 | Track stable channel, monthly updates |
| Node OS | Talos Linux or RHEL 9 | MPL 2.0 / Commercial | LTS releases |
| CNI | Cilium | Apache 2.0 | Track latest stable |
| Storage | Rook-Ceph | Apache 2.0 | Track latest stable |
| VM Workloads | KubeVirt | Apache 2.0 | Track latest stable |
| GitOps | ArgoCD | Apache 2.0 | Track latest stable |
| Registry | Harbor | Apache 2.0 | Track latest stable |
| Identity | FreeIPA | GPL | Track latest stable |
| SSO | Keycloak | Apache 2.0 | Track latest stable |
| Secrets | HashiCorp Vault | BSL 1.1 | Track latest stable |
| Monitoring | Prometheus | Apache 2.0 | Track latest stable |
| Dashboards | Grafana | AGPL 3.0 | Track latest stable |
| Logging | Loki | AGPL 3.0 | Track latest stable |
| Policy | Kyverno | Apache 2.0 | Track latest stable |
| Runtime Security | Falco | Apache 2.0 | Track latest stable |
| Runtime Enforcement | Tetragon | Apache 2.0 | Track latest stable |
| Image Scanning | Trivy | Apache 2.0 | Track latest stable |
| Backup | Velero | Apache 2.0 | Track latest stable |
| Git Server | Gitea | MIT | Track latest stable |
| CI/CD | Tekton | Apache 2.0 | Track latest stable |
| IaC | OpenTofu | MPL 2.0 | Track latest stable |
| Config Mgmt | Ansible | GPL 3.0 | Track latest stable |
| DCIM/IPAM | NetBox | Apache 2.0 | Track latest stable |
| Data Diode | Advenica SecuriCDS | Commercial | Vendor-supported |

## Appendix B: Network Port Matrix

| Source | Destination | Port | Protocol | Purpose |
|--------|------------|------|----------|---------|
| Admin workstations | K8s API (kube-vip VIP) | 6443 | TCP/TLS | Kubernetes API access |
| Admin workstations | Talos API | 50000 | TCP/TLS | Talos node management |
| Worker nodes | K8s API | 6443 | TCP/TLS | Kubelet registration |
| Control plane nodes | etcd peers | 2380 | TCP/TLS | Etcd cluster communication |
| Control plane nodes | etcd clients | 2379 | TCP/TLS | Etcd client access |
| All nodes | All nodes | 4240 | TCP | Cilium health checks |
| All nodes | All nodes | 8472 | UDP | VXLAN (Cilium overlay) |
| All nodes | All nodes | 51871 | UDP | WireGuard (Cilium encryption) |
| All nodes | Ceph MON | 6789, 3300 | TCP | Ceph monitor |
| OSD nodes | OSD nodes | 6800-7300 | TCP | Ceph OSD communication |
| All nodes | Harbor | 443 | TCP/TLS | Container image pull |
| All nodes | DNS (CoreDNS) | 53 | TCP/UDP | DNS resolution |
| Admin workstations | Grafana | 443 | TCP/TLS | Monitoring dashboards |
| Admin workstations | ArgoCD | 443 | TCP/TLS | GitOps management |
| Admin workstations | Keycloak | 443 | TCP/TLS | SSO authentication |
| Admin workstations | Vault | 443 | TCP/TLS | Secrets management |
| Admin workstations | Gitea | 443, 22 | TCP/TLS, SSH | Code repository |

## Appendix C: Glossary of Norwegian Terms

| Norwegian | English | Context |
|-----------|---------|---------|
| Sikkerhetsloven | Security Act | Primary legislation governing national security |
| Sikkerhetsgradert | Security-graded | Classified |
| Klarering | Security clearance | Personnel vetting for classified access |
| Sikkerhetssamtale | Security interview | Part of clearance process |
| Sikkerhetsansvarlig | Security officer | Organizational role responsible for security |
| NSM | Nasjonal Sikkerhetsmyndighet | National Security Authority |
| Grunnprinsipper for IKT-sikkerhet | Basic Principles for ICT Security | NSM's primary ICT security framework |
| HEMMELIG | Secret | Fourth classification level |
| STRENGT HEMMELIG | Top Secret | Fifth (highest) classification level |
| KONFIDENSIELT | Confidential | Third classification level |
| BEGRENSET | Restricted | Second classification level |
| UGRADERT | Unclassified | Lowest level (no classification) |
| Sikkerhetsgradert omrade | Security-graded area | Physically protected area for classified processing |
| Sperret omrade | Restricted area | Controlled access zone |
| Virksomhetsikkerhetsforskriften | Enterprise Security Regulation | Regulation under Sikkerhetsloven |
| Klareringsforskriften | Clearance Regulation | Regulation governing clearance process |
| Sikkerhetsgraderte anskaffelser | Security-graded procurements | Regulation on classified contracts |
| Driftsgodkjenning | Operational approval | Authorization to operate classified system |
| Forsvaret | The Norwegian Armed Forces | Customer |
