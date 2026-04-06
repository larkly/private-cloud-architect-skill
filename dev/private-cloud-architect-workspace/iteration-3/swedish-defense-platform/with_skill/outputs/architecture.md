# Architecture: Private Cloud Platform for HEMLIG Classified Data Processing

## Swedish Defense Contractor -- Säkerhetsskyddslagen-Compliant Private Cloud

**Classification**: HEMLIG (Secret)
**Regulatory Framework**: Säkerhetsskyddslagen (2018:585), FMV/MUST accreditation
**Document Status**: Architecture Design Document
**Date**: 2026-03-20

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Requirements Analysis](#2-requirements-analysis)
3. [Regulatory and Accreditation Framework](#3-regulatory-and-accreditation-framework)
4. [Platform Selection and Rationale](#4-platform-selection-and-rationale)
5. [Physical Infrastructure](#5-physical-infrastructure)
6. [Network Architecture](#6-network-architecture)
7. [Compute Architecture](#7-compute-architecture)
8. [Storage Architecture](#8-storage-architecture)
9. [Kubernetes Platform Design](#9-kubernetes-platform-design)
10. [Identity, Access, and Personnel Controls](#10-identity-access-and-personnel-controls)
11. [Air-Gapped Operations](#11-air-gapped-operations)
12. [Data Flow and Cross-Domain Architecture](#12-data-flow-and-cross-domain-architecture)
13. [Security Architecture](#13-security-architecture)
14. [Observability and Audit](#14-observability-and-audit)
15. [Automation and Infrastructure as Code](#15-automation-and-infrastructure-as-code)
16. [Disaster Recovery and Business Continuity](#16-disaster-recovery-and-business-continuity)
17. [Staffing and Operational Model](#17-staffing-and-operational-model)
18. [Accreditation Roadmap](#18-accreditation-roadmap)
19. [Risk Register](#19-risk-register)
20. [Architectural Decision Records](#20-architectural-decision-records)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform designed to process data classified at the Swedish HEMLIG (Secret) level. The platform is purpose-built to meet the requirements of Säkerhetsskyddslagen (2018:585) and its associated regulations, and is designed for accreditation by FMV (Försvarets materielverk) in coordination with MUST (Militära underrättelse- och säkerhetstjänsten).

The platform is a fully air-gapped, Kubernetes-based private cloud deployed within Swedish territory, operated exclusively by personnel holding Swedish security clearances (säkerhetsprövade) at the appropriate level, granted via Säkerhetspolisen (Säpo). The architecture prioritizes FLOSS (Free/Libre Open Source Software) components to reduce supply chain risk and eliminate dependency on foreign-controlled commercial licensing.

**Key architectural characteristics:**

- Fully air-gapped -- no connectivity to the internet or lower-classification networks
- All data resident within Sweden, all processing within Swedish sovereign territory
- Kubernetes (RKE2) on bare-metal with Cilium networking and Rook-Ceph storage
- Hardware-enforced data diodes for controlled inbound data transfer
- TEMPEST considerations for emanation security at HEMLIG level
- 12 cleared personnel forming the operations team, with defined role separation
- GitOps-driven operations with offline ArgoCD
- Continuous compliance monitoring for sustained accreditation

---

## 2. Requirements Analysis

### 2.1 Business Requirements

| Requirement | Detail |
|---|---|
| Classification Level | HEMLIG (Secret) per Swedish classification scheme |
| Regulatory Compliance | Säkerhetsskyddslagen (2018:585), Säkerhetsskyddsförordningen |
| Accreditation Authority | FMV/MUST |
| Data Sovereignty | All data must remain within Swedish national borders |
| Air Gap | Full physical air gap -- no internet, no connection to lower-classification networks |
| Personnel | All operators must hold Swedish security clearances via Säpo |
| Platform Type | Container-native (Kubernetes-based), with VM capability for legacy workloads |
| Total Engineering Staff | 40 engineers |
| Cleared Staff | 12 engineers cleared at HEMLIG level |
| Availability Target | 99.95% (acknowledging air-gapped maintenance constraints) |

### 2.2 Workload Characteristics (Anticipated)

- Defense data processing and analytics applications
- Classified communication and collaboration tools
- Mission planning and simulation systems
- Secure software development environments (classified CI/CD)
- Document management and records systems
- Potentially: ML/AI inference workloads on classified data

### 2.3 International Context

As a Swedish defense contractor, the platform must be designed with awareness of:

- **EU obligations**: GDPR (though national security exemptions apply for classified processing), NIS2 Directive applicability for defense sector entities, EUCS (EU Cybersecurity Certification Scheme) as it matures
- **NATO context**: Sweden is a NATO member. NATO interoperability requirements (STANAG compliance, potential for NATO SECRET handling) should be considered in the architecture so the platform can be extended or federated in the future without fundamental redesign
- **Bilateral agreements**: Defense data sharing agreements with partner nations may impose additional requirements on data handling and compartmentalization

---

## 3. Regulatory and Accreditation Framework

### 3.1 Primary Legal Framework

**Säkerhetsskyddslagen (2018:585)** -- the Swedish Protective Security Act -- is the governing law. It covers:

- **Informationssäkerhet** (Information security): requirements for protecting classified information
- **Fysisk säkerhet** (Physical security): requirements for physical protection of facilities and equipment
- **Personalsäkerhet** (Personnel security): requirements for security clearances and personnel vetting
- **Säkerhetsskyddad upphandling** (Security-protected procurement): requirements for procurement of goods and services that touch classified information

The implementing regulation is **Säkerhetsskyddsförordningen (2018:658)**.

### 3.2 Accreditation Authorities

| Authority | Role |
|---|---|
| **FMV** (Försvarets materielverk) | Primary accreditation body for defense material and systems. Conducts technical security assessments and grants accreditation for IT systems processing classified defense information. |
| **MUST** (Militära underrättelse- och säkerhetstjänsten) | Military intelligence service. Provides SIGINT/TEMPEST guidance, approves cryptographic solutions, and advises on signals security and emanation requirements. |
| **Säpo** (Säkerhetspolisen) | Security Service. Responsible for personnel security clearance (säkerhetsprövning). Conducts background checks and grants clearances. |
| **FRA** (Försvarets radioanstalt) | National Defense Radio Establishment. Evaluates and approves cryptographic products for classified use. Provides TEMPEST evaluation services. |
| **MSB** (Myndigheten för samhällsskydd och beredskap) | Civil Contingencies Agency. Provides general cybersecurity guidance and national CERT function. Less directly involved at HEMLIG level but relevant for baseline security practices. |

### 3.3 Accreditation Process for This Platform

The accreditation follows the general pattern mandated by Säkerhetsskyddslagen:

1. **Säkerhetsskyddsanalys** (Protective Security Analysis) -- formal risk assessment identifying what needs protection, from whom, and what controls are needed. This must be completed and approved BEFORE system design is finalized.
2. **System Security Plan (Systemsäkerhetsplan)** -- detailed documentation of all security controls, mapping to Säkerhetsskyddslagen requirements and FMV/MUST guidance.
3. **Implementation** -- build the system with all controls in place.
4. **Säkerhetsgranskning** (Security Review) -- FMV conducts technical security assessment. MUST provides signals security review. Physical security inspection of the facility.
5. **Ackreditering** (Accreditation) -- FMV grants accreditation to process HEMLIG data. This is time-limited and conditional.
6. **Kontinuerlig uppföljning** (Continuous Monitoring) -- ongoing compliance monitoring with periodic reassessment. Significant changes trigger reaccreditation.

**Critical design principle**: The architecture in this document is designed to support accreditation, but the Säkerhetsskyddsanalys MUST be completed first and may impose additional or modified requirements. Never design the infrastructure first and try to get it accredited later.

### 3.4 Cryptographic Requirements

At the HEMLIG level:

- All cryptographic solutions must be approved by FRA (Försvarets radioanstalt) or MUST
- Swedish/European cryptographic products are preferred; MUST maintains a list of approved products
- US FIPS 140-2/140-3 validation is NOT sufficient on its own -- FRA/MUST approval is the primary requirement
- Encryption at rest and in transit is mandatory
- Key management procedures must be documented and approved as part of accreditation
- Hardware Security Modules (HSMs) should be FRA-approved for key storage

---

## 4. Platform Selection and Rationale

### 4.1 Decision: Kubernetes on Bare Metal (RKE2)

**Selected platform**: RKE2 (Rancher Kubernetes Engine 2) deployed on bare-metal servers.

**Rationale**:

| Factor | Assessment |
|---|---|
| **Security posture** | RKE2 is designed as a security-focused Kubernetes distribution. It runs with SELinux enforcing by default, uses embedded etcd (no external dependency), and is the basis for the US DoD's "Rancher Government" offering. While US DoD accreditation is not authoritative in Sweden, it demonstrates the distribution has undergone rigorous security scrutiny. |
| **Air-gapped support** | RKE2 has first-class support for air-gapped installation with pre-packaged binaries and images. |
| **CIS hardening** | RKE2 ships with CIS Kubernetes Benchmark compliance by default, reducing hardening effort. |
| **FIPS crypto modules** | RKE2 Government edition includes FIPS-validated Go crypto modules. While FIPS is not the Swedish requirement, having validated crypto modules provides a strong baseline that can be mapped to FRA-approved equivalents. |
| **FLOSS** | RKE2 is Apache 2.0 licensed -- fully open source, no proprietary licensing dependency, auditable source code. |
| **VM support** | KubeVirt can be deployed on RKE2 for legacy VM workloads, providing a converged platform. |
| **Operational simplicity** | Single-binary deployment, embedded etcd, built-in containerd runtime -- reduces operational complexity for a 12-person team. |

**Alternatives considered and rejected**:

| Alternative | Reason for Rejection |
|---|---|
| OpenShift | Strong security credentials (Common Criteria certified) but Red Hat subscription licensing creates foreign vendor dependency. Heavier operational footprint for 12-person team. Could be reconsidered if Red Hat establishes Swedish support presence. |
| Talos Linux | Excellent security properties (immutable, API-only) but smaller community, less proven in classified environments. Worth evaluating for future iterations. |
| Vanilla kubeadm | Too much undifferentiated heavy lifting for hardening and lifecycle management with a small team. |
| k3s | Designed for edge/lightweight -- lacks the security hardening defaults of RKE2. |
| OpenStack | VM-centric, larger operational footprint (Keystone, Nova, Neutron, Cinder, Glance, etc.). Requires more than 12 people to operate effectively. Not aligned with container-native strategy. |
| Nutanix / VMware | Proprietary licensing, foreign vendor dependency, not appropriate for HEMLIG where source code auditability and supply chain control matter. |

### 4.2 KubeVirt for Legacy VM Workloads

Some defense applications will require traditional VM hosting. Rather than running a separate hypervisor platform, KubeVirt is deployed within the Kubernetes cluster to provide VM capability:

- Single platform to operate (reduces operational burden on 12-person team)
- VMs and containers share the same networking, storage, and security policies
- Gradual migration path from VM-based to container-native workloads
- CDI (Containerized Data Importer) for managing VM disk images
- SR-IOV for high-performance network passthrough to VMs where needed

---

## 5. Physical Infrastructure

### 5.1 Facility Requirements

The data center facility must meet the physical security requirements mandated by Säkerhetsskyddslagen for HEMLIG processing:

- **Säkerhetsskyddat utrymme** (Security-protected area) at the level required for HEMLIG
- Controlled access with multi-factor authentication (badge + biometric or PIN)
- 24/7 physical security monitoring (guards, CCTV, intrusion detection)
- Visitor management with escort requirements
- Located within Sweden -- no exceptions
- Physical separation from non-classified areas
- Access logging with long-term retention

### 5.2 TEMPEST and Emanation Security

At the HEMLIG level, TEMPEST (emanation security) requirements apply. Consult MUST for specific requirements, but design for:

- **SDIP-27 Zone considerations**: equipment placement must account for emanation boundaries. The facility may require shielded rooms (Faraday cages) depending on the proximity of unclassified systems or public areas.
- **Red/black separation**: strict physical separation between classified (red) signal paths and unclassified (black) signal paths. Classified and unclassified cabling must never share conduits, raceways, or cable trays.
- **TEMPEST-rated equipment**: where mandated by MUST, servers, switches, and peripherals must be TEMPEST-rated (NATO SDIP-27 Level A, B, or C as required).
- **Cable routing**: minimum separation distances between red and black cables as specified by MUST guidance.
- **Workstations**: TEMPEST-rated KVM switches if multi-level workstations are used; otherwise, dedicated terminals within the secure area.

**Note**: MUST will specify exact TEMPEST requirements during the Säkerhetsskyddsanalys. The architecture must be flexible enough to accommodate these requirements, which may affect rack placement, cable routing, and equipment selection.

### 5.3 Hardware Architecture

#### 5.3.1 Server Specification (Reference)

The following is a reference specification. Final hardware selection must go through säkerhetsskyddad upphandling (security-protected procurement) with supply chain verification.

**Control Plane Nodes (3x)**:

| Component | Specification |
|---|---|
| Form Factor | 1U rackmount |
| CPU | 2x Intel Xeon Gold (or AMD EPYC) -- 16+ cores per socket |
| RAM | 256 GB DDR5 ECC |
| Boot | 2x 480 GB NVMe SSD (mirrored, OS + etcd) |
| Network | 2x 25 GbE (cluster), 1x 1 GbE (IPMI/management) |
| BMC | IPMI 2.0 / Redfish (on dedicated management network) |

**Worker Nodes (8x minimum, expandable)**:

| Component | Specification |
|---|---|
| Form Factor | 2U rackmount |
| CPU | 2x Intel Xeon Gold (or AMD EPYC) -- 32+ cores per socket |
| RAM | 512 GB DDR5 ECC |
| Boot | 2x 480 GB NVMe SSD (mirrored, OS) |
| Storage (Ceph OSD) | 4x 3.84 TB NVMe SSD (Ceph OSD) + 2x 1.6 TB NVMe (Ceph WAL/DB) |
| Network | 2x 25 GbE (cluster/workload), 2x 25 GbE (Ceph storage), 1x 1 GbE (IPMI) |
| GPU (optional) | NVIDIA GPU (if ML/AI inference workloads required) |

**Infrastructure Nodes (3x)** -- dedicated to platform services (monitoring, logging, registry, GitOps):

| Component | Specification |
|---|---|
| Form Factor | 1U rackmount |
| CPU | 2x Intel Xeon Gold -- 16+ cores per socket |
| RAM | 256 GB DDR5 ECC |
| Boot | 2x 480 GB NVMe SSD (mirrored) |
| Storage | 2x 3.84 TB NVMe SSD (local storage for logs/metrics) |
| Network | 2x 25 GbE (cluster), 1x 1 GbE (IPMI) |

#### 5.3.2 Network Hardware

| Component | Quantity | Purpose |
|---|---|---|
| Spine switches | 2x | 100 GbE spine layer |
| Leaf switches | 4x | 25 GbE leaf layer (server connectivity) |
| Management switches | 2x | 1 GbE out-of-band management (IPMI/BMC) |
| Data diode | 1x | Hardware-enforced one-way data ingest (see Section 12) |
| Terminal servers | 2x | Console access to all nodes |

**Switch selection**: Prefer switches that support open network operating systems (Cumulus Linux / NVIDIA Spectrum, SONiC, or equivalent FLOSS NOS). If Cisco is selected for organizational alignment, use Nexus 9000 series in NX-OS mode with VXLAN-EVPN fabric. The key requirement is that the switch firmware can be verified and that the supply chain is documented for the accreditation.

#### 5.3.3 Supply Chain Security

All hardware must be procured through **säkerhetsskyddad upphandling**:

- Procurement contracts must include security clauses per Säkerhetsskyddslagen
- Chain-of-custody documentation from manufacturer to rack
- Tamper-evident packaging verification on receipt
- Firmware verification against manufacturer checksums before deployment
- BIOS/UEFI settings hardened (Secure Boot enabled, boot order locked, unused ports disabled)
- BMC firmware verified and updated to latest patched version
- All hardware inventory recorded with serial numbers, firmware versions, and physical location

---

## 6. Network Architecture

### 6.1 Network Design Principles

- **Complete air gap**: no physical connection to the internet or any lower-classification network
- **Red/black separation**: classified network is the "red" network; management/BMC network requires careful classification assessment
- **Spine-leaf topology**: two-tier Clos network for scalability and predictable latency
- **VXLAN-EVPN overlay**: scalable Layer 2/Layer 3 segmentation over the spine-leaf fabric
- **Microsegmentation**: Cilium network policies enforce pod-level segmentation inside Kubernetes
- **Encrypted east-west traffic**: WireGuard-based encryption for all pod-to-pod traffic via Cilium

### 6.2 Network Topology

```
                    ┌──────────────────────────────────────────────────┐
                    │           AIR-GAPPED BOUNDARY                    │
                    │  (No connectivity to external networks)          │
                    └──────────────────────────────────────────────────┘

                         ┌─────────┐     ┌─────────┐
                         │ Spine-1 │─────│ Spine-2 │
                         │ 100 GbE │     │ 100 GbE │
                         └────┬┬───┘     └───┬┬────┘
                              ││             ││
                    ┌─────────┘│     ┌───────┘│
                    │  ┌───────┘     │  ┌─────┘
                    │  │             │  │
               ┌────┴──┴──┐    ┌────┴──┴──┐    ┌──────────┐    ┌──────────┐
               │  Leaf-1   │   │  Leaf-2   │   │  Leaf-3   │   │  Leaf-4   │
               │  25 GbE   │   │  25 GbE   │   │  25 GbE   │   │  25 GbE   │
               └─────┬─────┘   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                     │               │               │               │
              ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
              │ Control     │ │ Worker      │ │ Worker      │ │ Infra       │
              │ Plane (3x)  │ │ Nodes (4x)  │ │ Nodes (4x)  │ │ Nodes (3x)  │
              └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

              ┌──────────────────────────────────────────────────────────┐
              │  OUT-OF-BAND MANAGEMENT NETWORK (Separate switches)      │
              │  IPMI/BMC access, console servers                        │
              │  1 GbE -- physically separate cabling                    │
              └──────────────────────────────────────────────────────────┘

              ┌──────────────────────────────────────────────────────────┐
              │  CEPH STORAGE NETWORK (Dedicated VLAN on leaf switches)  │
              │  25 GbE dedicated NICs on worker nodes                   │
              └──────────────────────────────────────────────────────────┘
```

### 6.3 Network Segmentation

| Network Segment | VLAN/VNI | Purpose | Access Control |
|---|---|---|---|
| **Control Plane** | VLAN 10 / VNI 10010 | Kubernetes API, etcd | Restricted to control plane nodes and admin workstations |
| **Pod Network (overlay)** | Cilium VXLAN | Pod-to-pod communication | Cilium network policies (default deny) |
| **Service Network** | Cilium-managed | Kubernetes Service ClusterIPs | Cilium network policies |
| **Ceph Storage** | VLAN 20 / VNI 10020 | Ceph OSD replication, client traffic | Restricted to nodes running Ceph components |
| **Ceph Cluster** | VLAN 21 / VNI 10021 | Ceph internal cluster replication | Restricted to OSD nodes only |
| **Management/BMC** | VLAN 99 | IPMI, Redfish, console | Restricted to admin jump hosts; physically separate |
| **MetalLB Ingress** | VLAN 30 / VNI 10030 | Load balancer VIPs for workload ingress | Application-specific access |
| **Data Diode Ingest** | VLAN 40 | Receiving side of data diode | Restricted to ingest processing pipeline |

### 6.4 Cilium Configuration

Cilium is the CNI (Container Network Interface) plugin, selected for:

- **eBPF-based networking**: high performance, kernel-level packet processing
- **Network policy enforcement**: L3/L4 and L7 policy with identity-based controls
- **Transparent encryption**: WireGuard-based node-to-node encryption for all pod traffic
- **No dependency on kube-proxy**: Cilium replaces kube-proxy with eBPF, reducing attack surface
- **Hubble observability**: network flow visibility for audit and troubleshooting
- **Host firewall**: Cilium host policies protect the nodes themselves, not just pods

**Default network policy**: All namespaces receive a default-deny ingress and egress policy. Workloads must have explicit allow policies to communicate. This is mandatory for HEMLIG.

### 6.5 DNS

- CoreDNS deployed as the cluster DNS (ships with RKE2)
- Internal DNS zones for cluster services
- No external DNS resolution (air-gapped)
- DNS query logging enabled for audit

### 6.6 Load Balancing

- **MetalLB** in Layer 2 or BGP mode for Kubernetes Service type LoadBalancer
- **kube-vip** for control plane API server HA (virtual IP)
- No external load balancers required in the air-gapped environment

---

## 7. Compute Architecture

### 7.1 Operating System

**Selected**: SUSE Linux Enterprise Micro (SLE Micro) or alternatively Rocky Linux / Alma Linux with FIPS-mode kernel.

**Rationale for SLE Micro**:
- Immutable operating system designed for container hosts
- Transactional updates with rollback capability
- Minimal attack surface -- no package manager in the running system
- SUSE has European headquarters and strong presence in Swedish defense
- Common Criteria evaluated
- If SLE Micro is not approved: Rocky Linux 9 with CIS Level 2 hardening and SELinux enforcing

**Alternative consideration**: Talos Linux (fully immutable, API-only, no SSH) provides the strongest security posture of any Kubernetes node OS. However, it has a smaller community and may raise questions during accreditation. Recommend evaluating Talos for future iterations once it gains broader adoption in European defense contexts.

### 7.2 Node Hardening

Regardless of OS choice, all nodes must be hardened:

- **SELinux**: enforcing mode, with custom policy modules as needed
- **Boot security**: UEFI Secure Boot enabled, Trusted Boot chain
- **Disk encryption**: LUKS2 encryption on all disks (OS, etcd, Ceph OSD). Encryption keys stored in TPM 2.0 with FRA-approved key hierarchy.
- **Kernel hardening**: sysctl parameters per CIS Benchmark and FMV guidance (disable ip_forward except where needed, restrict core dumps, randomize memory layout, etc.)
- **Firewall**: nftables host firewall in addition to Cilium host policies (defense in depth)
- **SSH**: disabled on immutable OS (SLE Micro / Talos); if enabled (Rocky Linux), restricted to key-based auth from jump host only, with session recording
- **Time synchronization**: Chrony NTP against internal NTP servers (no external NTP in air-gapped environment -- use GPS-disciplined NTP or rubidium clock)
- **Firmware**: verified against manufacturer checksums, updated through controlled process
- **Unnecessary services**: disabled. Minimal kernel modules loaded.
- **Audit**: auditd running with comprehensive rules per FMV hardening guide

### 7.3 NUMA and Performance Optimization

For performance-sensitive workloads:

- NUMA-aware scheduling enabled in kubelet
- CPU Manager policy set to "static" for guaranteed QoS pods
- Hugepages (2 Mi and 1 Gi) enabled for database and VM workloads (KubeVirt)
- Topology Manager policy set to "best-effort" or "restricted" depending on workload
- IRQ affinity configured for network-intensive workloads

---

## 8. Storage Architecture

### 8.1 Decision: Rook-Ceph

**Selected**: Rook-Ceph (Ceph deployed and managed by the Rook operator within Kubernetes).

**Rationale**:

- Software-defined storage with no proprietary licensing
- Provides block (RBD), file (CephFS), and object (RGW) storage from a single platform
- Rook operator manages Ceph lifecycle within Kubernetes (deployment, scaling, upgrades, health monitoring)
- Battle-tested at scale in both commercial and government environments
- Fully FLOSS (Apache 2.0 / LGPL)
- Eliminates need for external SAN/NAS infrastructure
- Encryption at rest via LUKS on OSD disks (dm-crypt managed by Rook)

### 8.2 Ceph Topology

```
┌──────────────────────────────────────────────────────────────┐
│                    ROOK-CEPH CLUSTER                          │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  MON/MGR  │  │  MON/MGR  │  │  MON/MGR  │  (3x monitors   │
│  │  (infra-1)│  │  (infra-2)│  │  (infra-3)│   on infra      │
│  └──────────┘  └──────────┘  └──────────┘   nodes)           │
│                                                              │
│  ┌──────────┐  ┌──────────┐       ┌──────────┐              │
│  │ OSD x4   │  │ OSD x4   │  ...  │ OSD x4   │  (8x worker  │
│  │(worker-1)│  │(worker-2)│       │(worker-8)│   nodes,      │
│  └──────────┘  └──────────┘       └──────────┘   4 OSDs each)│
│                                                              │
│  Total raw: ~123 TB (8 nodes x 4 x 3.84 TB NVMe)            │
│  Usable (3x replication): ~41 TB                             │
│  Usable (EC 4+2): ~82 TB                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 Storage Classes

| Storage Class | Ceph Pool | Replication | Use Case |
|---|---|---|---|
| `ceph-block-replicated` | RBD, 3x replication | 3x | Databases, etcd backups, high-IOPS workloads |
| `ceph-block-ec` | RBD, EC 4+2 | EC 4+2 | General block storage, VM disks (KubeVirt) |
| `ceph-filesystem` | CephFS, 3x replication | 3x | Shared filesystems (ReadWriteMany) |
| `ceph-object` | RGW | 3x | Object storage (S3-compatible), artifact storage |

### 8.4 Encryption at Rest

- All Ceph OSDs encrypted with LUKS2 via Rook's `encryptedDevice: true`
- LUKS keys sealed to TPM 2.0 on each node
- etcd data directory encrypted (RKE2 supports etcd encryption at rest with AES-CBC or AES-GCM)
- etcd encryption key rotation procedure documented

### 8.5 Backup

- **Velero** for Kubernetes resource backup (namespace-level backup/restore)
- **Ceph RBD snapshots** for point-in-time recovery of persistent volumes
- **Restic** (integrated with Velero) for file-level backup of PV data
- Backup data stored on dedicated Ceph pool with longer retention
- Backup media rotation for off-site storage in a secondary secure facility (if available)
- All backup media encrypted and handled per HEMLIG data handling procedures

---

## 9. Kubernetes Platform Design

### 9.1 Cluster Architecture

**Single cluster** with namespace-based multi-tenancy and strong isolation.

**Rationale for single cluster over multi-cluster**: With 12 cleared operators, the operational overhead of managing multiple clusters is not justified. A single well-configured cluster with namespace isolation, network policies, and RBAC provides adequate workload separation for same-classification-level tenants. If future requirements introduce compartmentalization within HEMLIG (e.g., different programs with different need-to-know), multi-cluster with Cluster API can be evaluated.

### 9.2 Node Roles

| Role | Count | Taints/Labels | Purpose |
|---|---|---|---|
| Control plane | 3 | `node-role.kubernetes.io/control-plane:NoSchedule` | API server, etcd, scheduler, controller manager |
| Infrastructure | 3 | `node-role.kubernetes.io/infra:NoSchedule` | Monitoring, logging, registry, GitOps, Ceph MON/MGR |
| Worker | 8+ | None (schedulable) | Application workloads, Ceph OSD |

### 9.3 RKE2 Configuration Highlights

```yaml
# /etc/rancher/rke2/config.yaml (control plane)
cni: cilium
disable:
  - rke2-ingress-nginx  # We deploy our own ingress
profile: cis-1.23       # CIS hardening profile
selinux: true
secrets-encryption: true
audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 28
tls-cipher-suites:
  - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
kube-apiserver-arg:
  - "anonymous-auth=false"
  - "authorization-mode=Node,RBAC"
  - "enable-admission-plugins=NodeRestriction,PodSecurityAdmission"
kubelet-arg:
  - "protect-kernel-defaults=true"
  - "streaming-connection-idle-timeout=5m"
  - "make-iptables-util-chains=true"
```

### 9.4 Namespaces and Tenancy

| Namespace Category | Examples | Purpose |
|---|---|---|
| **System** | `kube-system`, `rke2-*` | Core Kubernetes components |
| **Platform** | `monitoring`, `logging`, `registry`, `argocd`, `cert-manager`, `security` | Platform services |
| **Workload** | `project-alpha`, `project-bravo`, etc. | Application workloads, one namespace per project/program |
| **Data Ingest** | `data-ingest` | Data diode receiving pipeline |

Each workload namespace receives:

- Default deny NetworkPolicy (ingress and egress)
- ResourceQuota (CPU, memory, storage, pod count limits)
- LimitRange (default resource requests/limits per pod)
- Pod Security Standards: `restricted` profile enforced
- RBAC: project-specific roles bound to project team members
- Kyverno policies: image provenance, label requirements, etc.

### 9.5 Ingress

- **NGINX Ingress Controller** or **Envoy Gateway** deployed on infrastructure nodes
- TLS termination with certificates from internal PKI (cert-manager with internal CA)
- No external ingress (air-gapped) -- all ingress is from workstations within the secure facility
- Mutual TLS (mTLS) for service-to-service communication via Cilium

### 9.6 Container Registry

- **Harbor** deployed on infrastructure nodes
- All container images must be present in Harbor before they can be deployed
- Images imported via the data diode / sneakernet process (see Section 11)
- Image signing with Cosign (Sigstore) using internal signing keys
- Vulnerability scanning with Trivy (integrated into Harbor)
- Admission controller (Kyverno) blocks deployment of unsigned or unscanned images
- Notary v2 for image provenance attestation

### 9.7 Pod Security

- **Pod Security Admission (PSA)**: `restricted` profile enforced cluster-wide
- **Kyverno** policies:
  - Block privileged containers
  - Require non-root execution
  - Require read-only root filesystem
  - Enforce image pull from internal Harbor only
  - Require image signatures
  - Require resource limits
  - Enforce label standards
  - Block hostPath mounts
  - Block hostNetwork, hostPID, hostIPC
- **Falco** for runtime security monitoring (detects unexpected system calls, file access, network connections)
- **Tetragon** (Cilium) for eBPF-based runtime enforcement (complementary to Falco)

---

## 10. Identity, Access, and Personnel Controls

### 10.1 Personnel Security Context

Per Säkerhetsskyddslagen, all personnel with access to the HEMLIG platform must:

- Hold a valid Swedish security clearance (säkerhetsprövning) at the appropriate level, granted by the organization with Säpo conducting the background investigation
- Have a documented **need-to-know** for the specific data or systems they access
- Have signed a **tystnadsplikt** (non-disclosure commitment) per the Protective Security Act
- Undergo periodic reassessment of clearance
- Be Swedish citizens (typically required for HEMLIG-level clearance)

### 10.2 Team Structure (12 Cleared Personnel)

| Role | Count | Responsibilities |
|---|---|---|
| **Platform Lead / Security Officer** | 1 | Architecture ownership, accreditation liaison with FMV/MUST, security policy |
| **Platform Engineers** | 4 | Kubernetes operations, RKE2 lifecycle, Ceph storage, Cilium networking |
| **Security Engineers** | 2 | Security monitoring (Falco/Tetragon), audit log review, vulnerability management, compliance automation |
| **Application/DevOps Engineers** | 3 | Application onboarding, CI/CD pipeline management, developer support |
| **Data Ingest Operators** | 2 | Data diode operations, media handling, import/export procedures |

**On-call**: With 12 people, a sustainable on-call rotation is challenging. Design for maximum automation to minimize out-of-hours intervention. Recommend a 4-person primary on-call rotation (one week each, cycling monthly) with secondary escalation.

### 10.3 Identity Management

**Selected**: FreeIPA for identity management within the air-gapped environment.

**Rationale**:
- FLOSS (GPLv3), no licensing dependency
- Provides LDAP directory, Kerberos authentication, PKI (Dogtag CA), DNS, and sudo/HBAC policies
- Well-suited for air-gapped deployment
- Avoids dependency on Microsoft Active Directory (foreign vendor, proprietary)
- If the organization already runs AD for non-classified environments, FreeIPA can be deployed standalone for the classified environment (no federation across classification levels)

**FreeIPA deployment**:
- 2x FreeIPA replicas (HA) running on infrastructure nodes (as Kubernetes pods or dedicated VMs via KubeVirt)
- Internal Certificate Authority (Dogtag) for issuing TLS certificates, user certificates, and service certificates
- cert-manager Kubernetes integration for automated certificate issuance to pods/services

### 10.4 Authentication and Authorization

| Layer | Mechanism |
|---|---|
| **Node SSH** (if enabled) | SSH key + FreeIPA LDAP, session recorded |
| **Kubernetes API** | OIDC via Keycloak (backed by FreeIPA), or FreeIPA client certificates |
| **Kubernetes RBAC** | Role/ClusterRole bindings mapped to FreeIPA groups |
| **Application access** | Keycloak (OIDC/SAML) backed by FreeIPA LDAP |
| **Harbor registry** | OIDC via Keycloak |
| **Grafana/Monitoring** | OIDC via Keycloak |
| **ArgoCD** | OIDC via Keycloak with RBAC mapped to FreeIPA groups |

**Keycloak** (FLOSS, CNCF) is deployed as the OIDC/SAML identity broker, with FreeIPA as the backend identity store. This provides a modern authentication layer for all platform and application services.

### 10.5 Multi-Factor Authentication

MFA is mandatory for all access to the HEMLIG platform:

- **Hardware tokens**: YubiKey 5 series (FIPS) or equivalent, providing FIDO2/WebAuthn and PIV
- **Smart cards**: if the organization uses Swedish defense smart cards (likely), integrate via PIV/PKCS#11
- Software tokens (TOTP) may be acceptable as a secondary factor but hardware tokens are preferred at HEMLIG level

### 10.6 Privileged Access Management

- **No standing privileged access**: admin access is requested, approved, and time-limited
- **Break-glass procedure**: documented emergency access procedure with immediate audit notification
- **Session recording**: all administrative sessions recorded (Teleport or similar, deployed internally)
- **Four-eyes principle**: destructive operations (node decommission, data deletion, cluster upgrades) require approval from two cleared personnel
- **Separate admin accounts**: administrators have both a regular user account and a separate admin account; admin accounts are only used for administrative tasks

---

## 11. Air-Gapped Operations

### 11.1 Principles

The air gap is **physical, not logical**. There is no network path -- no firewall rule, no VPN, no proxy -- between this environment and any external network. The only data paths are:

1. **Data diode** (hardware-enforced one-way inbound) for operational data ingest
2. **Sneakernet** (physical media transfer) for software updates, container images, and configuration
3. **Controlled export** (manual review + physical media) for sanitized data egress

### 11.2 Software Update Pipeline

Software updates follow a rigorous pipeline from the unclassified development environment to the classified production environment:

```
┌─────────────────────────────────────────────────────────────────────┐
│  UNCLASSIFIED ENVIRONMENT (outside air gap)                         │
│                                                                     │
│  1. Engineers (28 uncleared + 12 cleared) develop and test          │
│     applications in unclassified dev/staging environments           │
│  2. Container images built, scanned (Trivy/Grype), signed (Cosign) │
│  3. SBOM generated (Syft) for every image                           │
│  4. Helm charts packaged and signed                                 │
│  5. OS packages mirrored and verified (GPG signatures)              │
│  6. All artifacts staged to transfer media                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  TRANSFER PREPARATION STATION (cleared personnel only)  │        │
│  │  - Verify all signatures and checksums                   │        │
│  │  - Malware scan with multiple engines                    │        │
│  │  - Audit manifest: list all artifacts being transferred  │        │
│  │  - Approve transfer (two-person authorization)           │        │
│  │  - Write to new, clean transfer media (encrypted USB/    │        │
│  │    encrypted portable drive)                             │        │
│  │  - Log the transfer in the chain-of-custody system       │        │
│  └─────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────┐
                    │  PHYSICAL TRANSFER    │
                    │  (Sneakernet)         │
                    │  Encrypted media      │
                    │  Chain of custody     │
                    │  Two-person control   │
                    └──────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  CLASSIFIED ENVIRONMENT (inside air gap)                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │  IMPORT STATION (air-gapped, classified side)           │        │
│  │  - Verify media integrity and chain-of-custody          │        │
│  │  - Re-verify all signatures and checksums               │        │
│  │  - Additional malware scan with classified-side tools   │        │
│  │  - Import artifacts into internal Harbor registry       │        │
│  │  - Import OS packages into internal repo mirrors        │        │
│  │  - Import Helm charts into internal chart museum        │        │
│  │  - Log the import in the classified audit system        │        │
│  │  - Destroy or securely wipe the transfer media          │        │
│  └─────────────────────────────────────────────────────────┘        │
│                                                                     │
│  ArgoCD detects new images/charts in Harbor/chart repo              │
│  → Deploys updates per GitOps workflow                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.3 Offline Artifact Management

| Artifact Type | Offline Tool | Internal Store |
|---|---|---|
| Container images | `skopeo copy`, Hauler bundles | Harbor registry |
| Helm charts | `helm package`, Hauler | Harbor (OCI) or ChartMuseum |
| OS packages (RPM/DEB) | Pulp, Aptly, or local createrepo | Internal repo server |
| Ansible collections | `ansible-galaxy collection download` | Internal Galaxy mirror (Pulp) |
| Python packages | `pip download` | Internal PyPI (Pulp or devpi) |
| Git repositories | `git bundle` | Internal Gitea or GitLab |
| RKE2 binaries | Pre-packaged air-gap bundles from RKE2 releases | Internal file server |

### 11.4 Internal Time Source

Air-gapped environments cannot use external NTP. Options:

- **GPS-disciplined NTP server**: GPS antenna (receive-only, no transmit capability -- acceptable for air-gapped environments per most accreditation authorities) with a GPS-disciplined oscillator feeding an internal NTP server
- **Rubidium oscillator**: free-running atomic clock for environments where even GPS reception is not permitted
- **Chrony** configured on all nodes pointing to the internal NTP server(s)

**Recommendation**: GPS-disciplined NTP (2x for redundancy) pending MUST approval. GPS is receive-only and does not create an information path out of the classified environment.

---

## 12. Data Flow and Cross-Domain Architecture

### 12.1 Data Ingest (Into Classified Environment)

For operational data that needs to flow into the classified environment continuously (e.g., sensor data, intelligence feeds from other systems):

**Hardware Data Diode**:

- **Product**: Advenica SecuriCDS (Swedish manufacturer, specifically designed for Swedish defense), or equivalent FRA/MUST-approved data diode
- **Function**: Hardware-enforced one-way data flow -- physically impossible for data to traverse in the reverse direction
- **Deployment**: The data diode connects a lower-classification "sending" network to the HEMLIG "receiving" network on VLAN 40
- **Receiving pipeline**: A dedicated ingest service on the classified side receives data from the diode, validates it, and stores it in the appropriate application data store

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────────┐
│  Lower-class  │────▶│  DATA DIODE  │────▶│  Classified ingest       │
│  sending      │     │  (Advenica   │     │  pipeline                │
│  system       │     │   one-way)   │     │  (VLAN 40, K8s pods)     │
│               │     │              │     │                          │
│  Cannot       │     │  Physically  │     │  Validates, stores,      │
│  receive      │     │  enforced    │     │  logs all received data  │
│  data back    │     │  one-way     │     │                          │
└──────────────┘     └──────────────┘     └──────────────────────────┘
```

**Advenica rationale**: Advenica is a Swedish company (headquartered in Malmö) specializing in cross-domain solutions for Swedish and European defense. Their products are designed for Swedish classification levels and are evaluated by FMV/FRA. Using a Swedish-manufactured data diode aligns with supply chain security requirements and simplifies the accreditation process.

### 12.2 Data Export (Out of Classified Environment)

Data export from HEMLIG requires:

1. **Formal request** with justification and recipient identification
2. **Sanitization review** (sekretessprövning/nedsekretisering) -- human review to verify the data can be released to a lower classification level
3. **Approval** by designated authority (typically the information owner + security officer)
4. **Transfer** via physical media with chain-of-custody documentation
5. **Audit logging** of the entire export process

There is **no automated export path**. All data leaving the HEMLIG environment goes through manual review and physical media transfer.

### 12.3 Cross-Domain Solution (If Required)

If bi-directional controlled data exchange is needed (e.g., with NATO systems or other Swedish defense systems at different classification levels):

- **Advenica SecuriCDS Cross Domain Solution** provides content-inspecting, policy-driven bi-directional transfer
- Requires separate accreditation for the CDS itself
- Content inspection policies must be approved by FMV/MUST
- Every transfer is logged and auditable
- This is a significant accreditation effort -- only implement if there is a clear operational requirement

---

## 13. Security Architecture

### 13.1 Defense in Depth

Security is implemented in layers. No single control is relied upon alone.

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: PHYSICAL SECURITY                                  │
│  Säkerhetsskyddat utrymme, access control, TEMPEST           │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Layer 2: NETWORK SECURITY                              │ │
│  │  Air gap, spine-leaf segmentation, Cilium policies      │ │
│  │  ┌─────────────────────────────────────────────────────┐│ │
│  │  │  Layer 3: HOST SECURITY                             ││ │
│  │  │  SELinux, LUKS, hardened OS, auditd                 ││ │
│  │  │  ┌─────────────────────────────────────────────────┐││ │
│  │  │  │  Layer 4: KUBERNETES SECURITY                   │││ │
│  │  │  │  RBAC, PSA, NetworkPolicy, admission control    │││ │
│  │  │  │  ┌─────────────────────────────────────────────┐│││ │
│  │  │  │  │  Layer 5: APPLICATION SECURITY              ││││ │
│  │  │  │  │  mTLS, image signing, SBOM, runtime sec     ││││ │
│  │  │  │  │  ┌─────────────────────────────────────────┐││││ │
│  │  │  │  │  │  Layer 6: DATA SECURITY                 │││││ │
│  │  │  │  │  │  Encryption at rest, etcd encryption,   │││││ │
│  │  │  │  │  │  backup encryption, key management      │││││ │
│  │  │  │  │  └─────────────────────────────────────────┘││││ │
│  │  │  │  └─────────────────────────────────────────────┘│││ │
│  │  │  └─────────────────────────────────────────────────┘││ │
│  │  └─────────────────────────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 13.2 Encryption

| Data State | Mechanism | Key Management |
|---|---|---|
| **At rest (OS disks)** | LUKS2 with AES-256-XTS | Keys sealed to TPM 2.0 |
| **At rest (Ceph OSD)** | LUKS2 via Rook dm-crypt | Keys managed by Rook, sealed to TPM |
| **At rest (etcd)** | RKE2 secrets encryption (AES-CBC/AES-GCM) | Encryption config in RKE2 |
| **In transit (pod-to-pod)** | WireGuard via Cilium | Cilium-managed key rotation |
| **In transit (API server)** | TLS 1.3 | Internal PKI (FreeIPA/Dogtag CA) |
| **In transit (Ceph)** | Ceph messenger v2 encryption | Ceph-managed |
| **Backup data** | LUKS2 on backup media + Restic encryption | Separate backup keys |

**Note**: All cryptographic implementations must be reviewed and approved by FRA/MUST during accreditation. The selections above represent strong defaults that may need to be replaced with FRA-approved equivalents.

### 13.3 Supply Chain Security

| Control | Implementation |
|---|---|
| **Image provenance** | All images signed with Cosign; Kyverno blocks unsigned images |
| **SBOM** | Syft generates SBOM for every image; stored alongside image in Harbor |
| **Vulnerability scanning** | Trivy scans all images in Harbor; Kyverno blocks images with critical CVEs |
| **Base image control** | Approved base images maintained in Harbor; all application images must derive from approved bases |
| **Package verification** | All OS packages GPG-verified against known signing keys |
| **Binary verification** | RKE2, Cilium, Rook, and all platform binaries verified against published checksums and signatures |
| **Dependency tracking** | Full dependency tree documented for all platform components |
| **SBOM for platform** | SBOM generated for the platform itself (all components and versions) |

### 13.4 Runtime Security

- **Falco**: monitors system calls and detects anomalous behavior (unexpected process execution, file access to sensitive paths, network connections to unusual destinations)
- **Tetragon** (Cilium): eBPF-based runtime enforcement -- can block (not just alert on) unauthorized operations
- **Audit logging**: Kubernetes audit log captures all API server interactions
- **Node audit**: auditd on every node captures system-level events
- **File integrity monitoring**: AIDE or similar to detect unauthorized changes to critical files

### 13.5 Vulnerability Management

In an air-gapped environment, vulnerability management requires a deliberate process:

1. Security engineers monitor vulnerability feeds in the unclassified environment (CVE databases, vendor advisories, CERT-SE bulletins)
2. Relevant vulnerabilities are assessed for impact on the classified platform
3. Patches are tested in the unclassified staging environment
4. Patched images/packages are transferred via the sneakernet process (Section 11.2)
5. Patches are deployed via GitOps (ArgoCD) or Ansible playbooks
6. Vulnerability scanning (Trivy) confirms remediation

**Patch SLA**: Critical vulnerabilities affecting the platform must be remediated within 72 hours of identification. This requires maintaining a ready-to-transfer set of base OS and platform component updates.

---

## 14. Observability and Audit

### 14.1 Monitoring Stack

All monitoring is deployed on the infrastructure nodes and runs entirely within the air-gapped environment.

| Component | Tool | Purpose |
|---|---|---|
| **Metrics** | Prometheus (with Thanos for long-term retention) | Cluster, node, pod, and application metrics |
| **Dashboards** | Grafana | Visualization and alerting dashboards |
| **Logs** | Loki + Promtail | Centralized log aggregation |
| **Traces** | Tempo | Distributed tracing for application debugging |
| **Network flows** | Hubble (Cilium) | Network traffic visibility and policy audit |
| **Alerting** | Alertmanager | Alert routing (email to internal mail, webhook to internal chat) |
| **Uptime** | Blackbox Exporter | Synthetic probes for service availability |
| **Hardware** | IPMI Exporter, node_exporter | Hardware health, temperatures, disk health |
| **Ceph** | Ceph MGR Prometheus module | Storage cluster health and performance |

### 14.2 Audit Logging

Audit logging is mandatory and must be comprehensive. Every action must be attributable to an individual.

| Audit Source | Destination | Retention |
|---|---|---|
| Kubernetes API audit log | Loki (via Promtail) | 5 years minimum (per FMV/MUST guidance) |
| Node-level auditd | Loki (via Promtail) | 5 years minimum |
| Falco alerts | Loki + Alertmanager | 5 years minimum |
| Network flow logs (Hubble) | Loki | 2 years minimum |
| Authentication events (Keycloak/FreeIPA) | Loki | 5 years minimum |
| Data transfer logs (import/export) | Loki + dedicated audit database | 10 years (or per organizational policy) |
| Physical access logs | Integrated with facility security system | Per facility security policy |
| Harbor image events | Loki | 5 years minimum |

**Audit log integrity**: Audit logs must be tamper-evident. Options:

- Write-once storage (Ceph RGW with object lock)
- Log signing with internal PKI
- Regular audit log backups to separate storage with integrity verification

### 14.3 Alerting Strategy

| Severity | Response Time | Examples |
|---|---|---|
| **Critical** | 15 minutes | Node failure, Ceph OSD down (degraded redundancy), security alert (Falco), control plane component failure |
| **Warning** | 4 hours | Disk space >80%, certificate expiring in <30 days, Ceph near-full, high error rate |
| **Info** | Next business day | Completed backups, successful deployments, routine health checks |

Alert delivery in an air-gapped environment:

- Internal email server (Postfix or similar, internal only)
- Internal chat system (Mattermost, self-hosted, classified-side only)
- Physical alert terminals / dashboard displays in the operations room
- On-call pager system (internal, or integration with commercial pager service via a one-way alert diode if approved)

---

## 15. Automation and Infrastructure as Code

### 15.1 GitOps with ArgoCD

ArgoCD is the primary deployment mechanism for all workloads and platform configuration.

**Architecture**:

- ArgoCD deployed on infrastructure nodes
- Internal Gitea instance (FLOSS Git hosting) holds all manifests and Helm charts
- ArgoCD watches Gitea repositories and reconciles cluster state
- All changes go through Git -- no `kubectl apply` in production
- ArgoCD RBAC mapped to FreeIPA groups via Keycloak OIDC
- Application Sets for managing multiple workload namespaces

**Workflow**:

```
Developer (on classified workstation)
    │
    ▼
Git commit to Gitea (classified-side)
    │
    ▼
ArgoCD detects change
    │
    ▼
ArgoCD syncs to cluster
    │
    ▼
Kyverno validates admission
    │
    ▼
Workload deployed
    │
    ▼
Monitoring confirms health
```

### 15.2 Ansible for Node Lifecycle

Ansible is used for operations that are outside Kubernetes scope:

- Node provisioning and OS configuration
- RKE2 installation and upgrades
- OS patching
- BIOS/firmware updates
- Certificate rotation
- Compliance scanning and remediation

**Ansible deployment**:

- AWX (FLOSS upstream of Ansible Automation Platform) for Ansible execution, inventory, and credential management
- AWX deployed on infrastructure nodes as Kubernetes pods
- Playbooks stored in Gitea
- Ansible Vault for secrets encryption in playbooks
- All Ansible execution logged and auditable in AWX

### 15.3 Infrastructure as Code

| Component | Tool | Repository |
|---|---|---|
| Kubernetes manifests | Helm charts + Kustomize overlays | Gitea |
| ArgoCD Application definitions | YAML manifests | Gitea |
| Ansible playbooks | Ansible (roles + playbooks) | Gitea |
| Network switch configuration | Ansible (cisco.nxos or equivalent) | Gitea |
| FreeIPA configuration | Ansible (freeipa.ansible_freeipa) | Gitea |
| Ceph configuration | Rook CRDs (managed via ArgoCD) | Gitea |
| Kyverno policies | YAML manifests (managed via ArgoCD) | Gitea |
| Monitoring configuration | Grafana dashboards as code, Prometheus rules | Gitea |

**Everything is in Git. Nothing is configured manually in production.**

---

## 16. Disaster Recovery and Business Continuity

### 16.1 RTO/RPO Targets

| Scenario | RTO | RPO |
|---|---|---|
| Single node failure | 5 minutes (automatic) | 0 (replicated) |
| Multiple node failure (up to N-1 in any role) | 30 minutes | 0 (replicated) |
| Complete control plane loss | 2 hours | 6 hours (etcd snapshot interval) |
| Complete storage loss | 4 hours | 24 hours (last backup) |
| Complete site loss (catastrophic) | Depends on secondary site availability | 24 hours |

### 16.2 Resilience Design

- **Control plane**: 3 nodes, etcd quorum tolerates 1 node failure
- **Worker nodes**: 8 nodes, workloads replicated with anti-affinity rules, tolerate 2+ node failures
- **Storage**: Ceph 3x replication tolerates loss of any 2 OSDs simultaneously; EC 4+2 tolerates loss of any 2 OSDs
- **Network**: Dual-homed servers (2x 25 GbE per role), dual spine switches, no single point of failure in the fabric
- **Infrastructure services**: All platform services (monitoring, logging, registry, ArgoCD) deployed with HA (multiple replicas with anti-affinity)

### 16.3 Backup Strategy

- **etcd**: Automated snapshots every 6 hours, retained for 28 days, stored on Ceph
- **Kubernetes resources**: Velero backup daily, retained for 90 days
- **Persistent volumes**: Ceph RBD snapshots daily, retained for 30 days
- **Application data**: Application-specific backup policies (database dumps, etc.)
- **Configuration**: All in Git (Gitea), Gitea backed up daily
- **Audit logs**: Backed up to write-once storage, retained per policy (5-10 years)

### 16.4 Secondary Site Considerations

For true disaster recovery, a secondary secure facility within Sweden is recommended:

- Ceph RBD mirroring (asynchronous) to secondary site over dedicated dark fiber
- The secondary site must meet the same physical security and accreditation requirements
- This is a significant cost and accreditation effort -- prioritize based on risk assessment in the Säkerhetsskyddsanalys
- If a secondary site is not immediately feasible, ensure offline backups (encrypted media) are stored in a separate approved secure facility

---

## 17. Staffing and Operational Model

### 17.1 The 12-Person Cleared Team

The constraint of 12 cleared personnel drives the architecture toward maximum automation and operational simplicity.

**Organizational structure**:

```
                    ┌─────────────────────────┐
                    │  Platform Lead /         │
                    │  Security Officer (1)    │
                    └────────────┬────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
    ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
    │  Platform    │      │  Security    │      │  Application │
    │  Engineering │      │  Engineering │      │  & Data Ops  │
    │  (4 people)  │      │  (2 people)  │      │  (5 people)  │
    │              │      │              │      │              │
    │  K8s ops     │      │  Monitoring  │      │  App onboard │
    │  Storage     │      │  Audit       │      │  CI/CD       │
    │  Networking  │      │  Compliance  │      │  Data ingest │
    │  Node mgmt   │      │  Vuln mgmt   │      │  Developer   │
    │              │      │              │      │  support     │
    └─────────────┘      └─────────────┘      └─────────────┘
```

### 17.2 Collaboration with Uncleared Staff

The 28 engineers without HEMLIG clearance contribute to:

- Application development in the unclassified environment
- Testing and staging environment management
- Container image building and initial scanning
- Documentation and runbook development
- Architecture design and review (at the unclassified level)
- Tooling and automation development (Ansible roles, Helm charts, etc.)

**The boundary is clear**: uncleared staff never access the classified environment, its data, or its operational interfaces. They produce artifacts (code, images, configurations) that are transferred into the classified environment via the controlled import process.

### 17.3 Training

All 12 cleared personnel must receive:

- RKE2 / Kubernetes administration (CKA-level competence)
- Cilium networking and troubleshooting
- Rook-Ceph storage administration
- Security operations (Falco, Kyverno, vulnerability management)
- Incident response procedures
- Data handling procedures for HEMLIG
- Physical security awareness per Säkerhetsskyddslagen

### 17.4 Vendor Support

Vendor support in a HEMLIG environment is restricted:

- Vendors cannot access the classified environment remotely
- On-site vendor support requires escorts by cleared personnel at all times
- Vendor personnel may need to be cleared depending on what they can see/access
- Prefer vendors with Swedish presence and Swedish-cleared engineers
- FLOSS components reduce vendor dependency -- the team must be capable of self-supporting all platform components

---

## 18. Accreditation Roadmap

### 18.1 Phased Approach

| Phase | Duration (Est.) | Activities |
|---|---|---|
| **Phase 0: Säkerhetsskyddsanalys** | 2-3 months | Complete the formal protective security analysis. Identify threats, vulnerabilities, and required controls. This phase MUST be completed before finalizing the architecture. Engage FMV early. |
| **Phase 1: Architecture Finalization** | 1-2 months | Finalize architecture based on Säkerhetsskyddsanalys findings. Complete system security plan. Procure hardware via säkerhetsskyddad upphandling. |
| **Phase 2: Facility Preparation** | 2-4 months | Physical security upgrades to the data center. TEMPEST assessment and any required shielding. Hardware installation and verification. |
| **Phase 3: Platform Build** | 2-3 months | Deploy RKE2, Cilium, Rook-Ceph, Harbor, ArgoCD, monitoring stack, FreeIPA, Keycloak. Implement all security controls. |
| **Phase 4: Hardening and Testing** | 2-3 months | CIS hardening verification. Penetration testing (by cleared testers). Compliance scanning. Vulnerability assessment. Security documentation completion. |
| **Phase 5: FMV/MUST Review** | 3-6 months | FMV conducts Säkerhetsgranskning (security review). MUST reviews TEMPEST compliance and crypto. Address findings. |
| **Phase 6: Accreditation** | 1-2 months | FMV grants accreditation (Ackreditering) to process HEMLIG data. |
| **Phase 7: Operational Readiness** | 1-2 months | Onboard initial workloads. Validate operational procedures. Train all staff. |

**Total estimated timeline: 14-25 months** from start to operational readiness.

### 18.2 Continuous Accreditation

After initial accreditation:

- Automated compliance scanning runs daily (OpenSCAP or equivalent)
- Compliance dashboard in Grafana shows real-time posture
- Quarterly internal security reviews
- Annual comprehensive security assessment
- Significant changes (hardware replacement, major software upgrade, architecture modification) trigger a change assessment with FMV to determine if reaccreditation is needed

---

## 19. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Accreditation delay due to FMV findings | Medium | High | Engage FMV early (Phase 0). Iterate on architecture with FMV feedback before building. |
| R2 | TEMPEST requirements more stringent than anticipated | Medium | High | Budget for shielded room. Consult MUST during Phase 0. |
| R3 | 12-person team insufficient for operations + development | Medium | Medium | Maximize automation (GitOps, auto-remediation). Prioritize training. Design for operational simplicity. |
| R4 | Key personnel departure (bus factor) | Medium | High | Cross-train all roles. Document all procedures. Ensure at least 2 people can perform any critical function. |
| R5 | Air-gapped update process too slow for critical patches | Medium | Medium | Maintain pre-staged patch bundles. Practice the transfer process regularly. Target 72-hour patch SLA. |
| R6 | Hardware supply chain issues | Medium | Medium | Procure spares (N+1 for each node role). Use commodity hardware where possible. |
| R7 | FRA does not approve selected crypto implementations | Low | High | Design crypto layer to be swappable. Engage FRA early for guidance on approved products. |
| R8 | Ceph storage performance insufficient for workloads | Low | Medium | Benchmark during Phase 3. All-NVMe design provides headroom. Tune Ceph placement groups and recovery settings. |
| R9 | KubeVirt insufficient for legacy VM workloads | Low | Medium | Evaluate specific VM workload requirements in Phase 1. Fallback: dedicated Proxmox cluster for VMs (increases operational burden). |
| R10 | International interoperability requirements (NATO) emerge | Medium | Medium | Architecture uses standard protocols and APIs. Design permits future federation. Document NATO STANAG alignment. |

---

## 20. Architectural Decision Records

### ADR-001: Platform Selection -- RKE2 on Bare Metal

**Status**: Proposed
**Context**: Need a Kubernetes distribution for air-gapped classified workloads operated by a 12-person team.
**Decision**: RKE2 on bare-metal servers.
**Rationale**: Security-hardened by default, first-class air-gapped support, FLOSS (Apache 2.0), single-binary simplicity, CIS compliance out of the box. See Section 4.1 for full analysis.
**Consequences**: Team must develop RKE2 expertise. No commercial support from SUSE/Rancher in the classified environment (self-support model).

### ADR-002: CNI Selection -- Cilium

**Status**: Proposed
**Context**: Need a container networking solution with strong security, encryption, and observability.
**Decision**: Cilium with WireGuard encryption and Hubble observability.
**Rationale**: eBPF-based for performance and security, WireGuard for transparent encryption, Hubble for network flow visibility (audit requirement), replaces kube-proxy (reduced attack surface), strong L7 policy capabilities.
**Consequences**: eBPF requires modern kernel (>= 5.10). Team must develop Cilium expertise.

### ADR-003: Storage -- Rook-Ceph

**Status**: Proposed
**Context**: Need software-defined storage providing block, file, and object storage.
**Decision**: Rook-Ceph deployed within Kubernetes.
**Rationale**: Unified storage platform (block + file + object), FLOSS, no licensing cost, Rook operator simplifies lifecycle management, encryption at rest via dm-crypt, proven at scale.
**Consequences**: Ceph operations require specialized knowledge. Workers serve dual purpose (compute + storage), which introduces resource contention risk.

### ADR-004: Identity Management -- FreeIPA + Keycloak

**Status**: Proposed
**Context**: Need identity management, PKI, and OIDC/SAML for the air-gapped environment.
**Decision**: FreeIPA for identity directory and PKI, Keycloak for OIDC/SAML.
**Rationale**: Both FLOSS, no foreign commercial vendor dependency, FreeIPA provides integrated LDAP + Kerberos + PKI, Keycloak provides modern authentication protocols for Kubernetes and applications.
**Consequences**: Two systems to manage (FreeIPA + Keycloak). Team must develop FreeIPA expertise.

### ADR-005: Data Diode -- Advenica SecuriCDS

**Status**: Proposed
**Context**: Need hardware-enforced one-way data ingest into the classified environment.
**Decision**: Advenica SecuriCDS data diode.
**Rationale**: Swedish manufacturer, designed for Swedish defense classification levels, evaluated by FMV/FRA, simplifies accreditation.
**Consequences**: Vendor dependency on Advenica. Data export remains manual (sneakernet).

### ADR-006: Single Cluster Architecture

**Status**: Proposed
**Context**: Single vs. multi-cluster for a 12-person team.
**Decision**: Single cluster with namespace-based tenancy.
**Rationale**: Operational simplicity for small team. Strong namespace isolation via NetworkPolicy, RBAC, PSA, and Kyverno is sufficient for same-classification-level multi-tenancy. Multi-cluster adds operational overhead not justified by current requirements.
**Consequences**: Blast radius of cluster-level failure affects all workloads. If compartmentalization requirements emerge, must migrate to multi-cluster.

### ADR-007: GitOps -- ArgoCD

**Status**: Proposed
**Context**: Need declarative, auditable deployment mechanism.
**Decision**: ArgoCD for Kubernetes GitOps, with Gitea as the internal Git server.
**Rationale**: CNCF graduated project, strong community, RBAC integration, audit trail for all deployments, drift detection and reconciliation. Gitea is lightweight FLOSS Git hosting suitable for the air-gapped environment.
**Consequences**: All deployment changes must go through Git. Requires discipline in the team to avoid ad-hoc `kubectl` changes.

### ADR-008: Operating System -- SLE Micro (Primary) / Rocky Linux (Alternative)

**Status**: Proposed
**Context**: Need a hardened, minimal OS for Kubernetes nodes.
**Decision**: SLE Micro as primary; Rocky Linux 9 as fallback.
**Rationale**: SLE Micro is immutable, transactional-update based, minimal attack surface, European vendor (SUSE). Rocky Linux is a viable fallback with strong CIS hardening guides and SELinux support.
**Consequences**: SLE Micro requires SUSE subscription (cost, but acceptable for defense). Rocky Linux requires more manual hardening effort.

---

## Appendix A: Component Summary

| Category | Component | License | Version (Pin at deployment) |
|---|---|---|---|
| Kubernetes | RKE2 | Apache 2.0 | Latest stable at deployment |
| CNI | Cilium | Apache 2.0 | Latest stable |
| Storage | Rook-Ceph | Apache 2.0 / LGPL | Latest stable |
| Registry | Harbor | Apache 2.0 | Latest stable |
| GitOps | ArgoCD | Apache 2.0 | Latest stable |
| Git hosting | Gitea | MIT | Latest stable |
| Policy engine | Kyverno | Apache 2.0 | Latest stable |
| Runtime security | Falco | Apache 2.0 | Latest stable |
| Runtime enforcement | Tetragon | Apache 2.0 | Latest stable |
| Monitoring | Prometheus + Thanos | Apache 2.0 | Latest stable |
| Dashboards | Grafana | AGPL 3.0 | Latest stable |
| Logging | Loki + Promtail | AGPL 3.0 | Latest stable |
| Tracing | Tempo | AGPL 3.0 | Latest stable |
| Alerting | Alertmanager | Apache 2.0 | Latest stable |
| Identity | FreeIPA | GPLv3 | Latest stable |
| Auth broker | Keycloak | Apache 2.0 | Latest stable |
| Cert management | cert-manager | Apache 2.0 | Latest stable |
| Backup | Velero | Apache 2.0 | Latest stable |
| Automation | Ansible + AWX | GPLv3 | Latest stable |
| Data diode | Advenica SecuriCDS | Commercial (Swedish) | Per FMV/FRA recommendation |
| Ingress | NGINX Ingress Controller | Apache 2.0 | Latest stable |
| Load balancer | MetalLB | Apache 2.0 | Latest stable |
| Control plane HA | kube-vip | Apache 2.0 | Latest stable |
| VM workloads | KubeVirt | Apache 2.0 | Latest stable |
| Image signing | Cosign (Sigstore) | Apache 2.0 | Latest stable |
| SBOM | Syft | Apache 2.0 | Latest stable |
| Vuln scanning | Trivy | Apache 2.0 | Latest stable |
| Network observability | Hubble (Cilium) | Apache 2.0 | Latest stable |

## Appendix B: Network Port Matrix

| Source | Destination | Port | Protocol | Purpose |
|---|---|---|---|---|
| Admin workstation | Control plane | 6443 | TCP/TLS | Kubernetes API |
| Control plane | Control plane | 2379-2380 | TCP/TLS | etcd peer/client |
| All nodes | All nodes | 8472 | UDP | Cilium VXLAN |
| All nodes | All nodes | 4240 | TCP | Cilium health |
| All nodes | All nodes | 51871 | UDP | WireGuard (Cilium encryption) |
| Worker nodes | Worker nodes | 6800-7300 | TCP | Ceph OSD |
| All nodes | Infra nodes | 6789 | TCP | Ceph MON |
| All nodes | Infra nodes | 9090 | TCP | Prometheus |
| All nodes | Infra nodes | 3100 | TCP | Loki |
| All nodes | Infra nodes | 443 | TCP/TLS | Harbor registry |
| All nodes | Infra nodes | 389/636 | TCP | FreeIPA LDAP/LDAPS |
| All nodes | Infra nodes | 88/464 | TCP/UDP | Kerberos |
| Infra nodes | All nodes | 9100 | TCP | node_exporter |
| Infra nodes | All nodes | 10250 | TCP/TLS | kubelet |

## Appendix C: Glossary of Swedish Terms

| Swedish | English | Context |
|---|---|---|
| Säkerhetsskyddslagen | Protective Security Act | Primary legislation governing classified information |
| Säkerhetsskyddsförordningen | Protective Security Ordinance | Implementing regulation |
| Säkerhetsskyddsanalys | Protective Security Analysis | Mandatory risk assessment |
| Säkerhetsskyddat utrymme | Security-protected area | Physical security zone |
| Säkerhetsprövning | Security vetting/clearance | Personnel clearance process |
| Tystnadsplikt | Non-disclosure obligation | Legal secrecy commitment |
| Säkerhetsskyddad upphandling | Security-protected procurement | Procurement rules for classified systems |
| Säkerhetsgranskning | Security review | FMV technical security assessment |
| Ackreditering | Accreditation | Formal approval to process classified data |
| HEMLIG | Secret | Classification level |
| BEGRÄNSAT HEMLIG | Restricted | Lower classification level |
| KVALIFICERAT HEMLIG | Top Secret | Higher classification level |
| FMV | Försvarets materielverk | Swedish Defence Materiel Administration |
| MUST | Militära underrättelse- och säkerhetstjänsten | Military Intelligence and Security Service |
| FRA | Försvarets radioanstalt | National Defence Radio Establishment |
| Säpo | Säkerhetspolisen | Swedish Security Service |
| MSB | Myndigheten för samhällsskydd och beredskap | Civil Contingencies Agency |

---

*This architecture document is designed to support the Säkerhetsskyddsanalys and FMV accreditation process. All architectural decisions are subject to revision based on findings from the Säkerhetsskyddsanalys and feedback from FMV/MUST during the accreditation process. The document should be treated as HEMLIG-adjacent and handled according to organizational information security policies.*
