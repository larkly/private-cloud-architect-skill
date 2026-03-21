# VMware vSphere to KubeVirt Migration Architecture
## IL5 Air-Gapped Environment -- DoD Contract

**Document Classification:** CUI
**Version:** 1.0
**Date:** 2026-03-20
**Author:** Private Cloud Architecture Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Assessment](#2-current-state-assessment)
3. [Target Architecture Decision](#3-target-architecture-decision)
4. [Target Architecture Design](#4-target-architecture-design)
5. [Network Architecture](#5-network-architecture)
6. [Storage Architecture](#6-storage-architecture)
7. [Security and Compliance Architecture](#7-security-and-compliance-architecture)
8. [Air-Gapped Operations Model](#8-air-gapped-operations-model)
9. [Migration Strategy](#9-migration-strategy)
10. [ATO Maintenance Plan](#10-ato-maintenance-plan)
11. [Disaster Recovery](#11-disaster-recovery)
12. [Operational Runbook Framework](#12-operational-runbook-framework)
13. [Risk Register](#13-risk-register)
14. [Cost Analysis](#14-cost-analysis)
15. [Timeline and Milestones](#15-timeline-and-milestones)
16. [Architectural Decision Records](#16-architectural-decision-records)

---

## 1. Executive Summary

### Problem Statement

The current environment operates 200 virtual machines on VMware vSphere 7 within an IL5-rated, fully air-gapped enclave. Following the Broadcom acquisition of VMware, licensing costs have escalated significantly, renewal terms have become unpredictable, and the long-term viability of vSphere as a cost-effective platform for DoD workloads is in question. The organization requires a migration to an alternative virtualization platform that:

- Eliminates VMware/Broadcom licensing dependency
- Maintains IL5 accreditation and the existing Authority to Operate (ATO)
- Operates entirely within an air-gapped network with zero internet connectivity
- Provides a path toward cloud-native modernization without forcing application rewrites
- Reduces total cost of ownership over a 5-year horizon

### Recommended Solution

**KubeVirt on RKE2 Government**, deployed as a converged VM and container platform, backed by Rook-Ceph for software-defined storage, Cilium for networking, and managed through the DoD Platform One / Big Bang reference architecture for security tooling. This approach provides:

- **Zero hypervisor licensing costs** -- KubeVirt is CNCF-incubating, Apache 2.0 licensed
- **FIPS 140-2 validated Kubernetes** via RKE2 Government
- **CIS-hardened by default** with STIG compliance tooling
- **Converged platform** running both legacy VMs and future container workloads
- **Alignment with DoD modernization direction** (Platform One, Big Bang, Iron Bank)

### Alternatives Considered

| Platform | Pros | Cons | Verdict |
|----------|------|------|---------|
| **KubeVirt on RKE2 Gov** | FIPS validated, zero license cost, DoD ecosystem alignment, converged VM+container | Operational learning curve, newer platform | **Selected** |
| **Proxmox VE** | Mature VM management, simple operations, FLOSS | No FIPS validation, no DoD ecosystem alignment, limited STIG coverage, no container orchestration | Rejected |
| **OpenStack (Kolla-Ansible)** | Full IaaS, mature VM lifecycle | Operational complexity, large team required, no container convergence | Rejected |
| **Nutanix AHV** | Turnkey HCI, good DoD presence | Still proprietary licensing (defeats purpose), cost | Rejected |
| **Harvester (SUSE)** | KubeVirt-based HCI, simpler UX | Less DoD traction than RKE2 Gov, fewer STIG resources | Considered as fallback |

---

## 2. Current State Assessment

### Existing VMware Environment

| Attribute | Current State |
|-----------|--------------|
| Hypervisor | VMware vSphere 7.x (ESXi) |
| Management | vCenter Server Appliance |
| VM Count | ~200 |
| Classification | IL5 |
| Network Connectivity | Fully air-gapped, no internet |
| ATO Status | Active, must be maintained |

### Assumed Workload Profile

Based on typical DoD IL5 environments of this scale:

| Category | Estimated Count | Characteristics |
|----------|----------------|-----------------|
| Windows Server VMs | 60-80 | AD, file servers, MSSQL, custom .NET apps |
| RHEL/CentOS VMs | 80-100 | Java apps, middleware, databases (PostgreSQL, MariaDB), web servers |
| Appliance VMs | 15-25 | Security tools (Nessus/ACAS, HBSS/Trellix, Splunk forwarders), SIEM, log collectors |
| Infrastructure VMs | 15-20 | DNS, DHCP, NTP, LDAP/IPA, jump boxes, Ansible/AWX |
| Database VMs | 10-15 | Oracle, PostgreSQL, MSSQL clusters |

### Assumed Hardware Profile

| Component | Assumed Spec |
|-----------|-------------|
| Compute Hosts | 10-16 servers, dual-socket Intel Xeon (Icelake/Sapphire Rapids), 512GB-1TB RAM each |
| Storage | SAN (Fibre Channel or iSCSI) or vSAN with SSD/NVMe tiers |
| Networking | 10/25GbE data, 1GbE management, likely Cisco Nexus or similar |
| Total vCPU | ~1,600-2,400 vCPUs allocated |
| Total RAM | ~4-8 TB allocated |
| Total Storage | ~100-200 TB provisioned |

> **Action Required:** Validate these assumptions with a detailed discovery scan (RVTools export from vCenter, or manual inventory) before finalizing hardware sizing for the target platform.

---

## 3. Target Architecture Decision

### ADR-001: KubeVirt on RKE2 Government as VMware Replacement

**Status:** Accepted

**Context:** Need to replace VMware vSphere 7 in an IL5 air-gapped environment for 200 VMs while maintaining ATO and eliminating Broadcom licensing dependency.

**Decision:** Deploy KubeVirt on RKE2 Government as the target virtualization and container platform.

**Rationale:**

1. **FIPS 140-2 Validation:** RKE2 Government ships with FIPS-validated cryptographic modules. This is non-negotiable for IL5.
2. **CIS Hardened by Default:** RKE2 passes CIS Kubernetes Benchmark out of the box. Reduces STIG remediation burden.
3. **DoD Ecosystem Alignment:** Direct compatibility with Platform One Big Bang, Iron Bank container registry, and Rancher Government Solutions support contracts.
4. **Zero Hypervisor Licensing:** KubeVirt is Apache 2.0 FLOSS. RKE2 is Apache 2.0. The entire virtualization stack has zero licensing cost.
5. **Converged Platform:** Running VMs and containers on the same cluster provides a modernization path -- teams can containerize workloads incrementally without a second platform migration.
6. **Air-Gap Native:** RKE2 is designed for air-gapped deployment. Rancher provides tooling for disconnected registry mirroring and offline Helm chart bundles.
7. **Government Support:** Rancher Government Solutions holds IL5 credentials and provides dedicated government support channels with cleared personnel.

**Consequences:**
- Operations team must be trained on Kubernetes concepts, even though VMs will look similar to traditional VMs from a workload perspective.
- Some VMware-specific features (e.g., DRS, vMotion affinity rules) must be mapped to Kubernetes equivalents (pod affinity/anti-affinity, node selectors, live migration).
- Initial operational complexity is higher than a like-for-like hypervisor replacement (e.g., Proxmox), but the long-term platform value is significantly greater.

---

## 4. Target Architecture Design

### 4.1 High-Level Architecture

```
+------------------------------------------------------------------+
|                     IL5 AIR-GAPPED ENCLAVE                       |
|                                                                  |
|  +---------------------------+  +-----------------------------+  |
|  |   MANAGEMENT CLUSTER      |  |   WORKLOAD CLUSTER          |  |
|  |   (RKE2 Gov - 3 nodes)    |  |   (RKE2 Gov - 12+ nodes)   |  |
|  |                            |  |                             |  |
|  |  - Rancher MCM             |  |  - KubeVirt VMs (200)       |  |
|  |  - Harbor Registry         |  |  - Container workloads      |  |
|  |  - GitLab (GitOps)         |  |  - Rook-Ceph storage        |  |
|  |  - Vault (secrets)         |  |  - Cilium CNI + policies    |  |
|  |  - AWX (Ansible)           |  |  - MetalLB load balancing   |  |
|  |  - Monitoring stack        |  |  - Multus (multi-network)   |  |
|  +---------------------------+  +-----------------------------+  |
|                                                                  |
|  +---------------------------+  +-----------------------------+  |
|  |   STORAGE (Ceph)          |  |   SECURITY / COMPLIANCE     |  |
|  |                            |  |                             |  |
|  |  - 3+ OSD nodes (or       |  |  - ACAS/Nessus scanning     |  |
|  |    converged with compute) |  |  - HBSS/Trellix endpoints   |  |
|  |  - NVMe + SSD tiers        |  |  - Splunk/Elastic SIEM     |  |
|  |  - RBD for VM disks        |  |  - OpenSCAP/Compliance Op  |  |
|  |  - CephFS for shared       |  |  - Falco runtime security  |  |
|  |  - RGW for object store    |  |  - Kyverno policy engine   |  |
|  +---------------------------+  +-----------------------------+  |
|                                                                  |
+------------------------------------------------------------------+
```

### 4.2 Cluster Topology

#### Management Cluster (3 nodes)

Purpose: Rancher multi-cluster management, CI/CD, monitoring, registry, secrets.

| Role | Count | Spec (Minimum) |
|------|-------|-----------------|
| Control plane + etcd + worker | 3 | 16 vCPU, 64GB RAM, 500GB NVMe OS, 1TB SSD data |

Services hosted:
- **Rancher Server** -- multi-cluster management UI and API
- **Harbor** -- air-gapped container and Helm chart registry (mirrored from Iron Bank)
- **GitLab** (or Gitea) -- Git repository for GitOps, IaC, and configuration
- **HashiCorp Vault** -- secrets management (FIPS-enabled)
- **AWX** -- Ansible automation for day-2 VM configuration
- **Prometheus + Grafana + Alertmanager + Loki** -- full observability stack
- **Velero** -- backup orchestration for Kubernetes resources

#### Workload Cluster (12-16 nodes)

Purpose: Run the 200 migrated VMs via KubeVirt, plus any containerized workloads.

| Role | Count | Spec (Minimum) |
|------|-------|-----------------|
| Control plane + etcd | 3 | 8 vCPU, 32GB RAM, 500GB NVMe |
| Worker (compute + storage converged) | 9-13 | 64-128 vCPU, 512GB-1TB RAM, 2x NVMe (OS + Ceph journal), 4-8x SSD/NVMe (Ceph OSD) |

> **Sizing Note:** With 200 VMs averaging 8 vCPU and 32GB RAM each, total demand is approximately 1,600 vCPU and 6.4TB RAM. With a 1.5:1 overcommit ratio on CPU (conservative for IL5 with SLA requirements) and 1:1 on RAM, 12 worker nodes at 128 vCPU / 1TB RAM provides adequate capacity with N+2 redundancy.

#### Node Layout for Converged Storage

Each worker node contributes both compute and Ceph OSD capacity:

```
Worker Node (example: Dell PowerEdge R760 or HPE ProLiant DL380 Gen11)
+-------------------------------------------------------+
| 2x Intel Xeon Gold 6448Y (64 cores total)             |
| 1TB DDR5 ECC RAM (16x 64GB DIMMs)                     |
| 2x 1.6TB NVMe (RAID1 for OS + Ceph WAL/DB)           |
| 6x 7.68TB NVMe SSD (Ceph OSD -- ~46TB raw per node)  |
| 2x 25GbE (Ceph public + cluster network)              |
| 2x 25GbE (VM/pod data network)                        |
| 1x 1GbE (IPMI/management)                             |
+-------------------------------------------------------+
```

Total raw Ceph capacity (12 nodes): ~552TB raw, ~184TB usable at 3x replication (or ~276TB usable at erasure coding 4+2).

### 4.3 Software Stack

| Layer | Component | License | Notes |
|-------|-----------|---------|-------|
| OS | RHEL 8/9 or Rocky Linux 8/9 (STIG hardened) | RHEL subscription or FLOSS (Rocky) | FIPS mode enabled, DISA STIG applied |
| Kubernetes | RKE2 Government | Apache 2.0 | FIPS 140-2 validated, CIS hardened |
| VM Runtime | KubeVirt | Apache 2.0 (CNCF) | VM lifecycle, live migration, CDI |
| Container Runtime | containerd (bundled with RKE2) | Apache 2.0 | FIPS-compatible build |
| CNI | Cilium | Apache 2.0 | eBPF-based, network policy, observability |
| Multi-Network | Multus CNI | Apache 2.0 | Attach VMs to multiple networks, SR-IOV |
| Load Balancer | MetalLB | Apache 2.0 | L2/BGP mode for bare-metal LB |
| Storage | Rook-Ceph | Apache 2.0 | RBD for VM disks, CephFS for shared, RGW for S3 |
| Registry | Harbor | Apache 2.0 | Air-gapped image/chart mirror from Iron Bank |
| GitOps | ArgoCD (or Flux) | Apache 2.0 | Declarative cluster and app management |
| Secrets | HashiCorp Vault | BSL (pre-fork binaries or OpenBao) | FIPS-enabled, auto-unseal via HSM |
| Monitoring | Prometheus + Grafana + Loki | Apache 2.0 / AGPL | Full metrics, logs, dashboards |
| Security | Falco, Kyverno, OpenSCAP | Apache 2.0 / various | Runtime detection, policy enforcement, STIG scanning |
| Backup | Velero + Volsync | Apache 2.0 | K8s resource backup, PV replication |
| Cluster Mgmt | Rancher | Apache 2.0 | Multi-cluster UI, RBAC, catalogs |
| Automation | AWX + Ansible | GPL / Apache 2.0 | Day-2 VM config, patching, compliance |

> **Note on Vault licensing:** HashiCorp moved Vault to BSL in 2023. For strict FLOSS compliance, consider **OpenBao** (the community fork under Linux Foundation). For pragmatic government use, Vault Enterprise with HSM support is common in DoD environments. Evaluate based on procurement constraints.

---

## 5. Network Architecture

### 5.1 Network Segmentation

IL5 requires strict network segmentation. The following network zones are defined:

| Network | VLAN/Subnet | Purpose | Interface |
|---------|-------------|---------|-----------|
| Management | 10.10.1.0/24 | IPMI/iLO/iDRAC, out-of-band management | 1GbE dedicated |
| Cluster API | 10.10.10.0/24 | Kubernetes API, etcd, Rancher | 25GbE bond (LACP) |
| Ceph Public | 10.10.20.0/24 | Ceph client traffic (RBD, CephFS, RGW) | 25GbE dedicated |
| Ceph Cluster | 10.10.21.0/24 | Ceph OSD replication (backend only) | 25GbE dedicated |
| VM Data (Tenant) | 10.10.100.0/20 | VM-to-VM and VM-to-service traffic | 25GbE bond (LACP) |
| VM Legacy VLANs | Various | Existing VLANs bridged for migrated VMs | Multus bridge/VLAN |
| Services | 10.10.50.0/24 | MetalLB pool, ingress, service endpoints | Shared with VM Data |

### 5.2 Cilium CNI Configuration

Cilium provides the primary CNI for the Kubernetes cluster:

- **eBPF dataplane** -- replaces kube-proxy, provides high-performance forwarding
- **Network Policies** -- Kubernetes NetworkPolicy and CiliumNetworkPolicy for L3/L4/L7 microsegmentation
- **Hubble** -- network observability (flow logs, service map) for security monitoring
- **Encryption** -- WireGuard or IPsec node-to-node encryption (FIPS-compliant IPsec mode)
- **BGP** -- native BGP peering with ToR switches for service advertisement

### 5.3 Multus for Legacy VM Networking

Many migrated VMs will need to retain their existing network identities (IPs, VLANs, firewall rules) to minimize ATO impact. Multus CNI enables:

- **Bridge CNI** -- attach VM interfaces directly to host bridges (mapped to physical VLANs)
- **VLAN CNI** -- create VLAN sub-interfaces on host NICs
- **SR-IOV** -- hardware-accelerated NIC virtualization for performance-sensitive VMs (database servers, high-throughput apps)
- **macvlan/ipvlan** -- lightweight L2/L3 network attachment

This allows VMs to appear on their existing network segments with their existing IPs, making the migration transparent to upstream firewalls, IDS/IPS, and network security monitoring tools.

### 5.4 DNS and Service Discovery

| Service | Technology | Notes |
|---------|-----------|-------|
| Internal K8s DNS | CoreDNS (bundled with RKE2) | Pod/service resolution |
| Enclave DNS | Existing DNS infrastructure or PowerDNS | Authoritative for enclave zones |
| Split-horizon | CoreDNS forward plugin | Forward non-cluster queries to enclave DNS |

### 5.5 Network Policy Enforcement

```yaml
# Example: Isolate database VMs to only accept traffic from app tier
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: db-tier-isolation
  namespace: workload-databases
spec:
  endpointSelector:
    matchLabels:
      tier: database
  ingress:
    - fromEndpoints:
        - matchLabels:
            tier: application
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
            - port: "1433"
              protocol: TCP
```

---

## 6. Storage Architecture

### 6.1 Rook-Ceph Deployment

Ceph provides unified storage for the entire platform:

| Storage Type | Ceph Backend | Use Case |
|-------------|-------------|----------|
| Block (RBD) | Replicated pool (3x) | VM boot disks, database volumes, high-IOPS workloads |
| Block (RBD) | Erasure coded pool (4+2) | VM data disks, bulk storage, lower-IOPS workloads |
| Filesystem (CephFS) | Replicated MDS + data pools | Shared filesystems (replaces NFS), multi-attach |
| Object (RGW) | Erasure coded pool | S3-compatible object storage, backups, artifacts |

### 6.2 Storage Classes

```yaml
# High-performance: 3x replicated on NVMe, for databases and latency-sensitive VMs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-nvme-replicated
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  pool: nvme-replicated
  clusterID: rook-ceph
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true

# Standard: Erasure coded on SSD, for general VM workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-ssd-ec
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  pool: ssd-ec-42
  dataPool: ssd-ec-42-data
  clusterID: rook-ceph
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### 6.3 Storage Sizing

| Tier | Raw Capacity | Usable Capacity | Use Case |
|------|-------------|-----------------|----------|
| NVMe Replicated (3x) | 120 TB | 40 TB | Databases, high-IOPS VMs |
| SSD Erasure Coded (4+2) | 432 TB | 288 TB | General VM disks, bulk data |
| CephFS (replicated) | Shared with SSD pool | ~20 TB allocated | Shared filesystems |
| RGW Object | Shared with EC pool | ~50 TB allocated | Backups, artifacts, logs |

### 6.4 VM Disk Migration

VMware VMDK disks will be converted during migration:

```
VMDK (vSphere) --> qcow2 (intermediate) --> raw/qcow2 (KubeVirt CDI import)
```

The Containerized Data Importer (CDI) handles importing disk images into PersistentVolumeClaims:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: rhel-vm-001-disk
  namespace: workload-linux
spec:
  source:
    upload: {}  # or registry, http (from local artifact server)
  storage:
    storageClassName: ceph-rbd-ssd-ec
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Gi
```

---

## 7. Security and Compliance Architecture

### 7.1 IL5 Security Controls

IL5 (Impact Level 5) is required for Controlled Unclassified Information (CUI) and National Security Systems (NSS) data. Key requirements:

| Control Family | Implementation |
|---------------|----------------|
| **FIPS 140-2 Cryptography** | RKE2 Government FIPS mode, RHEL FIPS mode, Ceph encryption at rest (LUKS), TLS everywhere |
| **Access Control (AC)** | Rancher RBAC mapped to AD/LDAP groups, Kubernetes RBAC, namespace isolation |
| **Audit and Accountability (AU)** | Kubernetes audit logging, Falco syscall monitoring, all logs to SIEM (Splunk/Elastic) |
| **Configuration Management (CM)** | GitOps (ArgoCD), STIG-hardened base images, Kyverno policy enforcement, drift detection |
| **Identification and Authentication (IA)** | CAC/PIV via Keycloak or AD FS, MFA for all admin access, service account rotation |
| **Incident Response (IR)** | Falco alerts, Cilium Hubble flow analysis, SIEM correlation, automated quarantine via Kyverno |
| **System and Communications Protection (SC)** | Cilium network policies, namespace isolation, encrypted Ceph traffic, WireGuard/IPsec pod-to-pod |
| **System and Information Integrity (SI)** | ACAS/Nessus scanning, OpenSCAP STIG validation, container image scanning (Trivy in Harbor), SBOM generation |

### 7.2 STIG Compliance

| Component | STIG Source | Automation |
|-----------|-------------|------------|
| RHEL 8/9 OS | DISA RHEL STIG | OpenSCAP + Ansible remediation playbooks |
| Kubernetes | DISA Kubernetes STIG | RKE2 Gov ships CIS-hardened; residual items via Kyverno |
| Container Images | DISA Container STIG | Iron Bank hardened images; Trivy scanning in Harbor |
| Windows VMs | DISA Windows Server STIG | SCAP Compliance Checker, GPO enforcement |

### 7.3 Security Stack (Big Bang Aligned)

Platform One Big Bang provides a reference architecture for DoD Kubernetes security. The following components align with Big Bang:

| Big Bang Component | Our Implementation | Purpose |
|-------------------|-------------------|---------|
| Istio | Cilium service mesh (alternative) | mTLS, traffic management |
| Kiali | Hubble UI | Service observability |
| Jaeger | Grafana Tempo | Distributed tracing |
| Kyverno | Kyverno | Policy enforcement |
| Monitoring | Prometheus + Grafana | Metrics and dashboards |
| Logging | Loki (or EFK) | Log aggregation |
| Twistlock/Prisma | Falco + Trivy | Runtime and image security |
| Keycloak | Keycloak | SSO/SAML/OIDC for all platform services |
| Vault | Vault or OpenBao | Secrets management |
| Anchore/Grype | Trivy (in Harbor) | Container vulnerability scanning |
| ArgoCD | ArgoCD | GitOps deployment |
| Harbor | Harbor | Registry with Iron Bank mirror |

> **Note:** Full Big Bang deployment is an option if the organization wants exact alignment with the Platform One reference. The components listed above achieve equivalent security posture with potentially simpler operations.

### 7.4 Encryption Architecture

| Data State | Mechanism | FIPS Compliance |
|-----------|-----------|-----------------|
| Data at rest (VM disks) | Ceph RBD encryption (LUKS2 via dm-crypt) | FIPS 140-2 (kernel crypto module) |
| Data at rest (etcd) | RKE2 etcd encryption at rest | FIPS 140-2 (Go BoringCrypto) |
| Data at rest (OS disks) | LUKS2 full-disk encryption on all nodes | FIPS 140-2 (kernel crypto module) |
| Data in transit (pod-to-pod) | Cilium WireGuard or IPsec | IPsec mode is FIPS-compliant |
| Data in transit (Ceph) | Ceph messenger v2 with encryption | TLS 1.3 with FIPS-validated OpenSSL |
| Data in transit (API) | TLS 1.2/1.3 on all endpoints | FIPS-validated TLS libraries |
| Secrets at rest | Vault with HSM backend (or auto-unseal) | FIPS 140-2 Level 3 (HSM) |

### 7.5 Image Supply Chain Security

In an air-gapped environment, container image provenance is critical:

1. **Source:** Iron Bank (hardened, DISA-approved container images)
2. **Transfer:** Images exported on approved removable media via data transfer process (see Section 8)
3. **Validation:** Signature verification using cosign/Sigstore with Iron Bank signing keys
4. **Storage:** Harbor registry with vulnerability scanning (Trivy) and SBOM storage
5. **Enforcement:** Kyverno policy requires all images to originate from local Harbor and pass vulnerability thresholds
6. **Audit:** Full chain-of-custody documentation for all imported artifacts

```yaml
# Kyverno policy: Only allow images from internal Harbor
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must come from the internal Harbor registry."
        pattern:
          spec:
            containers:
              - image: "harbor.enclave.local/*"
            initContainers:
              - image: "harbor.enclave.local/*"
```

---

## 8. Air-Gapped Operations Model

### 8.1 Artifact Transfer Process

All software, updates, container images, and configuration must enter the air-gapped enclave through an approved data transfer process.

```
+-------------------+     +------------------+     +-------------------+
|  UNCLASS STAGING  | --> |  TRANSFER MEDIA  | --> |  IL5 ENCLAVE      |
|  (Internet-conn)  |     |  (Encrypted USB/ |     |  (Air-gapped)     |
|                   |     |   Data Diode)    |     |                   |
|  - Pull Iron Bank |     |  - AV scanned    |     |  - Verify sigs    |
|    images         |     |  - Checksummed   |     |  - Import to      |
|  - Pull Helm      |     |  - Chain of      |     |    Harbor          |
|    charts         |     |    custody log   |     |  - Import to      |
|  - Pull RPM repos |     |  - Approved by   |     |    local repos    |
|  - Pull security  |     |    security      |     |  - Scan & validate|
|    feeds (CVE)    |     |    officer       |     |  - Deploy via     |
+-------------------+     +------------------+     |    GitOps         |
                                                    +-------------------+
```

### 8.2 Disconnected Registry (Harbor)

Harbor operates as the single source of truth for all container images and Helm charts within the enclave:

- **Image Mirroring:** Periodic bulk import from Iron Bank via transfer media
- **Helm Charts:** All charts stored in Harbor's chart museum
- **Vulnerability Database:** Trivy DB updated via offline bundle transfer
- **Replication:** Harbor can replicate between management and workload clusters if needed
- **Garbage Collection:** Automated cleanup of old image tags per retention policy

### 8.3 Disconnected Package Repository

For OS-level packages (RPM):

| OS | Repository Tool | Content |
|----|----------------|---------|
| RHEL | Local Satellite or Pulp | RHEL BaseOS, AppStream, EPEL subset, custom RPMs |
| Rocky Linux | Pulp or createrepo | Rocky BaseOS, AppStream, EPEL subset |
| Windows | WSUS | Windows Updates, approved patches |

### 8.4 Offline Kubernetes Operations

| Operation | Offline Method |
|-----------|---------------|
| RKE2 install/upgrade | `rke2-images.linux-amd64.tar.zst` bundle, air-gap install script |
| Helm chart install | Charts stored in Harbor OCI registry or local chartmuseum |
| ArgoCD sync | Git repo hosted on internal GitLab, all images pre-staged in Harbor |
| Ansible execution | Collections bundled in tar, Galaxy local mirror, all roles in Git |
| CVE/vulnerability feeds | Trivy offline DB, ACAS plugin updates via transfer media |

### 8.5 Patch Management Cadence

| Category | Frequency | Method |
|----------|-----------|--------|
| Critical CVEs | Within 72 hours of notification | Emergency transfer, tested in staging, rolled to prod |
| Monthly OS patches | Monthly | Bulk RPM import, Ansible-driven rolling update |
| Kubernetes upgrades | Quarterly | RKE2 air-gap bundle, staged rollout (mgmt cluster first, then workload) |
| KubeVirt upgrades | Quarterly (aligned with K8s) | Helm chart update via Harbor, rolling VM live migration |
| Container image updates | Monthly (aligned with Iron Bank releases) | Bulk image import, ArgoCD auto-sync |

---

## 9. Migration Strategy

### 9.1 Migration Approach: Phased Waves

The 200 VMs will be migrated in 5 waves over approximately 16-20 weeks of active migration (following an 8-12 week platform build phase).

| Wave | VMs | Category | Risk | Duration |
|------|-----|----------|------|----------|
| Wave 0 (Pilot) | 5-10 | Non-critical Linux infra (dev/test, jump boxes) | Low | 2 weeks |
| Wave 1 | 30-40 | Linux infrastructure (DNS, NTP, monitoring, Ansible) | Low-Medium | 3 weeks |
| Wave 2 | 50-60 | Linux application servers (stateless/low-state) | Medium | 4 weeks |
| Wave 3 | 50-60 | Windows servers, stateful Linux (databases) | High | 4-5 weeks |
| Wave 4 | 40-50 | Security appliances, remaining complex workloads | High | 3-4 weeks |

### 9.2 Per-VM Migration Process

```
Phase 1: Assessment
  |-- Inventory VM (CPU, RAM, disk, network, OS, apps)
  |-- Document network dependencies (IPs, VLANs, firewall rules, DNS entries)
  |-- Document storage dependencies (disk sizes, IOPS requirements, shared storage)
  |-- Identify application dependencies and startup order
  |-- Classify migration complexity (simple/moderate/complex)

Phase 2: Preparation
  |-- Export VMDK from vSphere (ovftool or vCenter export)
  |-- Convert VMDK to qcow2: qemu-img convert -f vmdk -O qcow2 vm-disk.vmdk vm-disk.qcow2
  |-- Transfer qcow2 to local artifact server within enclave
  |-- Create KubeVirt VirtualMachine manifest (CPU, RAM, networks, disks)
  |-- Create DataVolume for CDI import
  |-- Create NetworkAttachmentDefinition for legacy VLAN connectivity (Multus)
  |-- Prepare Ansible post-migration playbook (install qemu-guest-agent, update configs)

Phase 3: Migration Execution
  |-- Upload disk image via CDI (virtctl image-upload or DataVolume with HTTP source)
  |-- Apply VirtualMachine manifest
  |-- Start VM, verify boot
  |-- Validate network connectivity (ping, traceroute, service-level checks)
  |-- Run application-level smoke tests
  |-- Update DNS if IP changed (or retain IP via Multus bridge)
  |-- Validate STIG compliance scan on migrated VM

Phase 4: Validation & Cutover
  |-- Run full application test suite
  |-- Verify monitoring and alerting (Prometheus scrape targets, SIEM log flow)
  |-- Verify backup (Velero snapshot of VM PVC)
  |-- Cutover production traffic
  |-- Monitor for 48-72 hours
  |-- Decommission source VM on vSphere
  |-- Update CMDB/inventory
```

### 9.3 KubeVirt VM Manifest Example

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel-app-server-001
  namespace: workload-linux-apps
  labels:
    app: myapp
    tier: application
    migration-wave: "2"
    original-vsphere-name: "rhel-app-001"
spec:
  running: true
  template:
    metadata:
      labels:
        app: myapp
        tier: application
        kubevirt.io/domain: rhel-app-server-001
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
          dedicatedCpuPlacement: false  # set true for latency-sensitive
        memory:
          guest: 16Gi
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
            - name: datadisk
              disk:
                bus: virtio
            - name: cloudinit
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}  # or bridge for legacy VLAN
            - name: legacy-vlan-100
              bridge: {}
          networkInterfaceMultiqueue: true
        machine:
          type: q35
        resources:
          requests:
            memory: 16Gi
          limits:
            memory: 16Gi
      networks:
        - name: default
          pod: {}
        - name: legacy-vlan-100
          multus:
            networkName: vlan100-bridge
      volumes:
        - name: rootdisk
          dataVolume:
            name: rhel-app-server-001-rootdisk
        - name: datadisk
          dataVolume:
            name: rhel-app-server-001-datadisk
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              package_update: false
              runcmd:
                - [systemctl, enable, --now, qemu-guest-agent]
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: rhel-app-server-001-rootdisk
  namespace: workload-linux-apps
spec:
  source:
    http:
      url: "http://artifact-server.enclave.local/vm-images/rhel-app-001-root.qcow2"
  storage:
    storageClassName: ceph-rbd-ssd-ec
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
```

### 9.4 Network Attachment for Legacy VLANs

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-bridge
  namespace: workload-linux-apps
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-vlan100",
      "vlan": 100,
      "ipam": {},
      "preserveDefaultVlan": false
    }
```

### 9.5 Migration Tooling

| Tool | Purpose |
|------|---------|
| `qemu-img` | Convert VMDK to qcow2 format |
| `virtctl` (KubeVirt CLI) | Upload images, console access, start/stop/migrate VMs |
| CDI (Containerized Data Importer) | Import disk images into PVCs |
| `ovftool` (VMware) | Export VMs from vSphere (OVA/VMDK) |
| Ansible | Post-migration configuration (guest agent, network config, app validation) |
| Custom migration tracker | Spreadsheet or GitLab issues tracking per-VM migration status |

### 9.6 Rollback Strategy

For each wave:

- **vSphere VMs are NOT decommissioned** until 72 hours of successful production operation on KubeVirt
- If a VM fails validation on KubeVirt, the original vSphere VM is restarted immediately
- DNS/LB changes are the last step and are easily reversible
- All migration manifests are stored in Git -- rollback is `kubectl delete vm <name>` and DNS revert

---

## 10. ATO Maintenance Plan

### 10.1 ATO Impact Assessment

Migrating from VMware to KubeVirt is a **significant change** to the system boundary. This will require a **security impact analysis (SIA)** and likely a **modified ATO** rather than a complete re-accreditation, provided:

1. The data classification level does not change (remains IL5)
2. The physical boundary does not change (same air-gapped enclave, same hardware)
3. Security controls are maintained or strengthened
4. The ISSO/ISSM and AO are engaged early

### 10.2 RMF Artifacts to Update

| Artifact | Action Required |
|----------|----------------|
| System Security Plan (SSP) | Update technology descriptions, architecture diagrams, control implementations |
| Security Control Assessment (SCA) | Re-assess controls affected by platform change (CM, SC, SI, AC families) |
| Hardware/Software Inventory | Update with all new components (RKE2, KubeVirt, Ceph, Cilium, etc.) |
| Network Diagrams | Update to reflect new network architecture (Cilium, Multus, MetalLB) |
| Data Flow Diagrams | Update to reflect new storage paths (Ceph RBD vs vSAN/SAN) |
| POA&M | Document any temporary gaps during migration; track remediation |
| Ports, Protocols, and Services (PPS) | Update for Kubernetes API (6443), etcd (2379-2380), Kubelet (10250), Ceph (6789, 3300, 6800-7300) |
| Contingency Plan (CP) | Update DR procedures for KubeVirt/Ceph (see Section 11) |
| Configuration Management Plan | Update to reflect GitOps model, ArgoCD, Kyverno enforcement |

### 10.3 STIG Mapping

| VMware STIG | KubeVirt/K8s Equivalent |
|-------------|------------------------|
| VMware vSphere 7 ESXi STIG | DISA Kubernetes STIG + RHEL STIG (on host OS) |
| VMware vSphere 7 vCenter STIG | Rancher hardening guide + Kubernetes STIG |
| VMware vSphere 7 VM STIG | KubeVirt VM security context + guest OS STIG |

### 10.4 Continuous Authorization (cATO) Path

The migration to Kubernetes positions the organization for cATO:

- **Automated STIG scanning:** OpenSCAP on hosts, Kyverno policies on K8s, Trivy on images
- **Continuous monitoring:** Prometheus metrics, Falco runtime alerts, Cilium flow logs -- all piped to SIEM
- **Automated compliance reporting:** Generate STIG/SCAP results on schedule, push to compliance dashboard
- **Drift detection:** ArgoCD detects configuration drift, Kyverno blocks non-compliant deployments
- **Immutable infrastructure:** Base node images are hardened and versioned; changes require new image build, not ad-hoc patching

This provides the AO with real-time compliance visibility, which is the foundation for cATO approval.

---

## 11. Disaster Recovery

### 11.1 RTO/RPO Targets

| Tier | Workloads | RPO | RTO |
|------|-----------|-----|-----|
| Tier 1 (Critical) | Databases, authentication, core apps | 1 hour | 4 hours |
| Tier 2 (Important) | Application servers, middleware | 4 hours | 8 hours |
| Tier 3 (Standard) | Development, test, non-critical infra | 24 hours | 24 hours |

### 11.2 Backup Architecture

| Component | Backup Tool | Backup Target | Frequency |
|-----------|------------|---------------|-----------|
| Kubernetes resources (manifests) | Velero | Ceph RGW (S3) | Every 6 hours |
| VM persistent volumes | Velero + CSI snapshots | Ceph RBD snapshots | Daily (Tier 1: hourly) |
| etcd | RKE2 built-in etcd snapshot | Local + Ceph RGW | Every 2 hours |
| Ceph cluster | Ceph RBD snapshots + rbd export | Secondary Ceph pool or external NAS | Daily |
| GitOps state | Git (GitLab) | GitLab backup to Ceph RGW | Continuous (every commit) |
| Harbor registry | Harbor built-in backup + Velero | Ceph RGW | Daily |
| Vault secrets | Vault snapshot agent | Encrypted to Ceph RGW | Daily |

### 11.3 Recovery Procedures

**Scenario: Total cluster loss (catastrophic)**

1. Rebuild RKE2 cluster from automation (Ansible playbooks for node provisioning, RKE2 install)
2. Restore etcd from snapshot
3. Restore ArgoCD -- it will reconcile all workloads from Git
4. Restore PVCs from Ceph RBD snapshots (or Velero CSI snapshot restore)
5. Verify VM boot and application health
6. Estimated RTO: 4-8 hours (depending on cluster size and data volume)

**Scenario: Single node failure**

1. Kubernetes automatically reschedules VMs to other nodes (with live migration if node is cordoned gracefully)
2. Ceph self-heals with data replicated across remaining OSDs
3. No manual intervention required for transient failures
4. Replace failed node, re-add to cluster via Ansible
5. Estimated RTO: Minutes (automatic failover)

**Scenario: Ceph OSD failure**

1. Ceph automatically re-replicates data from remaining replicas
2. Replace failed drive, add new OSD via Rook operator
3. Ceph rebalances automatically
4. No data loss (3x replication or EC provides redundancy)

### 11.4 DR Testing

- **Monthly:** Restore a single VM from backup to a test namespace, validate application function
- **Quarterly:** Simulate node failure (cordon + drain), verify VM live migration and workload continuity
- **Semi-annually:** Full DR drill -- restore entire wave from backup, validate end-to-end

---

## 12. Operational Runbook Framework

### 12.1 Day-2 Operations

| Operation | Tool | Frequency |
|-----------|------|-----------|
| VM lifecycle (create/delete/resize) | Rancher UI, kubectl, virtctl | On-demand |
| VM live migration | `virtctl migrate <vm>` | As needed (maintenance) |
| Node maintenance (OS patching) | Ansible + `kubectl drain` | Monthly |
| Kubernetes upgrade | RKE2 air-gap upgrade procedure | Quarterly |
| KubeVirt upgrade | Helm chart update via ArgoCD | Quarterly |
| Ceph maintenance | Rook operator, `ceph` CLI via toolbox pod | As needed |
| Certificate rotation | cert-manager (auto) or manual | Before expiry (automated) |
| Capacity expansion | Add nodes via Ansible, join to RKE2 cluster | As needed |
| User/RBAC management | Rancher + AD/LDAP integration | On-demand |
| Incident response | Falco alerts -> SIEM -> IR process | Event-driven |

### 12.2 Monitoring and Alerting

| Metric Category | Tool | Alert Threshold |
|-----------------|------|-----------------|
| Node health (CPU, RAM, disk) | Prometheus node_exporter | CPU > 80% sustained, RAM > 90%, disk > 85% |
| Kubernetes health | kube-state-metrics | Pod crash loops, pending pods, failed deployments |
| KubeVirt VM health | KubeVirt metrics exporter | VM not running, migration failures |
| Ceph health | Rook Ceph metrics | HEALTH_WARN, OSD down, PG degraded |
| Network health | Cilium Hubble metrics | Dropped flows, policy denies, high latency |
| Certificate expiry | cert-manager metrics | < 30 days to expiry |
| STIG compliance | OpenSCAP metrics | Any CAT I finding = critical alert |
| Security events | Falco | Any high-severity syscall alert |

### 12.3 Grafana Dashboards

Pre-built dashboards for:
- Cluster overview (node status, resource utilization, pod count)
- KubeVirt VM dashboard (per-VM CPU, RAM, disk I/O, network I/O)
- Ceph storage dashboard (capacity, IOPS, latency, OSD health)
- Cilium network dashboard (flow rates, policy enforcement, drop reasons)
- Security dashboard (Falco alerts, STIG compliance scores, CVE counts)
- Capacity planning dashboard (trend lines, forecasting)

---

## 13. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|-----------|--------|------------|
| R1 | KubeVirt VM performance lower than native ESXi | Medium | Medium | Benchmark before migration; use CPU pinning, hugepages, SR-IOV for performance-critical VMs; tune virtio drivers |
| R2 | Operations team unfamiliar with Kubernetes | High | High | 4-week training program before migration; Rancher UI reduces CLI dependency; hire/contract K8s-experienced staff |
| R3 | Air-gapped artifact transfer bottleneck | Medium | Medium | Establish regular transfer cadence; pre-stage all artifacts before each wave; automate import pipeline |
| R4 | ATO modification rejected or delayed | Low | Critical | Engage ISSO/ISSM/AO at project kickoff; provide SIA early; demonstrate equivalent or better security posture |
| R5 | Windows VM compatibility issues in KubeVirt | Medium | Medium | Test Windows VMs in pilot wave; ensure virtio drivers installed pre-migration; fallback to IDE/SATA bus if needed |
| R6 | Ceph storage performance insufficient | Low | High | Size NVMe tier appropriately for databases; benchmark with fio before migration; tune Ceph OSD parameters |
| R7 | Migration takes longer than planned | Medium | Medium | Conservative wave sizing; dedicated migration team; parallel migration tracks for independent VMs |
| R8 | Vendor support gaps for KubeVirt | Low | Medium | Rancher Government Solutions provides support; engage SUSE support contract; community is active and responsive |
| R9 | Hardware failures during migration | Low | Medium | N+2 node redundancy; Ceph 3x replication; maintain vSphere as fallback until wave validated |
| R10 | Loss of VMware-specific features (DRS, HA) | Medium | Low | Map to K8s equivalents: pod affinity = DRS, ReplicaSet/reschedule = HA, `virtctl migrate` = vMotion |

---

## 14. Cost Analysis

### 14.1 VMware Licensing Cost (Current -- Annual)

| Component | Estimated Annual Cost |
|-----------|----------------------|
| vSphere Enterprise Plus (per CPU) | $200,000 - $400,000 (depending on core count and Broadcom bundle) |
| vCenter Standard | $15,000 - $25,000 |
| vSAN (if used) | $100,000 - $200,000 |
| Support & Subscription renewals | $80,000 - $150,000 |
| **Total VMware Annual** | **$395,000 - $775,000** |

> **Note:** Post-Broadcom acquisition, many customers report 2-5x price increases on renewal. These estimates may be conservative.

### 14.2 Target Platform Cost (Annual)

| Component | Annual Cost |
|-----------|-------------|
| RKE2 Government | $0 (Apache 2.0) |
| KubeVirt | $0 (Apache 2.0, CNCF) |
| Rook-Ceph | $0 (Apache 2.0) |
| Cilium | $0 (Apache 2.0) |
| Rancher | $0 (Apache 2.0) |
| All other CNCF/FLOSS components | $0 |
| **Rancher Government Solutions Support Contract** | **$100,000 - $200,000** (for IL5 cleared support) |
| **RHEL subscriptions** (if not using Rocky) | **$50,000 - $80,000** (for 15-20 nodes) |
| **OR Rocky Linux** (FLOSS alternative) | **$0** (community support, or CIQ support contract ~$30,000-$50,000) |
| **Total Target Platform Annual** | **$100,000 - $280,000** |

### 14.3 One-Time Migration Costs

| Item | Estimated Cost |
|------|---------------|
| Training (K8s, KubeVirt, Ceph for ops team, 6-8 people) | $40,000 - $80,000 |
| Professional services (architecture, migration assistance) | $150,000 - $300,000 |
| Additional hardware (if current hardware insufficient) | $0 - $500,000 (depends on current hardware) |
| ATO modification (SIA, documentation, SCA) | $50,000 - $100,000 |
| **Total One-Time** | **$240,000 - $980,000** |

### 14.4 5-Year TCO Comparison

| | VMware (5 years) | KubeVirt/RKE2 Gov (5 years) |
|---|---|---|
| Year 1 | $600,000 (licensing) | $470,000 (migration + platform) |
| Year 2 | $650,000 (price increase) | $200,000 (support + OS) |
| Year 3 | $715,000 (price increase) | $200,000 |
| Year 4 | $790,000 (price increase) | $210,000 |
| Year 5 | $870,000 (price increase) | $210,000 |
| **5-Year Total** | **~$3,625,000** | **~$1,290,000** |
| **5-Year Savings** | | **~$2,335,000 (64% reduction)** |

> **Assumptions:** VMware costs increase ~10% annually (conservative post-Broadcom). KubeVirt platform costs increase ~5% for support contract inflation. Hardware costs are assumed equal (same physical infrastructure reused).

---

## 15. Timeline and Milestones

### Phase 0: Planning and Procurement (Weeks 1-6)

| Week | Activity |
|------|----------|
| 1-2 | Detailed discovery: RVTools export, workload profiling, dependency mapping |
| 2-3 | Hardware assessment: validate existing hardware supports target architecture |
| 3-4 | Procurement: Rancher Gov support contract, RHEL subscriptions (if needed), training |
| 4-5 | ATO engagement: brief ISSO/ISSM, begin Security Impact Analysis |
| 5-6 | Team training: Kubernetes fundamentals, KubeVirt, Rook-Ceph, Cilium |

### Phase 1: Platform Build (Weeks 7-16)

| Week | Activity |
|------|----------|
| 7-8 | Artifact staging: transfer all container images, Helm charts, RPMs to enclave |
| 8-9 | Base OS install: RHEL/Rocky STIG-hardened on all nodes, FIPS mode enabled |
| 9-10 | Management cluster: deploy RKE2, Rancher, Harbor, GitLab, Vault |
| 10-11 | Workload cluster: deploy RKE2, join workers, configure Cilium, MetalLB |
| 11-12 | Storage: deploy Rook-Ceph, create pools, storage classes, benchmark |
| 12-13 | KubeVirt: deploy KubeVirt operator, CDI, configure live migration |
| 13-14 | Networking: configure Multus, bridge/VLAN attachments for legacy networks |
| 14-15 | Security stack: deploy Falco, Kyverno, OpenSCAP, ArgoCD, monitoring |
| 15-16 | Integration testing: end-to-end validation, STIG scan, performance benchmarks |

### Phase 2: Migration Execution (Weeks 17-36)

| Week | Activity |
|------|----------|
| 17-18 | Wave 0 (Pilot): 5-10 non-critical VMs, validate full migration process |
| 19-21 | Wave 1: 30-40 Linux infrastructure VMs |
| 22-25 | Wave 2: 50-60 Linux application VMs |
| 26-30 | Wave 3: 50-60 Windows + stateful Linux (databases) |
| 31-34 | Wave 4: 40-50 security appliances + remaining complex workloads |
| 35-36 | Migration validation: full regression, performance validation, security scan |

### Phase 3: Decommission and Close (Weeks 37-40)

| Week | Activity |
|------|----------|
| 37-38 | Decommission vSphere: power off all source VMs, export final backups |
| 38-39 | ATO update: submit final SSP, complete SCA, update POA&M |
| 39-40 | Remove VMware: uninstall ESXi from hosts (or repurpose hardware as additional K8s workers) |
| 40 | Project close: lessons learned, runbook finalization, handoff to operations |

**Total Duration: 40 weeks (~10 months)**

---

## 16. Architectural Decision Records

### ADR-001: KubeVirt on RKE2 Government as VMware Replacement
**Status:** Accepted
**See:** Section 3 (detailed rationale above)

### ADR-002: Rook-Ceph for Software-Defined Storage
**Status:** Accepted
**Context:** Need block, file, and object storage without proprietary SAN licensing.
**Decision:** Deploy Rook-Ceph as the unified storage platform.
**Rationale:** Ceph is the most mature open-source software-defined storage platform. Rook provides Kubernetes-native orchestration. Supports RBD (block for VM disks), CephFS (shared filesystems), and RGW (S3-compatible object). Eliminates vSAN licensing. Runs on commodity NVMe/SSD drives already in the servers.
**Consequences:** Ceph requires minimum 3 nodes for replication. OSD placement and CRUSH map tuning require expertise. Team must learn Ceph operations.

### ADR-003: Cilium as Primary CNI
**Status:** Accepted
**Context:** Need high-performance, policy-enforcing CNI with observability for IL5.
**Decision:** Use Cilium as the primary CNI with Hubble for observability.
**Rationale:** eBPF-based dataplane provides superior performance vs. iptables-based CNIs. Native support for Kubernetes NetworkPolicy and extended CiliumNetworkPolicy for L7 filtering. Hubble provides network flow visibility critical for security monitoring and incident response. IPsec mode is FIPS-compliant for pod-to-pod encryption. Replaces kube-proxy entirely.
**Consequences:** Requires kernel 5.x+ (standard on RHEL 8.6+/9.x). Team must learn Cilium policy model.

### ADR-004: Multus for Legacy VM Network Attachment
**Status:** Accepted
**Context:** Migrated VMs need to retain existing IP addresses and VLAN memberships to minimize ATO impact and application changes.
**Decision:** Use Multus CNI to attach VMs to legacy VLANs via bridge and VLAN plugins.
**Rationale:** Allows VMs to appear on existing network segments with same IPs. Upstream firewalls, IDS/IPS, and network monitoring remain unchanged. Critical for maintaining ATO without re-accrediting all network flows.
**Consequences:** Requires physical NIC access to VLAN trunks on worker nodes. Bridge configuration must be managed via Ansible/NMState.

### ADR-005: Harbor as Air-Gapped Registry
**Status:** Accepted
**Context:** Need a container registry that operates fully offline and supports image scanning, signing, and Helm charts.
**Decision:** Deploy Harbor as the single container image and Helm chart registry.
**Rationale:** Harbor supports OCI artifacts (images + Helm charts), integrated Trivy vulnerability scanning, image signing with cosign/Notary, replication policies, RBAC, and LDAP/AD authentication. It is the standard registry for DoD Platform One environments. Apache 2.0 licensed.
**Consequences:** Requires regular offline updates to Trivy vulnerability database. Image import process must be formalized and documented.

### ADR-006: GitOps with ArgoCD for Configuration Management
**Status:** Accepted
**Context:** IL5 requires strict configuration management with audit trails and drift detection.
**Decision:** Use ArgoCD for GitOps-based deployment of all cluster workloads and configuration.
**Rationale:** All changes are committed to Git (audit trail). ArgoCD detects and optionally auto-remediates drift. Declarative desired state eliminates manual `kubectl apply` commands. Aligns with Big Bang reference architecture. Supports multi-cluster management from management cluster.
**Consequences:** All operational changes must go through Git. Team must adopt Git-based workflow. Emergency changes still require Git commit (no direct cluster modification in production).

### ADR-007: Vault for Secrets Management
**Status:** Accepted (with license caveat)
**Context:** Need centralized secrets management with FIPS-compliant encryption and HSM support.
**Decision:** Deploy HashiCorp Vault (or OpenBao) with HSM auto-unseal.
**Rationale:** Vault provides dynamic secrets, PKI, transit encryption, and Kubernetes auth integration. HSM auto-unseal eliminates manual unseal operations. External Secrets Operator syncs Vault secrets into Kubernetes Secrets. If BSL licensing is a concern, OpenBao (Linux Foundation fork) provides equivalent functionality under open-source license.
**Consequences:** Vault is a critical dependency. Must be deployed in HA mode (3+ replicas). HSM procurement may be required.

---

## Appendix A: Prerequisite Checklist

Before beginning the migration, confirm:

- [ ] Hardware inventory validated (CPU, RAM, disk, NIC) against target sizing
- [ ] All nodes have IPMI/BMC access for out-of-band management
- [ ] Network switches configured with required VLANs (management, Ceph, data, legacy)
- [ ] Physical rack space and power verified for any additional hardware
- [ ] RVTools or equivalent export from vCenter completed
- [ ] Application dependency map documented for all 200 VMs
- [ ] ISSO/ISSM briefed and Security Impact Analysis initiated
- [ ] Rancher Government Solutions support contract procured
- [ ] Training scheduled for operations team
- [ ] Air-gap transfer process established and tested
- [ ] Iron Bank account provisioned (or access to mirrored Iron Bank content)
- [ ] All required container images and Helm charts staged on transfer media
- [ ] RPM repository snapshot created and transferred
- [ ] Backup of all existing VMware VMs completed (pre-migration insurance)

## Appendix B: Key Contacts and Resources

| Resource | URL / Contact |
|----------|--------------|
| Rancher Government Solutions | https://ranchergovernment.com (from unclassified network) |
| Platform One / Big Bang | https://p1.dso.mil (from appropriate network) |
| Iron Bank | https://ironbank.dso.mil (from appropriate network) |
| RKE2 Documentation | https://docs.rke2.io |
| KubeVirt Documentation | https://kubevirt.io/user-guide |
| Rook-Ceph Documentation | https://rook.io/docs/rook/latest |
| DISA STIGs | https://public.cyber.mil/stigs |

## Appendix C: Glossary

| Term | Definition |
|------|-----------|
| ATO | Authority to Operate -- formal authorization to run a system at a given classification level |
| cATO | Continuous Authority to Operate -- ongoing authorization based on automated compliance monitoring |
| CDI | Containerized Data Importer -- KubeVirt component for importing VM disk images |
| CDS | Cross-Domain Solution -- system enabling controlled data transfer between classification levels |
| CIS | Center for Internet Security -- publishes hardening benchmarks |
| CMDB | Configuration Management Database |
| DISA | Defense Information Systems Agency |
| FIPS | Federal Information Processing Standards -- cryptographic module validation program |
| IL5 | Impact Level 5 -- DoD classification for CUI and NSS on non-classified networks |
| ISSO | Information System Security Officer |
| ISSM | Information System Security Manager |
| KubeVirt | CNCF project enabling VM management within Kubernetes |
| POA&M | Plan of Action and Milestones -- tracks security deficiencies and remediation |
| RKE2 | Rancher Kubernetes Engine 2 -- CNCF-conformant K8s distribution |
| RMF | Risk Management Framework -- NIST-based accreditation framework used by DoD |
| SCA | Security Control Assessment |
| SIA | Security Impact Analysis |
| STIG | Security Technical Implementation Guide -- DISA configuration hardening standard |
| TCO | Total Cost of Ownership |

---

*This document should be treated as a living architecture artifact. Update it as decisions evolve, discoveries are made during migration execution, and lessons learned are incorporated.*
