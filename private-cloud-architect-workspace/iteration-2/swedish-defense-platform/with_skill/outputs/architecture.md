# Private Cloud Architecture for HEMLIG Classified Data Processing

## Swedish Defense Contractor -- Kubernetes-Based Air-Gapped Platform

**Classification**: HEMLIG (Secret)
**Regulatory Framework**: Sakerhetssk yddslagen (2018:585), FMV/MUST Accreditation
**Document Status**: Architecture Design Document
**Date**: 2026-03-20

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Requirements Analysis](#2-requirements-analysis)
3. [Accreditation Strategy](#3-accreditation-strategy)
4. [Physical Security Architecture](#4-physical-security-architecture)
5. [Network Architecture](#5-network-architecture)
6. [Compute Architecture](#6-compute-architecture)
7. [Platform Architecture -- Kubernetes](#7-platform-architecture--kubernetes)
8. [Storage Architecture](#8-storage-architecture)
9. [Identity and Access Management](#9-identity-and-access-management)
10. [Air-Gapped Operations](#10-air-gapped-operations)
11. [Security Architecture](#11-security-architecture)
12. [Observability and Audit](#12-observability-and-audit)
13. [Disaster Recovery and Business Continuity](#13-disaster-recovery-and-business-continuity)
14. [Operational Model and Staffing](#14-operational-model-and-staffing)
15. [Supply Chain Security](#15-supply-chain-security)
16. [Infrastructure as Code and Automation](#16-infrastructure-as-code-and-automation)
17. [Migration and Onboarding](#17-migration-and-onboarding)
18. [Cost and Capacity Planning](#18-cost-and-capacity-planning)
19. [Risk Register](#19-risk-register)
20. [Architectural Decision Records](#20-architectural-decision-records)

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform designed to process HEMLIG (Secret) classified data for a Swedish defense contractor. The platform is fully air-gapped, physically located within Sweden, and designed from the ground up to meet accreditation requirements under Sakerhetsskyddslagen (2018:585) as assessed by FMV (Forsvarets materielverk) and MUST (Militara underrattelse- och sakerhetstjansten).

The platform is built on Kubernetes (RKE2 Government) running on hardened bare-metal servers, with Ceph for software-defined storage, Cilium for network policy enforcement, and a comprehensive GitOps-driven operational model. All 12 cleared engineers operate under strict need-to-know and personnel security controls vetted by Sapo.

**Key design principles**:

- Classification drives architecture -- every decision flows from the HEMLIG classification requirement
- Air gap is physical, not logical -- no network path exists between this environment and any other network
- Data sovereignty is absolute -- all data, processing, backups, and keys remain within Sweden
- Accreditation is continuous -- automated compliance scanning from day one
- Least privilege and need-to-know -- enforced at every layer from physical access to namespace-level RBAC
- Audit everything -- comprehensive, tamper-evident audit trails with long retention

---

## 2. Requirements Analysis

### 2.1 Regulatory Requirements

| Requirement | Source | Impact |
|---|---|---|
| Data must be processed and stored within Sweden | Sakerhetsskyddslagen 2018:585 | All infrastructure physically in Sweden |
| Personnel must hold Swedish security clearances | Sakerhetsskyddslagen Ch. 3 | Only 12 of 40 engineers can operate the platform |
| Physical security must meet HEMLIG standards | FMV/MUST directives | Controlled access area, intrusion detection, guards |
| TEMPEST/emanation security | MUST TEMPEST directives (aligned with NATO SDIP-27) | Shielded facility, red/black separation |
| Cryptographic modules must be Swedish/NATO approved | FRA (Forsvarets radioanstalt) guidance | National crypto for data at rest and in transit |
| Continuous monitoring and reaccreditation | Sakerhetsskyddslagen 2018:585, updated 2021 | Automated compliance, periodic reassessment |
| Supply chain integrity | FMV procurement requirements | Trusted sourcing, tamper-evident delivery, firmware verification |
| Audit trail retention | Sakerhetsskyddslagen + FMV directives | Minimum 5 years, tamper-evident, attributable to individuals |

### 2.2 Technical Requirements

| Requirement | Specification |
|---|---|
| Classification level | HEMLIG (Secret) |
| Air gap | Full physical air gap -- no connectivity to any external network |
| Data residency | All data within Swedish borders, single facility initially |
| Availability target | 99.95% (allows ~4.4 hours downtime/year; realistic for single-site air-gapped) |
| Platform type | Kubernetes-based container orchestration |
| VM support | KubeVirt for legacy workloads requiring VMs |
| Staffing constraint | 12 cleared engineers for operations |
| Total engineering capacity | 40 engineers (28 unclearable for this system) |

### 2.3 Workload Characteristics (Assumed)

- Intelligence analysis applications (containerized)
- Geospatial processing (potentially GPU-accelerated)
- Database workloads (PostgreSQL, custom databases)
- Legacy applications requiring VM execution
- Messaging and collaboration tools (internal only)
- CI/CD pipelines for classified software development
- Document management and search

---

## 3. Accreditation Strategy

### 3.1 Accreditation Lifecycle

The accreditation process follows the Swedish model under FMV/MUST authority:

1. **Kategorisering (Categorization)** -- System is categorized as processing HEMLIG data. This has been completed: the platform handles national defense classified material at the Secret level.

2. **Sakerhetsskyddsanalys (Protective Security Analysis)** -- A formal protective security analysis must be conducted per Sakerhetsskyddslagen Ch. 2, identifying:
   - What information needs protection
   - Threat actors and threat scenarios
   - Vulnerabilities in the proposed architecture
   - Required protective measures (physical, personnel, information security)

3. **Sakerhetsskyddsplan (Protective Security Plan)** -- Documented plan covering:
   - Physical protective security (fysiskt sakerhetsskydd)
   - Personnel protective security (personsakerhet)
   - Information security (informationssakerhet)
   - Signaling protective security (signalskydd) -- crypto per FRA

4. **Implementation** -- Build the platform to specification with all controls in place

5. **Granskning (Assessment)** -- FMV/MUST assessors review the implementation:
   - Physical inspection of the facility
   - Technical security testing
   - Review of operational procedures
   - Personnel verification
   - Crypto implementation review by FRA

6. **Ackreditering (Accreditation)** -- MUST or delegated authority grants accreditation to process HEMLIG data

7. **Kontinuerlig uppfoljning (Continuous Monitoring)** -- Ongoing compliance monitoring with periodic reassessment, typically annually or upon significant change

### 3.2 Design-for-Accreditation Approach

Every architectural decision in this document is made with accreditation in mind. We do not design first and accredit later. Instead:

- Controls are mapped to Sakerhetsskyddslagen requirements in each section
- Automated compliance scanning is embedded from day one (OpenSCAP, custom policy checks)
- Documentation is generated as part of the build process (Infrastructure as Code produces auditable artifacts)
- Change management requires accreditation impact assessment

### 3.3 Engagement with Authorities

Engage FMV/MUST early and continuously:

- Initial consultation before detailed design begins
- Preliminary design review at 30% completion
- Detailed design review at 70% completion
- Pre-assessment walkthrough before formal assessment
- Maintain a designated security manager (sakerhetsskyddschef) as primary point of contact

---

## 4. Physical Security Architecture

### 4.1 Facility Requirements

The classified processing environment must be housed in a facility meeting HEMLIG physical security standards:

**Controlled Access Area (Skyddsobjekt)**:
- The facility must be designated as a skyddsobjekt under Skyddslagen (2010:305) if required
- Perimeter security: fencing, surveillance cameras, motion detection, lighting
- Access control: multi-factor (badge + PIN minimum), mantrap/airlock entry
- Guard force or monitored alarm with armed response
- Visitor management: escorts required, all visits logged
- Vehicle barriers at approach points

**Server Room (IT-utrymme for HEMLIG)**:
- Dedicated room with reinforced walls, floor, and ceiling
- Independent access control (separate from general facility access)
- Intrusion detection on all surfaces (walls, floor, ceiling, ducts)
- No windows; if windows exist, they must be opaque, alarmed, and meet attack resistance standards
- Environmental monitoring: temperature, humidity, water leak, smoke
- Fire suppression: gas-based (FM-200 or Novec 1230) to protect equipment
- UPS and generator backup with automatic failover
- Separate HVAC not shared with unclassified areas (emanation risk)

**TEMPEST Considerations (Signalskydd)**:
- Facility must meet NATO SDIP-27 Zone requirements as determined by MUST TEMPEST assessment
- For HEMLIG processing, typically Zone B or Zone A depending on threat assessment
- If Zone A: full Faraday cage shielding of the server room
- If Zone B: controlled separation distances from facility boundary, shielded cabling
- Red/black separation: classified (red) network cabling physically separated from any unclassified (black) cabling
- No classified and unclassified cables in the same conduit, cable tray, or wall penetration
- TEMPEST-rated KVM switches if multi-level workstations are used (unlikely needed if fully HEMLIG)
- FRA must approve the signalskydd (cryptographic protection) implementation

### 4.2 Rack Layout

```
+------------------------------------------------------------------+
|                     HEMLIG SERVER ROOM                             |
|                                                                    |
|  +----------+  +----------+  +----------+  +----------+           |
|  | Rack A1  |  | Rack A2  |  | Rack A3  |  | Rack A4  |  Row A   |
|  | Compute  |  | Compute  |  | Compute  |  | Storage  |           |
|  | Nodes    |  | Nodes    |  | Nodes    |  | Nodes    |           |
|  | 1-6      |  | 7-12     |  | 13-18    |  | (Ceph)   |           |
|  +----------+  +----------+  +----------+  +----------+           |
|                                                                    |
|  +----------+  +----------+  +----------+  +----------+           |
|  | Rack B1  |  | Rack B2  |  | Rack B3  |  | Rack B4  |  Row B   |
|  | Storage  |  | Storage  |  | Network  |  | Mgmt &   |           |
|  | (Ceph)   |  | (Ceph)   |  | Switches |  | Services |           |
|  |          |  |          |  | + FW     |  | + Backup |           |
|  +----------+  +----------+  +----------+  +----------+           |
|                                                                    |
|  [UPS A]  [UPS B]  [PDU-A]  [PDU-B]  [Transfer]  [Console]       |
|                                        [Station]   [Terminal]      |
+------------------------------------------------------------------+
```

**Key layout principles**:
- Dual PDU feeds (A/B) to every rack
- UPS sized for 15-minute runtime minimum (allows generator startup)
- Network switches in dedicated rack with structured cabling
- Transfer station: physically isolated workstation for data import (see Section 10)
- Console terminal: out-of-band management access point

---

## 5. Network Architecture

### 5.1 Air-Gapped Network Design

The HEMLIG network has zero connectivity to any external network. There is no internet access, no connection to company unclassified LAN, and no wireless of any kind.

```
+-------------------------------------------------------------------+
|                    HEMLIG NETWORK (AIR-GAPPED)                     |
|                                                                    |
|  +------------------+          +------------------+                |
|  | Spine Switch 1   |----------| Spine Switch 2   |   (SPINE)     |
|  | (Cisco Nexus     |          | (Cisco Nexus     |               |
|  |  9336C-FX2)      |          |  9336C-FX2)      |               |
|  +--------+---------+          +--------+---------+                |
|           |    \                    /        |                      |
|           |     \                  /         |                      |
|           |      \                /          |                      |
|  +--------+-------+------+------+-------+---+----+                |
|  |                |      |      |                |                 |
|  v                v      v      v                v                 |
|  +----------+ +----------+ +----------+ +----------+              |
|  | Leaf 1   | | Leaf 2   | | Leaf 3   | | Leaf 4   |  (LEAF)     |
|  | N9K-     | | N9K-     | | N9K-     | | N9K-     |             |
|  | C93180YC | | C93180YC | | C93180YC | | C93180YC |             |
|  +-----+----+ +----+-----+ +----+-----+ +----+-----+             |
|        |           |            |             |                    |
|   Compute      Compute      Storage       Management              |
|   Racks A1-A3  (overflow)   Racks A4,B1-B2  Rack B4              |
|                                                                    |
|  +----------+                                                      |
|  | OOB Mgmt |  (Out-of-Band Management Network)                   |
|  | Switch   |  -- IPMI/iDRAC/iLO only, no data path               |
|  +----------+                                                      |
+-------------------------------------------------------------------+
```

### 5.2 Network Fabric Design

**Topology**: Spine-leaf using Cisco Nexus 9000 series

**Why Cisco Nexus for a Swedish defense environment**:
- Cisco has established supply chain agreements with Swedish defense procurement
- Nexus 9000 supports VXLAN-EVPN which provides the segmentation needed
- Cisco ISE integration for 802.1X port-level authentication
- Well-understood by Swedish defense IT organizations
- Alternative considered: Juniper QFX -- also acceptable, but Cisco is more commonly pre-approved in Swedish defense contexts

**Fabric configuration**:
- VXLAN-EVPN fabric with BGP underlay
- Spine: 2x Cisco Nexus 9336C-FX2 (100G capable)
- Leaf: 4x Cisco Nexus C93180YC-FX (48x 25G + 6x 100G)
- All inter-switch links: 100GbE
- Server connectivity: 2x 25GbE per server (bonded, LACP)
- Storage network: dedicated VLAN/VNI, may use separate 25GbE ports on storage nodes for Ceph cluster traffic

**Network segmentation (VRFs/VNIs)**:

| Network Segment | VNI | Purpose | VLAN ID |
|---|---|---|---|
| k8s-node | 10010 | Kubernetes node-to-node communication | 10 |
| k8s-pod | 10020 | Pod network (Cilium overlay) | 20 |
| k8s-service | 10030 | Kubernetes service network | 30 |
| storage-public | 10040 | Ceph client access network | 40 |
| storage-cluster | 10050 | Ceph OSD replication traffic | 50 |
| management | 10060 | Management, monitoring, DNS, NTP | 60 |
| oob | N/A | Out-of-band IPMI/BMC (separate physical switch) | 99 |
| transfer | 10070 | Data transfer station (isolated, see Section 10) | 70 |

### 5.3 Firewall and Access Control

**Perimeter**: There is no perimeter firewall in the traditional sense because there is no perimeter -- the network is air-gapped. However:

- Inter-VLAN routing is controlled via ACLs on the leaf switches
- The transfer station VLAN (70) is isolated by hardware -- it connects only to the data import workstation and is reachable only from the management VLAN with explicit ACL rules
- Cilium network policies enforce microsegmentation at the pod level within Kubernetes (see Section 7)

**Host-level firewalling**:
- All nodes run nftables with default-deny ingress
- Only required ports are open per node role (e.g., kubelet ports on worker nodes, Ceph MON/OSD ports on storage nodes)

### 5.4 DNS and NTP

**DNS**: CoreDNS deployed within the Kubernetes cluster for service discovery. A pair of standalone BIND servers on management VLAN for infrastructure DNS (node names, BMC names, storage endpoints).

**NTP**: Critical for audit trail integrity. Two NTP servers on the management network synchronized to GPS-disciplined clocks (Meinberg or similar) physically located in the server room. No external NTP source is possible due to the air gap. GPS antenna is shielded and the signal path is unidirectional.

> **Accreditation note**: Accurate and tamper-evident time synchronization is a prerequisite for HEMLIG audit trail compliance. GPS-disciplined clocks with holdover capability (rubidium or OCXO) are required.

### 5.5 No Wireless

No WiFi, Bluetooth, or any wireless technology within the HEMLIG facility. This is a hard requirement for HEMLIG accreditation. All mobile devices must be stored outside the controlled area.

---

## 6. Compute Architecture

### 6.1 Server Selection

**Recommendation**: Dell PowerEdge R760 or HPE ProLiant DL380 Gen11

Selection criteria for HEMLIG environments:
- Servers must be procured through FMV-approved supply chains or directly from manufacturer with chain-of-custody documentation
- UEFI Secure Boot with verified firmware
- TPM 2.0 module (for measured boot and key sealing)
- iDRAC/iLO for out-of-band management on isolated OOB network
- Dual power supply (A/B feed)

### 6.2 Node Roles and Sizing

**Kubernetes Control Plane Nodes** (3 nodes):

| Component | Specification |
|---|---|
| CPU | 2x Intel Xeon Gold 6448Y (32C/64T) or equivalent |
| RAM | 256 GB DDR5 ECC |
| Boot/OS | 2x 480GB SSD (RAID1) |
| etcd | 2x 1.92TB NVMe SSD (RAID1, dedicated for etcd) |
| Network | 2x 25GbE (bonded) + 1GbE OOB |

**Kubernetes Worker Nodes -- General Purpose** (12 nodes):

| Component | Specification |
|---|---|
| CPU | 2x Intel Xeon Gold 6448Y (32C/64T) or equivalent |
| RAM | 512 GB DDR5 ECC |
| Boot/OS | 2x 480GB SSD (RAID1) |
| Network | 2x 25GbE (bonded) + 1GbE OOB |

**Kubernetes Worker Nodes -- GPU** (3 nodes, if geospatial/ML workloads):

| Component | Specification |
|---|---|
| CPU | 2x Intel Xeon Gold 6448Y (32C/64T) |
| RAM | 512 GB DDR5 ECC |
| GPU | 2x NVIDIA A100 80GB or H100 (depending on workload) |
| Boot/OS | 2x 480GB SSD (RAID1) |
| Network | 2x 25GbE (bonded) + 1GbE OOB |

> **Note**: GPU procurement for defense requires export license verification. NVIDIA A100/H100 are subject to export controls. Verify with FMV procurement.

**Ceph Storage Nodes** (5 nodes, minimum for HEMLIG reliability):

| Component | Specification |
|---|---|
| CPU | 2x Intel Xeon Silver 4416+ (20C/40T) |
| RAM | 256 GB DDR5 ECC |
| Boot/OS | 2x 480GB SSD (RAID1) |
| OSD drives | 10x 7.68TB NVMe SSD per node |
| WAL/DB | 2x 1.6TB NVMe (partitioned for WAL/DB acceleration) |
| Network | 2x 25GbE storage-public + 2x 25GbE storage-cluster + 1GbE OOB |

**Management/Infrastructure Nodes** (3 nodes):

| Component | Specification |
|---|---|
| CPU | 2x Intel Xeon Silver 4416+ (20C/40T) |
| RAM | 128 GB DDR5 ECC |
| Boot/OS | 2x 480GB SSD (RAID1) |
| Data | 2x 1.92TB SSD (for local services) |
| Network | 2x 25GbE (bonded) + 1GbE OOB |
| Purpose | Harbor registry, GitLab, Vault, monitoring, backup mgmt |

### 6.3 Total Capacity Summary

| Resource | Total |
|---|---|
| Compute nodes | 18 (12 general + 3 GPU + 3 control plane) |
| Storage nodes | 5 |
| Management nodes | 3 |
| Total servers | 26 |
| Total CPU cores | ~1,600 cores |
| Total RAM | ~10 TB |
| Raw storage (NVMe) | ~384 TB raw (5 nodes x 10 x 7.68TB) |
| Usable storage (3x replication) | ~128 TB usable |
| GPU | 6x NVIDIA A100/H100 (if deployed) |

### 6.4 Firmware and BIOS Hardening

All servers must undergo firmware hardening before deployment:

- Update to latest vendor-approved firmware (verified offline via checksums)
- Enable UEFI Secure Boot
- Set TPM 2.0 to active, owned by organization
- Disable unused I/O (USB ports on rear can remain for emergency console; front USB disabled)
- Disable unused boot devices (PXE disabled after initial provisioning)
- Set BIOS passwords (stored in Vault)
- Enable boot audit logging
- Configure iDRAC/iLO:
  - Change default credentials
  - Restrict to OOB VLAN only
  - Enable LDAP authentication against FreeIPA
  - Enable NTP synchronization
  - Enable audit logging

---

## 7. Platform Architecture -- Kubernetes

### 7.1 Distribution Selection: RKE2

**Decision**: RKE2 (Rancher Kubernetes Engine 2, also called "RKE Government")

**Rationale**:
- RKE2 was specifically designed for security-sensitive and air-gapped environments
- Ships with FIPS 140-2 validated crypto modules (relevant: Swedish authorities often accept FIPS as a baseline, with FRA-approved crypto layered on top for signalskydd)
- CIS Kubernetes Benchmark hardened out of the box
- Uses containerd (not Docker) as container runtime
- Embedded etcd (no external dependency)
- Canal (Calico + Flannel) as default CNI, but we replace with Cilium for advanced network policy
- Built-in support for air-gapped installation
- SELinux enforcing mode supported
- Active upstream community with SUSE enterprise backing

**Alternatives considered**:

| Distribution | Verdict | Reason |
|---|---|---|
| OpenShift | Rejected | Heavy licensing cost, operator complexity for 12-person team; Common Criteria certified which is a plus, but RKE2 meets needs without license burden |
| Talos Linux | Considered | Excellent security posture (immutable, API-only), but smaller community and less established in defense contexts |
| Vanilla kubeadm | Rejected | Too much manual hardening required; no FIPS support out of box |
| k3s | Rejected | Designed for edge/lightweight; lacks FIPS and CIS hardening defaults |

### 7.2 Cluster Architecture

```
+--------------------------------------------------------------------+
|                     RKE2 CLUSTER ARCHITECTURE                       |
|                                                                     |
|  CONTROL PLANE (3 nodes, HA)                                       |
|  +---------------+  +---------------+  +---------------+           |
|  | cp-node-01    |  | cp-node-02    |  | cp-node-03    |           |
|  | kube-apiserver|  | kube-apiserver|  | kube-apiserver|           |
|  | etcd          |  | etcd          |  | etcd          |           |
|  | kube-scheduler|  | kube-scheduler|  | kube-scheduler|           |
|  | controller-mgr|  | controller-mgr|  | controller-mgr|           |
|  | kube-vip (VIP)|  |               |  |               |           |
|  +---------------+  +---------------+  +---------------+           |
|         |                   |                  |                    |
|         +------- VIP: 10.x.x.100 (API) -------+                   |
|                                                                     |
|  WORKER NODES (12 general + 3 GPU)                                 |
|  +------------+ +------------+ +------------+ +------------+       |
|  | worker-01  | | worker-02  | | ...        | | worker-12  |       |
|  | kubelet    | | kubelet    | |            | | kubelet    |       |
|  | Cilium     | | Cilium     | |            | | Cilium     |       |
|  | containerd | | containerd | |            | | containerd |       |
|  +------------+ +------------+ +------------+ +------------+       |
|                                                                     |
|  +-------------+ +-------------+ +-------------+                   |
|  | gpu-wrk-01  | | gpu-wrk-02  | | gpu-wrk-03  |                  |
|  | NVIDIA GPU  | | NVIDIA GPU  | | NVIDIA GPU  |                  |
|  | Operator    | | Operator    | | Operator    |                  |
|  +-------------+ +-------------+ +-------------+                   |
+--------------------------------------------------------------------+
```

### 7.3 Control Plane Configuration

- **HA**: 3 control plane nodes with embedded etcd (Raft consensus)
- **API Server VIP**: kube-vip in ARP mode providing a virtual IP for the API server
- **etcd**: Dedicated NVMe SSDs, encrypted at rest using LUKS
- **API Server**: Audit logging enabled with full request/response logging for write operations
- **Admission controllers**: Always-on: NodeRestriction, PodSecurity (Restricted profile default), plus:
  - Kyverno for custom policy enforcement
  - OPA Gatekeeper as secondary policy engine for complex policies
- **RBAC**: Integrated with FreeIPA via OIDC (Keycloak as OIDC provider bridging to FreeIPA)
- **Encryption at rest**: etcd encryption enabled for Secrets, ConfigMaps using AES-CBC with keys managed in Vault

### 7.4 Container Networking -- Cilium

**Decision**: Cilium replaces the default Canal CNI

**Rationale**:
- eBPF-based networking provides high performance with deep visibility
- CiliumNetworkPolicy supports L3/L4/L7 policy enforcement
- Built-in encryption (WireGuard or IPsec) for pod-to-pod traffic
- Hubble for network observability (critical for security monitoring in HEMLIG environments)
- No dependency on iptables (cleaner, more auditable)
- DNS-aware policies allow controlling egress by FQDN (useful even in air-gapped to prevent lateral movement)

**Network policy strategy**:
- Default deny all ingress and egress in every namespace
- Explicit allow rules per application
- Namespace-level isolation enforced
- Hubble flow logs exported to SIEM for security monitoring
- All pod-to-pod traffic encrypted with WireGuard (defense in depth; even within the air gap, this limits blast radius of a compromised node)

### 7.5 KubeVirt for VM Workloads

For legacy applications that cannot be containerized:

- KubeVirt deployed as an operator within the RKE2 cluster
- VM definitions managed as Kubernetes custom resources (GitOps-managed)
- Multus CNI for VMs requiring multiple network interfaces
- CDI (Containerized Data Importer) for importing VM disk images
- Live migration supported between worker nodes
- VMs managed with the same RBAC, network policy, and audit controls as pods

### 7.6 Ingress and Service Mesh

- **Ingress controller**: NGINX Ingress Controller or Contour (Envoy-based)
- **Internal load balancing**: MetalLB in L2 mode (no external integration needed in air-gapped)
- **Service mesh**: Cilium service mesh (preferred over Istio for reduced complexity)
  - mTLS between all services
  - L7 visibility and policy
  - Traffic encryption
- **TLS certificates**: cert-manager with internal CA (see Section 9)

### 7.7 Namespace Strategy

```
Namespaces:
  system/
    kube-system          -- Core Kubernetes components
    cilium-system        -- Cilium CNI and Hubble
    metallb-system       -- MetalLB load balancer
    cert-manager         -- Certificate management
    ingress-system       -- Ingress controller

  platform/
    monitoring           -- Prometheus, Grafana, Alertmanager
    logging              -- Loki, Promtail/Alloy
    vault                -- HashiCorp Vault
    argocd               -- ArgoCD GitOps
    harbor               -- Container registry (if in-cluster)
    kyverno              -- Policy engine
    falco                -- Runtime security
    kubevirt             -- VM management
    rook-ceph            -- Ceph operator

  workloads/
    app-<team>-<env>     -- Per-team, per-environment namespaces
    shared-services      -- Shared databases, message queues
    ci-cd                -- Build pipelines (Tekton)
```

All workload namespaces have:
- ResourceQuota and LimitRange applied
- Default deny NetworkPolicy
- PodSecurity admission set to "restricted"
- Kyverno policies requiring image signatures, resource limits, and security contexts

---

## 8. Storage Architecture

### 8.1 Ceph Cluster

**Decision**: Rook-Ceph deployed as Kubernetes operator

**Configuration**:
- 5 storage nodes, 10 OSDs per node = 50 OSDs
- Replication factor: 3 (three copies of every piece of data)
- Failure domain: host-level (can lose any single storage node without data loss)
- BlueStore on NVMe SSDs
- Dedicated WAL/DB devices for performance
- Ceph monitors co-located on storage nodes (3 monitors, one on each of 3 nodes)
- Ceph manager daemons for monitoring integration

**Storage classes provided to Kubernetes**:

| StorageClass | Ceph Pool | Replication | Use Case |
|---|---|---|---|
| ceph-block-ssd | rbd-ssd | 3x replicated | General database, application state |
| ceph-block-fast | rbd-fast | 3x replicated, NVMe-only | High-IOPS workloads (etcd backup, databases) |
| ceph-fs | cephfs | 3x replicated | Shared filesystem (ReadWriteMany) |
| ceph-object | rgw | 3x replicated | S3-compatible object storage (via Ceph RGW) |

### 8.2 Encryption at Rest

**All data at rest must be encrypted** -- this is a HEMLIG requirement:

- **LUKS (dm-crypt)** on all OSD devices before Ceph initialization
- Keys managed by Vault and sealed to TPM 2.0 on each storage node
- Automatic unlock at boot using TPM-sealed keys (no manual intervention for planned restarts)
- Boot drives encrypted with LUKS, key sealed to TPM
- etcd volumes encrypted with LUKS
- Kubernetes Secrets additionally encrypted in etcd using AES-CBC envelope encryption

> **Accreditation note**: FRA may require specific national cryptographic algorithms for encryption at rest at HEMLIG level. Consult FRA/MUST early. LUKS supports pluggable ciphers; AES-256-XTS is typically acceptable, but confirmation is required.

### 8.3 Backup

**Backup solution**: Velero for Kubernetes resources + Ceph RBD snapshots for persistent volumes

- Velero backs up to a separate Ceph RGW (S3) bucket on the same cluster
- Backup schedule: daily incremental, weekly full
- Retention: 30 daily, 12 weekly, 12 monthly
- Ceph RBD snapshots for point-in-time recovery of databases
- Backup encryption: all backups encrypted using Vault-managed keys
- Backup verification: automated restore tests weekly (to a temporary namespace, then destroyed)

**Offsite backup** (for DR):
- Encrypted backup tapes or encrypted removable media
- Transported to a secondary HEMLIG-accredited storage facility under armed courier
- This is the physical air-gap equivalent of offsite replication

---

## 9. Identity and Access Management

### 9.1 Architecture

```
+--------------------------------------------------------------------+
|                    IDENTITY ARCHITECTURE                            |
|                                                                     |
|  +------------+      +------------+      +-------------------+     |
|  | FreeIPA    |----->| Keycloak   |----->| Kubernetes OIDC   |     |
|  | (LDAP/     |      | (OIDC/SAML |      | (API Server auth) |     |
|  |  Kerberos) |      |  Provider) |      |                   |     |
|  +-----+------+      +------+-----+      +-------------------+     |
|        |                     |                                      |
|        |                     +-----------> Harbor (registry auth)   |
|        |                     +-----------> Vault (auth backend)     |
|        |                     +-----------> Grafana (dashboard auth) |
|        |                     +-----------> GitLab (code auth)       |
|        |                     +-----------> ArgoCD (GitOps auth)     |
|        |                                                            |
|        +---> Linux PAM (node SSH access)                           |
|        +---> iDRAC/iLO LDAP (BMC auth)                             |
|        +---> Cisco ISE (network device auth)                       |
+--------------------------------------------------------------------+
```

### 9.2 FreeIPA

**Decision**: FreeIPA as the core identity provider (FLOSS alternative to Active Directory)

**Rationale**:
- No licensing costs
- Native Linux integration (SSSD, Kerberos, LDAP)
- HBAC (Host-Based Access Control) for fine-grained SSH access
- sudo rules centrally managed
- Certificate authority for internal PKI
- DNS integration (secondary to BIND if needed)
- 2 FreeIPA replicas on management nodes for HA

**User model**:

| Group | Members | Access |
|---|---|---|
| k8s-admins | 3 senior platform engineers | Cluster-admin, all namespaces |
| k8s-operators | 6 platform engineers | Namespace-admin for platform namespaces, read-only cluster |
| k8s-developers | 12 cleared engineers | Namespace-admin for assigned workload namespaces only |
| storage-admins | 3 platform engineers | Ceph dashboard, storage management |
| security-admins | 2 security engineers | Vault admin, Falco management, audit log access |
| readonly-audit | 2 designated auditors | Read-only access to all audit logs and dashboards |

> **Note**: All 12 cleared engineers hold roles; some engineers hold multiple roles. This is an acknowledged risk given the staffing constraint. Mitigated by comprehensive audit logging and four-eyes principle for destructive operations.

### 9.3 Keycloak

- OIDC/SAML identity broker fronting FreeIPA
- Provides OIDC tokens to Kubernetes API server for authentication
- MFA enforced: hardware token (YubiKey) + password
- Session timeout: 8 hours maximum, re-authentication required
- All authentication events logged and forwarded to SIEM

### 9.4 Internal PKI

- FreeIPA CA as root CA for all internal certificates
- cert-manager in Kubernetes requests certificates from FreeIPA CA via ACME or the FreeIPA cert plugin
- All internal communication uses TLS 1.3 (TLS 1.2 minimum where 1.3 is not supported)
- Certificate rotation automated via cert-manager
- Short-lived certificates: 90-day maximum validity, auto-renewed

> **Accreditation note**: FRA may mandate specific certificate requirements or use of nationally approved crypto for signalskydd. The internal PKI may need to be subordinated to an FRA-approved CA chain. Engage FRA early.

### 9.5 Secrets Management -- HashiCorp Vault

- Deployed in HA mode (3 instances, Raft storage backend)
- Auto-unseal via TPM 2.0 or Shamir shares held by 3 of 5 key holders (cleared personnel)
- All application secrets stored in Vault, injected via:
  - Vault Agent Sidecar Injector (for pods)
  - External Secrets Operator (syncing Vault secrets to Kubernetes Secrets)
- Dynamic secrets for database credentials (PostgreSQL, etc.)
- PKI secrets engine as secondary CA if needed
- Audit logging enabled -- every secret access is logged with accessor identity

---

## 10. Air-Gapped Operations

This is the most operationally challenging aspect of the platform. Every piece of software, every update, every container image must be brought into the air-gapped environment through a controlled, audited process.

### 10.1 Data Transfer Architecture

```
+-------------------------------------------------------------------+
|                                                                     |
|  UNCLASSIFIED                          HEMLIG (AIR-GAPPED)         |
|  NETWORK                               NETWORK                     |
|                                                                     |
|  +--------------+                      +------------------+        |
|  | Build &      |                      | Transfer Station |        |
|  | Staging      |   [Removable Media]  | (Isolated Host)  |        |
|  | Server       |------- via --------->|                  |        |
|  | (internet-   |  approved process    | - Malware scan   |        |
|  |  connected)  |                      | - Signature      |        |
|  +--------------+                      |   verification   |        |
|                                        | - Manifest check |        |
|                                        | - Audit logging  |        |
|                                        +--------+---------+        |
|                                                 |                   |
|                                                 v                   |
|                                        +------------------+        |
|                                        | Harbor Registry  |        |
|                                        | / Artifact Repo  |        |
|                                        +------------------+        |
|                                                 |                   |
|                                                 v                   |
|                                        +------------------+        |
|                                        | Production       |        |
|                                        | Kubernetes       |        |
|                                        | Cluster          |        |
|                                        +------------------+        |
+-------------------------------------------------------------------+
```

### 10.2 Transfer Process (Sneakernet)

**Step 1: Preparation (Unclassified side)**

On the internet-connected build server:
1. Pull required container images, OS packages, Helm charts, Ansible collections
2. Run vulnerability scanning (Trivy, Grype) on all artifacts
3. Generate SBOMs (Syft) for all container images
4. Sign all artifacts with cosign (organization's signing key)
5. Generate manifest file listing all artifacts with SHA256 checksums
6. Sign the manifest file
7. Write all artifacts to approved removable media (encrypted, write-once media preferred)
8. Two-person integrity (TPI): a second cleared person verifies the manifest

**Step 2: Transfer**

1. Removable media is logged in the transfer register
2. Media is physically transported to the HEMLIG facility
3. Chain of custody documented at every handoff

**Step 3: Import (HEMLIG side)**

1. Media is inserted into the Transfer Station (isolated host on VLAN 70)
2. Transfer Station is a hardened Linux workstation with:
   - No network access except to management VLAN via explicit ACL
   - ClamAV malware scanning
   - cosign signature verification
   - SHA256 checksum verification against the signed manifest
   - SBOM review tooling
3. If all checks pass, artifacts are pushed to the internal Harbor registry and/or package repositories
4. Media is securely wiped or physically destroyed after import (per HEMLIG media handling procedures)
5. Transfer is logged with: who, what, when, source, checksums, verification results

**Step 4: Deployment**

1. ArgoCD detects new images/charts in Harbor and initiates deployment per GitOps workflow
2. Kyverno admission policies verify that images are signed and present in the approved registry before allowing deployment

### 10.3 Offline Repositories

The following repositories are maintained within the air-gapped environment:

| Repository | Tool | Content |
|---|---|---|
| Container images | Harbor | All application and system container images |
| Helm charts | Harbor (OCI) or ChartMuseum | Kubernetes deployment charts |
| OS packages (RPM) | Pulp | RHEL/Rocky Linux packages for node OS |
| OS packages (DEB) | Aptly | If Ubuntu-based nodes are used |
| Python packages | Pulp or devpi | Python wheels for build pipelines |
| Ansible collections | Local Galaxy mirror | Ansible automation content |
| Git repositories | GitLab (self-hosted) | All source code and IaC |

### 10.4 Update Cadence

| Update Type | Frequency | Process |
|---|---|---|
| Critical security patches | Within 72 hours of release | Emergency transfer process (expedited but still audited) |
| OS security updates | Bi-weekly | Standard transfer process |
| Kubernetes minor upgrades | Quarterly | Tested in staging namespace first, then rolling update |
| Application updates | Per sprint/release | Standard transfer and GitOps deployment |
| Ceph upgrades | Semi-annually | Carefully planned, rolling upgrade with I/O monitoring |
| Firmware updates | Annually or as needed | Vendor-verified, staged rollout |

---

## 11. Security Architecture

### 11.1 Defense in Depth

```
Layer 1: Physical Security
  |-- Facility access control, guards, intrusion detection
  |-- TEMPEST shielding
  |
Layer 2: Network Security
  |-- Air gap (physical isolation)
  |-- VXLAN-EVPN segmentation
  |-- ACLs on leaf switches
  |-- No wireless
  |
Layer 3: Host Security
  |-- Hardened OS (CIS benchmark, DISA STIG applied)
  |-- SELinux enforcing
  |-- LUKS encryption at rest
  |-- TPM-based measured boot
  |-- nftables host firewall
  |
Layer 4: Platform Security
  |-- RKE2 CIS-hardened Kubernetes
  |-- Cilium network policies (default deny)
  |-- Pod Security Standards (restricted)
  |-- Kyverno/OPA admission policies
  |-- Image signature verification
  |-- etcd encryption
  |
Layer 5: Application Security
  |-- mTLS via Cilium service mesh
  |-- Vault-managed secrets
  |-- RBAC per namespace
  |-- Runtime monitoring (Falco, Tetragon)
  |
Layer 6: Data Security
  |-- Encryption at rest (LUKS, etcd encryption)
  |-- Encryption in transit (TLS 1.3, WireGuard)
  |-- Data classification labels
  |-- Audit logging on all data access
```

### 11.2 OS Hardening

Base OS: Rocky Linux 9 (or RHEL 9 if vendor support is required)

Hardening applied via Ansible at provisioning time:
- CIS Rocky Linux 9 Benchmark Level 2 applied
- SELinux set to enforcing mode
- FIPS mode enabled in kernel
- Unnecessary services disabled
- USB mass storage disabled (kernel module blacklisted)
- Core dump disabled
- Audit daemon (auditd) configured with comprehensive rules:
  - All authentication events
  - All privilege escalation
  - All file access to sensitive paths
  - All administrative commands
  - All kernel module operations
- Password policy: minimum 16 characters, complexity requirements (though SSH key + MFA is the primary auth method)
- SSH hardened: key-only authentication, no root login, AllowGroups restricted, MFA via PAM
- Automatic security updates disabled (managed through the air-gapped update process)
- OpenSCAP scans run nightly; results exported to monitoring

### 11.3 Runtime Security

**Falco**:
- Deployed as DaemonSet on all worker nodes
- Custom rules for HEMLIG-specific detections:
  - Unexpected process execution in containers
  - File access outside expected paths
  - Network connections to unexpected endpoints
  - Privilege escalation attempts
  - Container escape attempts
  - Sensitive file reads (e.g., /etc/shadow, /proc/kcore)
- Alerts forwarded to Prometheus/Alertmanager and SIEM

**Tetragon** (Cilium):
- eBPF-based runtime enforcement
- Process execution policies per pod
- File integrity monitoring
- Network activity correlation with process identity

### 11.4 Supply Chain Security for Containers

- All container images must originate from the internal Harbor registry
- Kyverno ClusterPolicy enforces:
  - Images must be from `harbor.hemlig.internal/*` only
  - Images must be signed with the organization's cosign key
  - Images must have a passing Trivy scan (no Critical/High CVEs, or explicitly approved exceptions)
  - Images must have an SBOM attached
- Harbor configured with:
  - Vulnerability scanning on push (Trivy scanner)
  - Immutable tags for production images
  - Robot accounts for CI/CD (no human credentials for automated pulls)
  - Audit logging of all pull/push operations

### 11.5 Compliance Automation

- **OpenSCAP**: Nightly scans of all nodes against CIS and STIG profiles
- **Kyverno Policy Reporter**: Dashboard showing policy compliance across all namespaces
- **Custom compliance operator**: Checks HEMLIG-specific requirements:
  - Encryption at rest verified
  - Network policies present in every namespace
  - No pods running as root (unless explicitly approved)
  - All PVCs using encrypted storage classes
  - Audit logging enabled on all components
- Results aggregated in Grafana compliance dashboard
- Non-compliance triggers immediate alert to security-admins

---

## 12. Observability and Audit

### 12.1 Monitoring Stack

```
+--------------------------------------------------------------------+
|                    OBSERVABILITY ARCHITECTURE                       |
|                                                                     |
|  +-----------+    +-------------+    +------------+                |
|  | Prometheus |-->| Thanos      |--->| Grafana    |                |
|  | (metrics)  |   | (long-term  |    | (dashboards|                |
|  |            |   |  storage)   |    |  & alerts) |                |
|  +-----------+    +-------------+    +------------+                |
|                                                                     |
|  +-----------+    +-------------+    +------------+                |
|  | Promtail/ |-->| Loki        |--->| Grafana    |                |
|  | Alloy     |   | (log        |    | (log       |                |
|  | (log      |   |  aggregation|    |  search)   |                |
|  |  collect) |   |             |    |            |                |
|  +-----------+    +-------------+    +------------+                |
|                                                                     |
|  +-----------+    +-------------+                                  |
|  | Hubble    |--->| Grafana     |                                  |
|  | (network  |    | (network    |                                  |
|  |  flows)   |    |  visibility)|                                  |
|  +-----------+    +-------------+                                  |
|                                                                     |
|  +-----------+    +-------------+                                  |
|  | Falco     |--->| Alertmanager|---> On-call (PagerDuty           |
|  | (runtime  |    | (alerting)  |     equivalent on internal       |
|  |  security)|    |             |     system or SMS gateway)       |
|  +-----------+    +-------------+                                  |
+--------------------------------------------------------------------+
```

### 12.2 Audit Trail

**This is a critical accreditation requirement.** Every action must be attributable to an individual.

**Audit sources**:

| Source | What It Captures | Retention |
|---|---|---|
| Kubernetes audit log | All API server requests (who did what to which resource) | 5 years |
| Linux auditd | All system calls on nodes (file access, process exec, auth) | 5 years |
| FreeIPA/Keycloak | All authentication and authorization events | 5 years |
| Vault audit log | All secret accesses and management operations | 5 years |
| Harbor audit log | All image push/pull operations | 5 years |
| GitLab audit log | All code changes, merge requests, CI/CD runs | 5 years |
| Cilium/Hubble | All network flows between pods | 1 year (volume) |
| Falco | All security events and anomalies | 5 years |
| Transfer station log | All data imports with checksums and approvals | 10 years |
| Physical access log | All facility entry/exit events | 5 years |
| Network switch logs | All port up/down, ACL hits, authentication events | 2 years |

**Audit log integrity**:
- Logs are written to append-only storage (Ceph RGW with object lock / WORM)
- Logs are signed with a dedicated audit signing key
- Log forwarding uses mutual TLS
- Separate audit log access controls (only security-admins and readonly-audit groups)
- Regular audit log integrity verification (automated)

### 12.3 Alerting

- Alertmanager routes alerts to:
  - Internal messaging system (Mattermost or equivalent deployed within the HEMLIG environment)
  - On-call rotation via internal alerting system
  - For critical security events: SMS gateway (one-way, outbound only, if approved by accreditation authority; otherwise physical alerting via dedicated terminal)
- Alert classification: P1 (immediate), P2 (1 hour), P3 (next business day), P4 (informational)

---

## 13. Disaster Recovery and Business Continuity

### 13.1 RTO and RPO

| Scenario | RPO | RTO |
|---|---|---|
| Single node failure | 0 (no data loss) | 5 minutes (automatic) |
| Single rack failure | 0 (no data loss) | 15 minutes (automatic) |
| Ceph OSD failure | 0 (3x replication) | Self-healing, 30 min to rebalance |
| Control plane node loss | 0 (etcd quorum maintained) | 5 minutes (automatic) |
| Full facility loss | 24 hours (last offsite backup) | 72 hours (rebuild at alternate site) |
| Ransomware/compromise | 4 hours (last snapshot) | 8 hours (restore from clean backup) |

### 13.2 Backup Strategy

- **Kubernetes state**: Velero daily backups to Ceph RGW (encrypted, WORM)
- **Persistent volumes**: Ceph RBD snapshots every 4 hours, retained 7 days
- **Databases**: Application-level logical backups (pg_dump etc.) daily, WAL archiving for point-in-time recovery
- **Configuration**: All configuration in Git (GitLab). Git is the source of truth. Backed up to Ceph.
- **Secrets**: Vault has its own backup procedure; Raft snapshots encrypted and stored in Ceph
- **Offsite**: Encrypted backup tapes transported to secondary HEMLIG facility weekly

### 13.3 DR Site Planning

For a single-site deployment, full DR requires a secondary HEMLIG-accredited facility. Planning considerations:

- Secondary site must meet the same physical security, TEMPEST, and accreditation requirements
- Hardware pre-positioned or rapid procurement agreement with vendor
- RKE2 cluster can be rebuilt from Git (IaC) and restored from Velero backups
- Estimated rebuild time: 48-72 hours once hardware is available
- DR drill: annual (minimum), with MUST notification

---

## 14. Operational Model and Staffing

### 14.1 Team Structure

With only 12 cleared engineers, the team must be organized for maximum coverage while maintaining separation of duties where possible.

```
+--------------------------------------------------------------------+
|                    CLEARED TEAM (12 engineers)                      |
|                                                                     |
|  Platform Team (6 engineers)                                        |
|  +---+---+---+---+---+---+                                        |
|  | Platform Lead          |  -- Architecture, accreditation liaison |
|  | K8s Engineer (Sr)      |  -- RKE2, Cilium, ArgoCD               |
|  | K8s Engineer           |  -- Workload support, troubleshooting   |
|  | Storage Engineer       |  -- Ceph, backup, DR                   |
|  | Network Engineer       |  -- Cisco Nexus, network ops            |
|  | Automation Engineer    |  -- Ansible, IaC, CI/CD pipelines       |
|  +------------------------+                                        |
|                                                                     |
|  Security & Operations Team (3 engineers)                           |
|  +---+---+---+                                                     |
|  | Security Lead          |  -- Sakerhetsskyddschef liaison, policy |
|  | Security Engineer      |  -- Vault, Falco, compliance, audit    |
|  | Ops/SRE Engineer       |  -- Monitoring, on-call, incident resp. |
|  +------------------------+                                        |
|                                                                     |
|  Application Platform Team (3 engineers)                            |
|  +---+---+---+                                                     |
|  | Dev Platform Lead      |  -- Developer experience, onboarding   |
|  | DevOps Engineer        |  -- CI/CD, Harbor, GitLab              |
|  | DevOps Engineer        |  -- App support, troubleshooting       |
|  +------------------------+                                        |
+--------------------------------------------------------------------+

  Unclearable Engineers (28) -- work on UNCLASSIFIED systems only
  -- Can develop and test on unclassified environments
  -- Container images built on unclassified side, transferred in
  -- Cannot access the HEMLIG environment
```

### 14.2 On-Call

- Minimum 2 engineers on-call at all times (one platform, one security)
- On-call rotation across 12 engineers: 6 rotation pairs, each pair on-call for 1 week
- Escalation path: on-call engineer -> team lead -> platform lead -> management
- All on-call communication via internal systems only (no external phone/messaging for incident details)

### 14.3 Four-Eyes Principle

For high-impact operations, the four-eyes principle (tva-personers-regel) is required:

- Cluster upgrades
- Ceph pool or replication changes
- Vault unseal operations
- Security policy changes
- Data transfer imports
- User privilege changes (granting cluster-admin, etc.)
- Firewall/ACL changes
- Backup restore operations

This is implemented via:
- Git merge request approvals (minimum 2 approvers for protected branches)
- Vault requires 2 of 3 key holders for unseal
- ArgoCD sync for critical applications requires manual approval from 2 engineers

### 14.4 Leveraging the 28 Uncleared Engineers

The 28 engineers who are not cleared for HEMLIG can still contribute significantly:

- **Develop applications** on an unclassified development environment (separate network, separate cluster)
- **Build and test container images** on the unclassified side
- **Write Ansible playbooks and Terraform modules** that are reviewed and tested in unclassified before transfer
- **Develop Helm charts** and Kubernetes manifests
- **Run integration tests** on an unclassified mirror environment
- **Document** operational procedures (that don't contain classified details)

The unclassified development environment should mirror the HEMLIG platform as closely as possible (same RKE2 version, same Cilium, same tooling) so that workloads tested there will work when transferred.

---

## 15. Supply Chain Security

### 15.1 Hardware Supply Chain

- All servers, switches, and components procured through FMV-approved supply chains or directly from manufacturers with chain-of-custody documentation
- Delivery to a secure receiving area; tamper-evident seals inspected
- Firmware hashes verified against manufacturer-published checksums
- Serial numbers recorded in asset management system (NetBox, deployed on management nodes)
- Random sample firmware inspection recommended for high-assurance

### 15.2 Software Supply Chain

- All open-source software pulled from upstream on the unclassified side
- Verified using GPG signatures, checksums, and reproducible build verification where available
- SBOM generated for every artifact entering the HEMLIG environment
- Vulnerability database (Trivy DB, NVD mirrors) regularly updated on unclassified side and transferred in
- No direct vendor access to the HEMLIG environment (all support is via documented procedures on the unclassified side, with classified details sanitized)

### 15.3 Software Bill of Materials (SBOM)

Every container image and software package in the HEMLIG environment must have a corresponding SBOM:

- Generated using Syft on the unclassified build server
- Stored in Harbor alongside the container image (OCI artifact)
- Reviewed during the transfer import process
- Queryable for vulnerability impact analysis (e.g., "which images contain log4j?")

---

## 16. Infrastructure as Code and Automation

### 16.1 GitOps Model

**Everything is in Git. Git is the source of truth.**

```
+--------------------------------------------------------------------+
|                    GITOPS WORKFLOW                                   |
|                                                                     |
|  GitLab (HEMLIG)                                                   |
|  +------------------+          +------------------+                |
|  | infrastructure/  |--------->| ArgoCD           |                |
|  |   cluster-config/|          | (sync to cluster)|                |
|  |   namespaces/    |          +------------------+                |
|  |   network-policy/|                   |                          |
|  |   monitoring/    |                   v                          |
|  |   security/      |          +------------------+                |
|  +------------------+          | RKE2 Cluster     |                |
|                                +------------------+                |
|  +------------------+                                              |
|  | ansible/         |--------->  Ansible (node config, OS updates) |
|  |   playbooks/     |           Triggered manually or via CI       |
|  |   roles/         |                                              |
|  |   inventory/     |                                              |
|  +------------------+                                              |
|                                                                     |
|  +------------------+                                              |
|  | terraform/       |--------->  OpenTofu (Ceph pools, Vault cfg,  |
|  |   modules/       |           FreeIPA users/groups)              |
|  |   environments/  |                                              |
|  +------------------+                                              |
+--------------------------------------------------------------------+
```

### 16.2 Repository Structure

```
hemlig-platform/
  README.md
  cluster/
    rke2/
      server.yaml              -- RKE2 server configuration
      agent.yaml               -- RKE2 agent configuration
      registries.yaml          -- Registry mirror config (Harbor)
    argocd/
      applications/            -- ArgoCD Application manifests
      projects/                -- ArgoCD AppProject definitions
  infrastructure/
    cilium/                    -- Cilium Helm values
    rook-ceph/                 -- Rook-Ceph operator and cluster config
    metallb/                   -- MetalLB config
    cert-manager/              -- cert-manager and ClusterIssuers
    ingress/                   -- Ingress controller config
  platform/
    monitoring/                -- Prometheus, Grafana, Alertmanager, Loki
    security/
      falco/                   -- Falco rules and config
      kyverno/                 -- Kyverno ClusterPolicies
      tetragon/                -- Tetragon policies
    vault/                     -- Vault Helm values and policies
    harbor/                    -- Harbor config
    gitlab/                    -- GitLab config (if in-cluster)
    keycloak/                  -- Keycloak realm and client config
  workloads/
    <team-name>/
      <app-name>/
        kustomization.yaml
        deployment.yaml
        service.yaml
        networkpolicy.yaml
  policies/
    network-policies/          -- Default deny policies per namespace
    pod-security/              -- PodSecurity configurations
    kyverno/                   -- Admission policies
    opa/                       -- OPA/Gatekeeper constraints
  ansible/
    playbooks/
      site.yaml                -- Main playbook
      harden.yaml              -- OS hardening
      rke2-install.yaml        -- RKE2 installation
      ceph-prepare.yaml        -- Ceph node preparation
    roles/
      common/                  -- Base OS configuration
      hardening/               -- CIS/STIG hardening
      rke2-server/             -- RKE2 control plane
      rke2-agent/              -- RKE2 worker node
      ceph-node/               -- Ceph OSD preparation
    inventory/
      hemlig/
        hosts.yaml             -- Node inventory
        group_vars/
          all.yaml             -- Common variables
          control_plane.yaml
          workers.yaml
          storage.yaml
  terraform/
    modules/
      ceph-pool/
      vault-config/
      freeipa-users/
    environments/
      hemlig-prod/
```

### 16.3 Ansible Automation

Key Ansible playbooks:

- **Node provisioning**: Base OS install (Kickstart/preseed), network config, LUKS setup, hardening
- **RKE2 installation**: Automated RKE2 server and agent deployment
- **OS hardening**: CIS Level 2 + STIG controls applied
- **Certificate rotation**: Automated renewal of node certificates
- **Firmware updates**: Staged firmware application with automatic rollback
- **Compliance scanning**: OpenSCAP scan execution and result collection

All playbooks tested on the unclassified mirror environment before execution in HEMLIG.

### 16.4 OpenTofu

Used for managing stateful infrastructure components where declarative management is beneficial:

- Ceph pool creation and configuration
- Vault policy and auth backend configuration
- FreeIPA user, group, and HBAC rule management
- Harbor project and robot account creation
- Keycloak realm and client configuration

State stored in an encrypted backend within the HEMLIG environment (Consul or PostgreSQL on the management nodes).

---

## 17. Migration and Onboarding

### 17.1 Phased Deployment

**Phase 1: Foundation (Weeks 1-6)**
- Physical facility preparation and TEMPEST assessment
- Hardware procurement and delivery
- Network fabric installation and configuration
- Node provisioning and hardening
- RKE2 cluster bootstrap (control plane + 3 worker nodes)
- Ceph cluster initialization
- Core platform services: FreeIPA, Keycloak, Vault, cert-manager

**Phase 2: Platform Services (Weeks 7-10)**
- Harbor registry deployment and initial image sync
- GitLab deployment
- Monitoring stack (Prometheus, Grafana, Loki, Alertmanager)
- ArgoCD deployment and GitOps workflow establishment
- Cilium network policy enforcement
- Kyverno policy deployment
- Falco and Tetragon deployment

**Phase 3: Workload Onboarding (Weeks 11-14)**
- Remaining worker nodes added to cluster
- GPU nodes added with NVIDIA GPU Operator
- KubeVirt deployment for VM workloads
- First application team onboarded
- CI/CD pipelines established (Tekton)
- Backup and DR procedures tested

**Phase 4: Accreditation (Weeks 15-20)**
- Comprehensive security testing
- Documentation completion
- Compliance scanning and remediation
- FMV/MUST preliminary review
- Formal accreditation assessment
- Remediation of any findings
- Accreditation granted

**Phase 5: Full Operations (Week 21+)**
- All application teams onboarded
- Continuous monitoring operational
- First DR drill conducted
- Operational handover complete

### 17.2 Application Onboarding Process

For each new application team:

1. Namespace creation via Git (merge request to hemlig-platform repo)
2. ResourceQuota and LimitRange configured per team allocation
3. NetworkPolicy (default deny) automatically applied
4. RBAC roles created in FreeIPA/Keycloak, bound to Kubernetes RBAC
5. Harbor project created for team's container images
6. CI/CD pipeline template provided (Tekton)
7. Team briefing on operational procedures (transfer process, incident handling, etc.)

---

## 18. Cost and Capacity Planning

### 18.1 Hardware Cost Estimate

| Component | Quantity | Unit Cost (SEK) | Total (SEK) |
|---|---|---|---|
| Compute servers (control plane) | 3 | 250,000 | 750,000 |
| Compute servers (workers, general) | 12 | 350,000 | 4,200,000 |
| Compute servers (workers, GPU) | 3 | 800,000 | 2,400,000 |
| Storage servers (Ceph) | 5 | 500,000 | 2,500,000 |
| Management servers | 3 | 200,000 | 600,000 |
| Cisco Nexus 9336C-FX2 (spine) | 2 | 400,000 | 800,000 |
| Cisco Nexus C93180YC-FX (leaf) | 4 | 250,000 | 1,000,000 |
| OOB management switch | 2 | 50,000 | 100,000 |
| UPS (rack-mount) | 4 | 100,000 | 400,000 |
| PDUs | 8 | 15,000 | 120,000 |
| Cabling, optics, accessories | -- | -- | 500,000 |
| GPS time source (Meinberg) | 2 | 80,000 | 160,000 |
| Transfer station workstation | 1 | 50,000 | 50,000 |
| **Hardware subtotal** | | | **~13,580,000** |

### 18.2 Facility Cost Estimate

| Component | Estimated Cost (SEK) |
|---|---|
| Server room build-out (if new) | 2,000,000 - 5,000,000 |
| TEMPEST shielding (if Zone A) | 3,000,000 - 8,000,000 |
| Access control and intrusion detection | 500,000 - 1,000,000 |
| Fire suppression | 300,000 - 600,000 |
| Generator and electrical | 500,000 - 1,500,000 |
| **Facility subtotal** | **6,300,000 - 16,100,000** |

### 18.3 Software Costs

| Component | Cost |
|---|---|
| RKE2 | Free (open source) |
| Cilium | Free (open source); enterprise support ~500,000 SEK/year |
| Ceph | Free (open source) |
| FreeIPA | Free (open source) |
| Keycloak | Free (open source) |
| Rocky Linux | Free (open source) |
| RHEL (if chosen) | ~3,000 SEK/node/year = ~78,000 SEK/year |
| Vault | Free (open source, BSL license applies to newer versions; use last MPL version or accept BSL) |
| SUSE/Rancher support | Optional, ~1,000,000 SEK/year for enterprise support |
| **Software subtotal** | **0 - 1,500,000 SEK/year** |

### 18.4 Operational Costs

| Component | Annual Cost (SEK) |
|---|---|
| 12 cleared engineers (loaded cost, estimated) | 12,000,000 - 18,000,000 |
| Power and cooling | 500,000 - 800,000 |
| Facility maintenance | 200,000 - 500,000 |
| Hardware maintenance/spare parts | 500,000 - 1,000,000 |
| Training and certifications | 300,000 - 600,000 |
| **Operational subtotal** | **~13,500,000 - 20,900,000 SEK/year** |

### 18.5 Total Cost of Ownership (5-year)

| Category | 5-Year Cost (SEK) |
|---|---|
| Hardware (initial + refresh at year 4) | ~20,000,000 |
| Facility (one-time + maintenance) | ~10,000,000 - 20,000,000 |
| Software licenses | 0 - 7,500,000 |
| Operations (personnel, power, maintenance) | ~67,500,000 - 104,500,000 |
| **5-year total** | **~97,500,000 - 152,000,000 SEK** |

> **Note**: The dominant cost is personnel. The FLOSS stack saves significantly on software licensing (versus VMware + proprietary alternatives which could add 5-10 MSEK/year), but the cleared personnel constraint is the binding cost driver.

---

## 19. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Key person dependency (12 engineers) | High | Critical | Cross-training, documentation, runbook automation |
| Accreditation delay/denial | Medium | Critical | Early FMV/MUST engagement, design-for-accreditation approach |
| Hardware supply chain delay | Medium | High | Order early, maintain spare inventory, pre-approved vendor list |
| Ceph data loss (triple node failure) | Very Low | Critical | 3x replication, offsite backups, monitoring for degraded state |
| Air-gap transfer process becomes bottleneck | Medium | Medium | Streamline process, dedicated transfer tooling, clear SLAs |
| Security incident within air-gapped environment | Low | Critical | Runtime monitoring (Falco/Tetragon), incident response plan, forensic capability |
| Kubernetes upgrade incompatibility | Low | Medium | Test in unclassified mirror, staged rollout, rollback procedure |
| TEMPEST assessment requires facility changes | Medium | High | Engage MUST TEMPEST assessors during design phase, not after build |
| FRA rejects cryptographic implementation | Medium | High | Engage FRA for signalskydd consultation during design phase |
| Loss of security clearance for key engineer | Medium | High | Maintain clearance pipeline, cross-train roles, succession planning |
| Power failure beyond UPS/generator capacity | Very Low | High | Generator testing, fuel supply agreement, graceful shutdown procedures |

---

## 20. Architectural Decision Records

### ADR-001: Kubernetes Distribution

**Decision**: RKE2 (Rancher Government)
**Context**: Need a Kubernetes distribution suitable for air-gapped HEMLIG classified environment with CIS hardening and FIPS crypto.
**Alternatives**: OpenShift (too heavy, licensing cost), Talos (too new for defense), kubeadm (too manual), k3s (not hardened enough).
**Consequences**: SUSE/Rancher dependency for enterprise support; well-suited for air-gapped deployment.

### ADR-002: Container Runtime

**Decision**: containerd (bundled with RKE2)
**Context**: CRI-compliant runtime required.
**Consequences**: Standard runtime, well-supported, CIS-benchmarkable.

### ADR-003: CNI Plugin

**Decision**: Cilium (replacing default Canal)
**Context**: Need advanced network policy (L3/L4/L7), encryption, and network observability.
**Alternatives**: Calico (good but less observability), Canal (default, limited L7), Multus only (for secondary networks with KubeVirt).
**Consequences**: eBPF dependency (kernel 5.10+ required, met by Rocky 9); strong observability via Hubble.

### ADR-004: Storage Platform

**Decision**: Rook-Ceph
**Context**: Need software-defined storage providing block, file, and object storage for Kubernetes.
**Alternatives**: Longhorn (simpler but less capable), OpenEBS (less mature for production), proprietary SAN (expensive, vendor lock-in).
**Consequences**: Operational complexity of Ceph mitigated by Rook operator; 5-node minimum for reliability.

### ADR-005: Identity Provider

**Decision**: FreeIPA + Keycloak
**Context**: Need centralized identity management for all platform components with MFA support.
**Alternatives**: Active Directory (licensing cost, Windows dependency), direct LDAP (no OIDC), Authentik (less mature).
**Consequences**: FLOSS, no licensing; requires FreeIPA expertise on team.

### ADR-006: GitOps Engine

**Decision**: ArgoCD
**Context**: Need declarative, Git-based deployment for all Kubernetes resources.
**Alternatives**: Flux CD (lighter, but less UI for the small team to visually verify), manual kubectl (unacceptable for classified environment).
**Consequences**: ArgoCD UI provides visual deployment status useful for audit and verification.

### ADR-007: Secrets Management

**Decision**: HashiCorp Vault
**Context**: Need centralized secrets management with dynamic secrets, PKI, and comprehensive audit.
**Alternatives**: Sealed Secrets (simpler but less capable), SOPS (file-level only), CyberArk (expensive).
**Consequences**: Vault BSL license; acceptable for internal deployment. Audit logging meets HEMLIG requirements.

### ADR-008: Network Fabric

**Decision**: Cisco Nexus 9000 spine-leaf with VXLAN-EVPN
**Context**: Need reliable, segmented network fabric for classified environment.
**Alternatives**: Juniper QFX (acceptable but less common in Swedish defense), Arista (less Swedish defense presence), white-box (insufficient support for classified).
**Consequences**: Cisco SmartNet cost; proven in Swedish defense environments.

### ADR-009: Operating System

**Decision**: Rocky Linux 9 (or RHEL 9 with vendor support)
**Context**: Need stable, long-term supported Linux distribution with CIS/STIG hardening profiles available.
**Alternatives**: Ubuntu (viable but RPM ecosystem more common in defense), Debian (less enterprise tooling), SLES (licensing).
**Consequences**: Community-supported; RHEL upgrade path available if required.

### ADR-010: FLOSS-First Approach

**Decision**: Prefer FLOSS for all software components where equivalent or better functionality exists.
**Context**: Minimize licensing costs and vendor lock-in while maintaining capability.
**Exceptions**: Cisco networking (justified by defense supply chain), NVIDIA GPU drivers (proprietary required).
**Consequences**: ~1-2 MSEK/year saved versus proprietary alternatives; requires deeper team expertise.

---

## Appendix A: Key Contacts and Authorities

| Role | Organization | Purpose |
|---|---|---|
| FMV | Forsvarets materielverk | Accreditation authority for defense systems |
| MUST | Militara underrattelse- och sakerhetstjansten | Military security authority |
| FRA | Forsvarets radioanstalt | Signals intelligence, cryptographic approval |
| Sapo | Sakerhetspolisen | Personnel security clearance vetting |
| MSB | Myndigheten for samhallsskydd och beredskap | Civil protection guidance |

## Appendix B: Glossary

| Swedish Term | English Translation | Context |
|---|---|---|
| Sakerhetsskyddslagen | Protective Security Act | Primary legislation |
| Sakerhetsskyddsanalys | Protective Security Analysis | Required threat/risk analysis |
| Sakerhetsskyddschef | Chief Security Officer | Required role in organization |
| Signalskydd | Signals protection / Crypto | FRA-regulated cryptographic protection |
| Sakerhetsproving | Security clearance vetting | Sapo process |
| HEMLIG | Secret | Classification level |
| KVALIFICERAT HEMLIG | Top Secret | Higher classification level |
| BEGRANSAT HEMLIG | Restricted/Confidential | Lower classification level |
| Skyddsobjekt | Protected facility | Legally designated protected site |
| Forsvarsindustri | Defense industry | Sector designation |
| Tva-personers-regel | Four-eyes principle | Dual-control requirement |

## Appendix C: Compliance Mapping

| Sakerhetsskyddslagen Requirement | Architecture Control |
|---|---|
| Ch. 2 - Sakerhetsskyddsanalys | Documented in Section 3.1 |
| Ch. 3 - Personsakerhet | All 12 operators Sapo-cleared (Section 14) |
| Ch. 4 - Fysiskt sakerhetsskydd | Facility design (Section 4) |
| Ch. 5 - Informationssakerhet | Encryption, access control, audit (Sections 8, 9, 11, 12) |
| Ch. 5 - Signalskydd | FRA-approved crypto, TLS everywhere, LUKS (Sections 5, 8, 9.4) |
| Ch. 6 - Sakerhetsskyddad upphandling | Supply chain controls (Section 15) |
| Data sovereignty | All infrastructure in Sweden, air-gapped (Sections 4, 5, 10) |

---

*This document should be treated as HEMLIG upon population with site-specific details. The template itself, without specific network addresses, facility locations, or personnel information, may be handled at a lower classification.*
