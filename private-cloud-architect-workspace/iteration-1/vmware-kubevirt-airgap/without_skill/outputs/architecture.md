# VMware vSphere 7 to KubeVirt on RKE2 Government Migration
# Architecture & Migration Plan -- IL5 Air-Gapped Environment

**Classification:** CUI // SP-EXPT
**Document Version:** 1.0
**Date:** 2026-03-20
**Distribution:** Limited to program stakeholders and AO staff

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Assessment](#2-current-state-assessment)
3. [Target Architecture Selection & Justification](#3-target-architecture-selection--justification)
4. [Target Architecture Design](#4-target-architecture-design)
5. [Network Architecture](#5-network-architecture)
6. [Storage Architecture](#6-storage-architecture)
7. [Security Architecture & STIG Compliance](#7-security-architecture--stig-compliance)
8. [Air-Gap Infrastructure](#8-air-gap-infrastructure)
9. [Migration Strategy](#9-migration-strategy)
10. [Migration Execution Plan](#10-migration-execution-plan)
11. [ATO Continuity Plan](#11-ato-continuity-plan)
12. [Operational Runbook Considerations](#12-operational-runbook-considerations)
13. [Risk Register](#13-risk-register)
14. [Resource Requirements](#14-resource-requirements)
15. [Timeline & Milestones](#15-timeline--milestones)
16. [Appendices](#16-appendices)

---

## 1. Executive Summary

### 1.1 Problem Statement

The organization currently operates approximately 200 virtual machines on VMware vSphere 7 within an Impact Level 5 (IL5) air-gapped enclave. Following Broadcom's acquisition of VMware, licensing costs have escalated dramatically and contract terms have become untenable. The vSphere 7 platform is also approaching end of general support, creating both financial and security risk for continued operations under the existing Authorization to Operate (ATO).

### 1.2 Proposed Solution

Migrate the entire virtualization workload from VMware vSphere 7 to **KubeVirt running on RKE2 Government (RKE2-Gov)**, an air-gap-capable, FIPS-validated Kubernetes distribution purpose-built for U.S. federal and defense use cases. KubeVirt provides VM orchestration as a Kubernetes-native workload, preserving existing VM-based applications while opening a path toward containerized modernization.

### 1.3 Key Decision Drivers

| Driver | Detail |
|--------|--------|
| **Cost** | Eliminate VMware/Broadcom per-CPU licensing; RKE2 is open source (Rancher support optional) |
| **Compliance** | RKE2-Gov ships FIPS 140-2/140-3 validated crypto modules; DISA STIG available |
| **Air-gap native** | RKE2 was designed for disconnected installation from day one |
| **ATO continuity** | Maintain existing IL5 ATO via POA&M during transition; target full re-authorization on new stack |
| **Operational maturity** | KubeVirt reached GA (v1.0+) and is deployed in multiple DoD programs of record |

### 1.4 Alternatives Considered

| Alternative | Verdict | Rationale |
|-------------|---------|-----------|
| **OpenShift Virtualization (OCP-V)** | Viable but more expensive | Red Hat subscription costs partially offset VMware savings; heavier footprint; excellent STIG coverage |
| **Proxmox VE** | Not recommended | No DISA STIG; no FIPS-validated crypto; limited DoD track record; would require significant custom hardening |
| **Harvester HCI** | Strong contender | Built on RKE2 + KubeVirt + Longhorn; purpose-built HCI; fewer knobs but less flexibility; evaluated as optional overlay in this design |
| **oVirt / OLVM** | Declining | Upstream oVirt community is winding down; Oracle Linux Virtualization Manager future uncertain |
| **Nutanix AHV** | Viable but licensed | Proprietary licensing; strong IL5 story; does not meet "eliminate licensing" goal |
| **Plain libvirt/KVM** | Not recommended at scale | No orchestration layer; operational burden for 200 VMs; no centralized management |

**Recommendation: RKE2-Gov + KubeVirt** provides the best balance of cost elimination, compliance posture, air-gap maturity, and future modernization optionality.

---

## 2. Current State Assessment

### 2.1 Existing VMware Environment

| Attribute | Value |
|-----------|-------|
| Hypervisor | VMware vSphere 7.0 U3 |
| vCenter | Single vCenter Server Appliance (VCSA) |
| ESXi Hosts | Estimated 12-16 (to be confirmed in discovery) |
| Total VMs | ~200 |
| Storage Backend | Likely VMFS on shared SAN (FC or iSCSI) or vSAN |
| Networking | vSphere Distributed Switch (VDS), likely NSX or VLANs |
| Backup | Veeam, Commvault, or similar |
| Monitoring | vROps, Nagios, or similar |
| Classification | IL5 / CUI |
| Connectivity | Air-gapped, no internet |

### 2.2 VM Workload Classification

Before migration, every VM must be categorized. Typical classifications:

| Category | Description | Example Workloads | Migration Complexity |
|----------|-------------|-------------------|---------------------|
| **Tier 1 -- Stateless/Web** | Stateless application servers, web frontends | Apache, Nginx, Tomcat | Low |
| **Tier 2 -- Stateful Apps** | Application servers with local state | Custom Java/.NET apps, middleware | Medium |
| **Tier 3 -- Databases** | Relational and NoSQL databases | PostgreSQL, Oracle, SQL Server, MongoDB | High |
| **Tier 4 -- Infrastructure** | DNS, AD/LDAP, NTP, SIEM, log collectors | Windows AD DCs, BIND, Splunk | High (dependencies) |
| **Tier 5 -- Special** | GPU workloads, real-time, SCADA, legacy OS | RHEL 6, Windows Server 2012, CUDA workloads | Very High |

### 2.3 Discovery Requirements

Run a comprehensive discovery before finalizing migration waves:

- **VM inventory export:** `govc` or PowerCLI to extract all VMs, resource allocations, snapshots, disk layouts
- **Dependency mapping:** Use network flow analysis (if NSX is in place, use its flow data) or deploy an agent-based tool via sneakernet
- **Performance baselines:** Capture 30 days of vROps or esxtop data for CPU, memory, disk IOPS, network throughput per VM
- **Application owner interviews:** Identify maintenance windows, RPO/RTO, and any VMware-specific dependencies (e.g., VMware Tools guest hooks, vGPU, FT, DRS affinity rules)
- **Storage audit:** Total capacity, thin vs. thick provisioning, VMDK sizes, RDMs, snapshots, datastore layout
- **Network audit:** All VLANs, port groups, firewall rules, load balancer VIPs, DNS entries

---

## 3. Target Architecture Selection & Justification

### 3.1 Why RKE2 Government

RKE2 (also known as "RKE Government") is Rancher's next-generation Kubernetes distribution focused on security and compliance:

- **FIPS 140-2 validated:** Ships with FIPS-validated Go crypto modules and etcd encryption. Meets the FIPS requirements for IL5.
- **CIS Kubernetes Benchmark:** Passes CIS Level 1 and Level 2 by default out of the box.
- **DISA STIG:** DISA has published a Kubernetes STIG and RKE2 has a published STIG hardening guide with automated remediation via inspec profiles.
- **Air-gap first:** Official air-gap installation method is documented and battle-tested. All images are packaged into tarballs for offline deployment.
- **Containerd runtime:** Uses containerd (not Docker), reducing attack surface.
- **Embedded etcd:** No external etcd dependency (though external etcd is supported for very large clusters).
- **SELinux support:** Full SELinux enforcing mode compatibility on RHEL-based hosts.

### 3.2 Why KubeVirt

KubeVirt extends Kubernetes with VM management capability by running VMs inside pods via QEMU/KVM:

- **Kubernetes-native:** VMs become `VirtualMachine` and `VirtualMachineInstance` custom resources. Standard kubectl, RBAC, and GitOps workflows apply.
- **KVM performance:** Near-native performance; KVM is the same hypervisor technology underlying RHEL, OpenShift Virtualization, and most cloud providers.
- **Live migration:** Supported for VMs between nodes (requires shared storage).
- **VM networking:** Multus CNI integration supports multiple network interfaces, VLANs, SR-IOV, and bridge networking.
- **Mature ecosystem:** KubeVirt v1.0+ GA; backed by Red Hat, SUSE, and a broad community; used in OpenShift Virtualization (same codebase).
- **Disk import:** The Containerized Data Importer (CDI) can import VMDKs, qcow2, raw images, and ISOs.

### 3.3 SUSE Rancher for Management (Optional but Recommended)

Rancher provides a centralized management UI and multi-cluster lifecycle management. In an air-gapped environment, Rancher can be deployed on the management cluster and used for:

- Cluster provisioning and upgrades
- Centralized RBAC and authentication (integrated with DoD CAC/PKI via LDAP or SAML)
- Monitoring and alerting (Rancher Monitoring based on Prometheus/Grafana)
- Audit logging

Rancher is open source; SUSE offers a paid support subscription if required for contract deliverables.

---

## 4. Target Architecture Design

### 4.1 High-Level Architecture

```
+----------------------------------------------------------------------+
|                        IL5 AIR-GAPPED ENCLAVE                        |
|                                                                      |
|  +----------------------------+   +-------------------------------+  |
|  |   MANAGEMENT CLUSTER       |   |   WORKLOAD CLUSTER            |  |
|  |   (RKE2-Gov, 3 nodes)      |   |   (RKE2-Gov, 3 CP + N workers)|  |
|  |                             |   |                               |  |
|  |  - Rancher Server           |   |  - KubeVirt Operator          |  |
|  |  - Harbor Registry          |   |  - CDI (Containerized Data    |  |
|  |  - GitLab (GitOps)          |   |    Importer)                  |  |
|  |  - Vault (secrets)          |   |  - Multus CNI                 |  |
|  |  - Monitoring Stack         |   |  - 200 VMs as                 |  |
|  |  - Neuvector (security)     |   |    VirtualMachine CRs         |  |
|  |  - Backup (Velero)          |   |  - Rook-Ceph / Longhorn       |  |
|  +----------------------------+   |    (storage)                   |  |
|                                   |  - MetalLB (load balancing)    |  |
|  +----------------------------+   +-------------------------------+  |
|  |   AIR-GAP TRANSFER STATION |                                      |
|  |   (Hardened bastion)        |                                      |
|  |  - Content mirroring        |                                      |
|  |  - Image scanning           |                                      |
|  |  - Patch staging            |                                      |
|  +----------------------------+                                      |
|                                                                      |
|  +----------------------------+                                      |
|  |   SHARED STORAGE (Ceph)    |                                      |
|  |   or EXISTING SAN          |                                      |
|  +----------------------------+                                      |
+----------------------------------------------------------------------+
```

### 4.2 Cluster Topology

#### Management Cluster

| Role | Count | Specs (minimum) | Purpose |
|------|-------|-----------------|---------|
| Control plane + worker | 3 | 8 vCPU, 32 GB RAM, 500 GB SSD | Rancher, Harbor, monitoring, GitOps |

#### Workload Cluster (KubeVirt)

| Role | Count | Specs (minimum) | Purpose |
|------|-------|-----------------|---------|
| Control plane | 3 | 8 vCPU, 32 GB RAM, 200 GB SSD | etcd, API server, scheduler, controller-manager |
| Worker (VM hosts) | 12-16 | 64-128 vCPU, 512-1024 GB RAM, NVMe local | Run KubeVirt VMs; sized to replace existing ESXi hosts 1:1 |
| Storage nodes (if Ceph) | 3-5 | 16 vCPU, 64 GB RAM, 10x 4TB NVMe | Rook-Ceph OSD nodes |

**Note:** Worker node sizing depends on discovery results. Plan for the same aggregate compute as existing ESXi hosts plus 15-20% overhead for Kubernetes system pods, KubeVirt overhead, and live migration headroom.

#### Hardware Considerations

- **CPU:** Must support Intel VT-x/VT-d or AMD-V/AMD-Vi for nested/KVM virtualization. Confirm `/dev/kvm` availability on all worker nodes.
- **BIOS/UEFI:** Enable IOMMU for SR-IOV or PCI passthrough if needed (GPU workloads, high-performance NIC).
- **NIC:** Minimum 2x 25 GbE per node (management + workload); 100 GbE recommended for storage traffic if using Ceph.
- **BMC/iLO/iDRAC:** Required for out-of-band management; must be on isolated management VLAN.

### 4.3 Operating System

**RHEL 8.x or RHEL 9.x** (with active DoD STIG):

- FIPS mode enabled at install (`fips=1` kernel parameter)
- SELinux enforcing
- DISA STIG applied via SCAP/OpenSCAP or Ansible hardening playbook
- Registered to local Satellite or DNF repository mirror (air-gapped)

**Alternative:** SUSE Linux Enterprise Server 15 SP5+ if standardizing on the SUSE stack (STIG available but less mature than RHEL).

### 4.4 Kubernetes Component Stack

| Component | Product | Version (target) | Purpose |
|-----------|---------|-------------------|---------|
| Kubernetes Distribution | RKE2-Gov | v1.28+ (latest stable) | Container orchestration |
| Container Runtime | containerd | Bundled with RKE2 | Pod runtime |
| CNI (primary) | Canal (Calico + Flannel) | Bundled with RKE2 | Pod networking |
| CNI (secondary) | Multus | Latest stable | Multi-NIC VM support |
| Ingress | NGINX Ingress | Bundled with RKE2 | HTTP/HTTPS ingress |
| Load Balancer | MetalLB | Latest stable | Bare-metal LB for services |
| Service Mesh | Istio (optional) | Latest stable | mTLS, traffic management |
| DNS | CoreDNS | Bundled with RKE2 | Cluster DNS |
| Virtualization | KubeVirt | v1.2+ | VM lifecycle management |
| VM Disk Import | CDI | Matched to KubeVirt | Disk image import/clone |
| Storage (primary) | Rook-Ceph | Latest stable | Distributed block/file/object storage |
| Storage (alternative) | Longhorn | Latest stable | Lightweight distributed block storage |
| Registry | Harbor | Latest stable | Air-gapped container image registry |
| Secrets | HashiCorp Vault | Latest stable | Secret management, PKI |
| GitOps | Fleet or ArgoCD | Latest stable | Declarative cluster config |
| Monitoring | Prometheus + Grafana | Via Rancher Monitoring chart | Metrics and dashboards |
| Logging | Fluentd/Fluentbit + Elasticsearch or Loki | Latest stable | Centralized logging |
| Security Scanner | NeuVector | Latest stable | Runtime container security, admission control |
| Backup | Velero + Restic | Latest stable | Cluster and VM backup |
| Policy Engine | OPA Gatekeeper or Kyverno | Latest stable | Kubernetes policy enforcement |

---

## 5. Network Architecture

### 5.1 Network Segmentation

The IL5 enclave network must maintain strict segmentation:

```
+------------------------------------------------------------------+
|                     ENCLAVE BOUNDARY                              |
|                                                                   |
|  VLAN 10 - Management (BMC/iLO, SSH, Rancher UI)                |
|  VLAN 20 - Kubernetes Control Plane (API server, etcd)           |
|  VLAN 30 - Pod Network (Canal overlay or BGP-routed)             |
|  VLAN 40 - Storage Network (Ceph public + cluster)               |
|  VLAN 50 - VM Workload Network(s) (bridged via Multus)           |
|  VLAN 51-99 - Additional VM Networks (mapped from existing)      |
|  VLAN 100 - Air-Gap Transfer (one-way diode or sneakernet)       |
+------------------------------------------------------------------+
```

### 5.2 KubeVirt Networking with Multus

KubeVirt VMs require network configurations that mirror the existing VMware port groups. Multus CNI enables attaching multiple network interfaces to pods (and thus VMs):

**Strategy:** For each existing VMware port group / VLAN, create a corresponding `NetworkAttachmentDefinition` (NAD) using either:

- **Bridge CNI:** Linux bridge on each worker node mapped to a VLAN sub-interface. Best for most workloads.
- **SR-IOV CNI:** Direct hardware NIC virtual function passthrough. Best for high-throughput/low-latency workloads.
- **macvtap CNI:** Lightweight alternative to bridge; each VM gets a unique MAC on the physical network.

Example NetworkAttachmentDefinition:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan50-app-network
  namespace: vm-workloads
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-vlan50",
      "vlan": 50,
      "ipam": {}
    }
```

VMs can then reference this NAD:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: app-server-01
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
            - name: app-net
              bridge: {}
      networks:
        - name: default
          pod: {}
        - name: app-net
          multus:
            networkName: vlan50-app-network
```

### 5.3 DNS and IP Address Management

- Preserve existing IP addresses wherever possible to minimize application reconfiguration.
- Use bridge or macvtap networking so VMs get IPs directly from existing DHCP/static allocations on the physical network.
- Update DNS records to reflect any IP changes.
- Consider deploying a DHCP relay or dedicated IPAM (e.g., NetBox) for tracking.

### 5.4 Load Balancing

- **MetalLB** in L2 or BGP mode for Kubernetes Service type `LoadBalancer`.
- For VMs that previously used hardware load balancers (F5, etc.), those can continue unchanged if VMs retain their IPs on the bridged network.

### 5.5 Firewall and Network Policy

- **Calico NetworkPolicy** for pod-to-pod (and pod-to-VM) micro-segmentation.
- **KubeVirt-specific:** Apply NetworkPolicy to namespaces containing VM workloads to replicate existing firewall rules.
- Existing perimeter firewalls remain unchanged (the enclave boundary does not move).

---

## 6. Storage Architecture

### 6.1 Storage Strategy Decision

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Rook-Ceph** | Highly scalable, block + file + object, proven at scale, self-healing | Operational complexity, needs dedicated nodes or disks, heavier resource use | **Recommended for 200 VM environment** |
| **Longhorn** | Simpler, lightweight, good Rancher integration | Less mature at scale, block only, replicas consume more usable space | Good for smaller deployments (<50 VMs) |
| **Existing SAN (reuse)** | Zero migration for storage, known quantity | Requires CSI driver (e.g., democratic-csi for iSCSI/FC, vendor CSI), ties you to existing hardware lifecycle | **Recommended as bridge strategy during migration** |

**Recommended approach:** Use the **existing SAN** during migration to avoid moving storage and compute simultaneously. Deploy **Rook-Ceph** in parallel and migrate storage post-cutover as a Phase 2 activity.

### 6.2 Rook-Ceph Architecture

```
                     +-------------------+
                     |  Ceph Monitors (3) |
                     |  (on control plane |
                     |   or dedicated)    |
                     +-------------------+
                              |
        +---------------------+---------------------+
        |                     |                     |
+-------+-------+    +-------+-------+    +-------+-------+
| OSD Node 1    |    | OSD Node 2    |    | OSD Node 3    |
| 10x NVMe SSD  |    | 10x NVMe SSD  |    | 10x NVMe SSD  |
| Ceph OSD pods  |    | Ceph OSD pods  |    | Ceph OSD pods  |
+----------------+    +----------------+    +----------------+
```

- **CephBlockPool** with replication factor 3 for VM disks (`ReadWriteOnce` PVCs)
- **CephFilesystem (CephFS)** with `ReadWriteMany` for shared storage needs
- **StorageClass** definitions:
  - `ceph-block-ssd` -- for VM boot disks and high-performance data
  - `ceph-block-hdd` -- for bulk data (if spinning disks are mixed in)
  - `ceph-fs` -- for shared file systems

### 6.3 VM Disk Storage in KubeVirt

KubeVirt VM disks are backed by PersistentVolumeClaims (PVCs):

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: db-server-01
spec:
  template:
    spec:
      domain:
        devices:
          disks:
            - name: boot-disk
              disk:
                bus: virtio
            - name: data-disk
              disk:
                bus: virtio
      volumes:
        - name: boot-disk
          persistentVolumeClaim:
            claimName: db-server-01-boot
        - name: data-disk
          persistentVolumeClaim:
            claimName: db-server-01-data
```

### 6.4 Live Migration Storage Requirements

KubeVirt live migration requires `ReadWriteMany` (RWX) access mode for VM disks. Options:

- **Ceph RBD with `ReadWriteMany` (block mode):** Supported in Kubernetes via `volumeMode: Block`.
- **CephFS:** Supports RWX natively for file-mode volumes.
- **Shared SAN LUN via CSI:** Some SAN CSI drivers support RWX.

If live migration is not required for certain VMs, `ReadWriteOnce` (RWO) is sufficient and simpler.

---

## 7. Security Architecture & STIG Compliance

### 7.1 FIPS 140-2/140-3 Compliance

| Layer | FIPS Implementation |
|-------|---------------------|
| **OS** | RHEL FIPS mode (`fips=1`); all crypto operations use FIPS-validated modules |
| **RKE2** | Built with FIPS-validated Go crypto modules; etcd encryption at rest with FIPS-approved algorithms |
| **KubeVirt** | Runs atop RKE2 FIPS crypto; QEMU/KVM uses host OS FIPS modules for any crypto operations |
| **Storage (Ceph)** | Ceph encryption at rest using dm-crypt with FIPS-validated kernel modules |
| **TLS** | All inter-component communication uses TLS 1.2+ with FIPS-approved cipher suites |

### 7.2 DISA STIG Application

| Component | STIG | Notes |
|-----------|------|-------|
| RHEL 8/9 | V-xxxxx (RHEL STIG) | Apply at OS install, automate with OpenSCAP |
| Kubernetes | Kubernetes STIG (V-242XXX series) | RKE2 hardening guide maps controls |
| Container Runtime | Container STIG | Containerd-specific checks |
| KubeVirt VMs (guest OS) | Applicable guest OS STIG | Each VM guest must be STIGed independently (same as today) |
| Network devices | As applicable | Switches, firewalls per existing STIGs |

### 7.3 RKE2 Hardening Checklist

RKE2 CIS hardening profile (`profile: cis-1.23` in config):

- [x] API server audit logging enabled
- [x] etcd encryption at rest
- [x] Pod Security Admission (PSA) set to `restricted` for most namespaces
- [x] RBAC enabled (default)
- [x] Service account token volume projection
- [x] Network policies enforced (Canal/Calico)
- [x] Secrets encryption configuration
- [x] Kubelet authentication/authorization configured
- [x] Read-only root filesystem for system pods
- [x] Resource quotas and limit ranges per namespace

RKE2 server config (`/etc/rancher/rke2/config.yaml`):

```yaml
profile: cis-1.23
selinux: true
secrets-encryption: true
write-kubeconfig-mode: "0600"
kube-apiserver-arg:
  - "audit-log-path=/var/log/kubernetes/audit/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "audit-policy-file=/etc/rancher/rke2/audit-policy.yaml"
  - "encryption-provider-config=/etc/rancher/rke2/encryption-config.yaml"
kubelet-arg:
  - "protect-kernel-defaults=true"
  - "streaming-connection-idle-timeout=5m"
```

### 7.4 Additional Security Controls

| Control | Implementation |
|---------|----------------|
| **Image signing and verification** | Cosign / Notation; Harbor content trust; admission controller rejects unsigned images |
| **Runtime security** | NeuVector in enforce mode; detects anomalous container/VM behavior |
| **Admission control** | OPA Gatekeeper or Kyverno policies: no privileged containers, required labels, image source restrictions |
| **Secrets management** | HashiCorp Vault with Kubernetes auth; no secrets in etcd plaintext |
| **Audit logging** | API server audit logs forwarded to SIEM (Splunk/Elasticsearch) |
| **CAC/PKI authentication** | LDAP/SAML integration via Rancher; x509 client certs for kubectl |
| **Vulnerability scanning** | Trivy or Grype in CI/CD pipeline; NeuVector runtime scanning |
| **Host intrusion detection** | AIDE or OSSEC on all nodes |
| **ClamAV** | If required by site policy, deployed as DaemonSet scanning shared volumes |

### 7.5 Data at Rest Encryption

- **etcd:** RKE2 native secrets encryption (`aescbc` or `aesgcm` with FIPS modules)
- **Ceph OSD:** dm-crypt encryption on all OSDs with keys in Vault or KMIP
- **VM disks:** Encrypted at the storage layer (Ceph OSD encryption) or at the guest OS layer (LUKS within the VM)
- **Backup data:** Velero + Restic encrypts backup data at rest

### 7.6 Data in Transit Encryption

- All Kubernetes API traffic: TLS 1.2+ (mTLS between components)
- Ceph inter-node: `ms_cluster_mode = secure` (msgr2 encryption)
- VM-to-VM on overlay: Calico WireGuard encryption or IPsec
- Ingress: TLS termination at ingress controller with DoD-issued certificates

---

## 8. Air-Gap Infrastructure

### 8.1 Content Pipeline Architecture

```
UNCLASSIFIED SIDE              |  CLASSIFICATION BOUNDARY  |  IL5 ENCLAVE
                               |   (Data diode or          |
+-------------------+          |    sneakernet)            |  +-------------------+
| Internet-Connected |         |                           |  | Transfer Station  |
| Staging Server     |-------->|  Removable media /        |->| (scanning &       |
|                    |         |  data diode               |  |  staging)          |
| - Download images  |         |                           |  +--------+----------+
| - Download charts  |         |                           |           |
| - Download RPMs    |         |                           |  +--------v----------+
| - Cosign verify    |         |                           |  | Harbor Registry   |
| - Vulnerability    |         |                           |  | (air-gapped)      |
|   scan             |         |                           |  +--------+----------+
+-------------------+          |                           |           |
                               |                           |  +--------v----------+
                               |                           |  | RKE2 Clusters     |
                               |                           |  | (pull from Harbor) |
                               |                           |  +-------------------+
```

### 8.2 Air-Gap Content Bundle

The following must be mirrored and transferred for initial deployment and each update cycle:

| Content | Source | Format | Tool |
|---------|--------|--------|------|
| RKE2 binaries + images | GitHub releases / Rancher | Tarball (rke2-images.linux-amd64.tar.zst) | `rke2` air-gap docs |
| KubeVirt operator + images | KubeVirt releases | Container images (OCI tarballs) | `skopeo copy` |
| CDI images | KubeVirt CDI releases | Container images | `skopeo copy` |
| Rook-Ceph images | Rook GitHub + quay.io | Container images | `skopeo copy` |
| Rancher + charts | Rancher releases | Helm charts + images | `rancher-images.txt` + `hauler` or `skopeo` |
| Harbor images | Harbor releases | Container images | `skopeo copy` |
| NeuVector images | NeuVector releases | Container images | `skopeo copy` |
| Monitoring stack images | Prometheus, Grafana, etc. | Container images | `skopeo copy` |
| RHEL RPMs | Red Hat CDN | RPM repository | `reposync` |
| Helm charts | Various | `.tgz` chart archives | `helm pull` |
| Velero + plugins | Velero releases | Container images + CLI | `skopeo copy` |

### 8.3 Hauler for Content Management

**Hauler** (by Carbide/Rancher) is specifically designed for air-gap content management:

```bash
# On connected side: collect all content
hauler store sync -f manifest.yaml

# Save to transportable archive
hauler store save -f rke2-airgap-bundle.tar.zst

# On air-gapped side: load and serve
hauler store load -f rke2-airgap-bundle.tar.zst
hauler store serve registry   # starts local OCI registry on :5000
```

### 8.4 Harbor Registry Configuration

Harbor serves as the permanent, authoritative container image registry inside the enclave:

- **Replication:** Content pushed from transfer station to Harbor
- **Vulnerability scanning:** Trivy scanner integrated (database updated via air-gap)
- **Content trust:** Notary/Cosign signature verification
- **RBAC:** Project-level access control, integrated with LDAP
- **Storage backend:** S3 (Ceph RGW) or filesystem
- **HA:** Deploy Harbor in HA mode across management cluster nodes

### 8.5 RPM Repository Mirror

For RHEL host patching:

- Mirror relevant RHEL repos using `reposync` on the connected side
- Transfer via removable media
- Serve via local `httpd` or Satellite server inside enclave
- All nodes configured with `baseurl=https://repo.enclave.local/...`

---

## 9. Migration Strategy

### 9.1 Migration Approach: Phased Lift-and-Shift with Progressive Waves

**Core principle:** Move VMs as-is first (lift-and-shift), optimize/modernize later. This minimizes risk and maintains ATO continuity.

### 9.2 VM Disk Conversion Pipeline

VMware VMs use VMDK format. KubeVirt requires raw or qcow2 (qcow2 preferred for space efficiency). The Containerized Data Importer (CDI) handles this conversion.

**Pipeline:**

```
ESXi/vCenter                    KubeVirt Cluster
+-----------+    +-----------+    +------------------+    +---------------+
| VMDK disk | -> | Export OVA| -> | CDI DataVolume   | -> | PVC (qcow2   |
| (on VMFS) |    | or VMDK   |    | import from      |    |  or raw)      |
+-----------+    +-----------+    | HTTP/Registry/   |    +---------------+
                                  | upload           |
                                  +------------------+
```

**Methods for disk import:**

1. **Direct upload via `virtctl`:**
   ```bash
   virtctl image-upload dv my-vm-boot-disk \
     --size=100Gi \
     --image-path=/data/exports/vm-boot.vmdk \
     --storage-class=ceph-block-ssd \
     --access-mode=ReadWriteMany \
     --insecure
   ```

2. **CDI DataVolume with HTTP source:**
   - Stage VMDKs on an internal HTTP server
   - CDI pulls and converts automatically

   ```yaml
   apiVersion: cdi.kubevirt.io/v1beta1
   kind: DataVolume
   metadata:
     name: app-server-01-boot
   spec:
     source:
       http:
         url: "https://data-staging.enclave.local/exports/app-server-01-boot.vmdk"
     pvc:
       accessModes:
         - ReadWriteMany
       resources:
         requests:
           storage: 100Gi
       storageClassName: ceph-block-ssd
   ```

3. **qemu-img manual conversion (pre-stage):**
   ```bash
   qemu-img convert -f vmdk -O qcow2 vm-boot.vmdk vm-boot.qcow2
   ```

### 9.3 VM Configuration Translation

Each VMware VM definition must be translated to a KubeVirt `VirtualMachine` manifest. Key mappings:

| VMware Setting | KubeVirt Equivalent |
|---------------|---------------------|
| vCPU count | `spec.domain.cpu.cores` |
| Memory (MB) | `spec.domain.resources.requests.memory` |
| NIC (VMXNET3) | `spec.domain.devices.interfaces` (virtio recommended) |
| SCSI controller | `spec.domain.devices.disks[].disk.bus: virtio` (or scsi) |
| CD-ROM / ISO | `spec.domain.devices.disks[].cdrom` with `DataVolume` or `containerDisk` |
| UEFI boot | `spec.domain.firmware.bootloader.efi` |
| BIOS boot | `spec.domain.firmware.bootloader.bios` (default) |
| VMware Tools | Replace with `qemu-guest-agent` (install in guest before or after migration) |
| DRS affinity | `nodeSelector`, `affinity`, `tolerations` in pod spec |
| Resource pool | Kubernetes `ResourceQuota` and `LimitRange` per namespace |
| vSphere HA | Kubernetes pod rescheduling (automatic) + KubeVirt `evictionStrategy: LiveMigrate` |
| Snapshot | `VirtualMachineSnapshot` CR (KubeVirt snapshot API) |
| vMotion | KubeVirt live migration (`virtctl migrate <vm>`) |

### 9.4 Guest OS Preparation (Pre-Migration)

Before migrating each VM:

1. **Install `qemu-guest-agent`:**
   - RHEL/CentOS: `yum install qemu-guest-agent && systemctl enable --now qemu-guest-agent`
   - Windows: Install virtio-win drivers + QEMU guest agent MSI
   - This enables Kubernetes to report VM IP, perform graceful shutdown, etc.

2. **Install VirtIO drivers (Windows only):**
   - Mount `virtio-win.iso` inside the VM
   - Install VirtIO SCSI, VirtIO Network, VirtIO Balloon, VirtIO Serial drivers
   - **Critical:** Without VirtIO drivers, Windows VMs will not boot on KubeVirt with virtio bus

3. **Remove VMware Tools:**
   - Uninstall VMware Tools (or let it become inert; it won't cause harm but wastes resources)

4. **Verify network configuration:**
   - Ensure static IPs are configured at the OS level (not relying on VMware customization spec)
   - Update `/etc/fstab` if referencing VMware PVSCSI device paths (should use UUID or LVM)

5. **Document current state:**
   - Screenshot/export current IP config, routes, firewall rules, services, cron jobs

### 9.5 Windows VM Specific Considerations

Windows workloads require extra attention:

- **VirtIO drivers:** Must be installed BEFORE migration or the VM will blue-screen on boot.
- **Activation:** Windows KMS activation must point to enclave KMS server (unchanged if IPs are preserved).
- **Licensing:** Verify Windows Server Datacenter licensing covers KVM hypervisor (Microsoft licenses per physical host; KubeVirt/KVM qualifies for Datacenter unlimited VMs per host).
- **Hyper-V enlightenments:** KubeVirt supports Hyper-V enlightenments for Windows performance. Enable in VM spec:
  ```yaml
  domain:
    features:
      hyperv:
        relaxed: {}
        vapic: {}
        spinlocks:
          spinlocks: 8191
        vpindex: {}
        runtime: {}
        synic: {}
        stimer: {}
        reset: {}
        frequencies: {}
  ```

---

## 10. Migration Execution Plan

### 10.1 Migration Waves

Organize VMs into migration waves of 15-25 VMs each, grouped by dependency and risk:

| Wave | VMs | Category | Duration | Description |
|------|-----|----------|----------|-------------|
| **Wave 0 (Pilot)** | 5-10 | Low-risk, non-critical | 2 weeks | Proof of concept; test tooling, process, rollback |
| **Wave 1** | 15-20 | Dev/test environments | 2 weeks | Build confidence; validate at moderate scale |
| **Wave 2** | 20-30 | Tier 1 stateless web/app | 2 weeks | Stateless apps with easy rollback |
| **Wave 3** | 20-30 | Tier 2 stateful apps | 3 weeks | Application servers with state; more testing required |
| **Wave 4** | 25-30 | Tier 2/3 mixed workloads | 3 weeks | Mix of app servers and smaller databases |
| **Wave 5** | 25-30 | Tier 3 databases | 3 weeks | Databases require extended soak testing |
| **Wave 6** | 20-25 | Tier 4 infrastructure services | 3 weeks | AD, DNS, LDAP -- highest dependency; migrate last |
| **Wave 7** | 15-20 | Tier 5 special/legacy | 3 weeks | Legacy OS, GPU, high-complexity workloads |
| **Wave 8 (Cleanup)** | Remaining | Stragglers, deferred VMs | 2 weeks | Final migrations, decommission VMware |

### 10.2 Per-VM Migration Procedure

For each VM in a wave:

#### Pre-Migration (T-7 days)
1. Verify VM is in discovery inventory with complete metadata
2. Confirm application owner sign-off for migration window
3. Install qemu-guest-agent (and VirtIO drivers for Windows)
4. Take a VMware snapshot as rollback point
5. Generate KubeVirt VM manifest (YAML) from VMware config
6. Validate manifest in review (GitOps PR)
7. Pre-create PVC / DataVolume for disk import
8. Begin disk copy if VM is large (background pre-sync)

#### Migration (T-0)
1. Notify stakeholders; begin maintenance window
2. Gracefully shut down VM in VMware
3. Export final VMDK (or use last pre-synced copy + delta if tooling supports it)
4. Upload VMDK to KubeVirt via CDI (virtctl image-upload or DataVolume)
5. Apply KubeVirt VirtualMachine manifest
6. Start the VM: `virtctl start <vm-name>`
7. Verify boot, network connectivity, application health
8. Run application-specific smoke tests
9. Update DNS records if IP changed
10. Monitor for 30-60 minutes

#### Post-Migration (T+1 to T+7)
1. Extended monitoring (72-hour soak minimum for production workloads)
2. Application owner validation and sign-off
3. Update CMDB/inventory records
4. Update monitoring dashboards and alerts
5. Remove VMware snapshot (after soak period)
6. Document any issues in migration log

#### Rollback Procedure
If migration fails at any point:
1. Stop KubeVirt VM: `virtctl stop <vm-name>`
2. Revert to VMware snapshot
3. Power on VMware VM
4. Restore DNS if changed
5. Document failure reason; update migration plan

### 10.3 Automation and Tooling

Build automation to reduce per-VM manual effort:

| Tool | Purpose |
|------|---------|
| **PowerCLI / govc** | Export VM inventory, VMDK export |
| **Custom Python/Ansible** | Generate KubeVirt YAML manifests from VMware metadata |
| **qemu-img** | VMDK to qcow2 conversion |
| **virtctl** | Image upload, VM lifecycle management |
| **Ansible playbooks** | Guest OS preparation (install qemu-guest-agent, VirtIO drivers) |
| **GitOps (Fleet/ArgoCD)** | Version-controlled VM manifest deployment |
| **Validation scripts** | Post-migration health checks per application |

---

## 11. ATO Continuity Plan

### 11.1 Approach: Continuous ATO with POA&M

Replacing the entire virtualization platform is a significant change. The ATO strategy is:

1. **Do NOT let the ATO lapse.** Operate both platforms in parallel during transition.
2. **File a Significant Change Request (SCR)** with the Authorizing Official (AO) before beginning migration.
3. **Update the System Security Plan (SSP)** to document the target architecture.
4. **Create POA&Ms** for any controls not yet fully implemented on the new platform.
5. **Conduct a delta assessment** (not a full assessment) once migration is complete.

### 11.2 Security Documentation Updates

| Document | Required Updates |
|----------|-----------------|
| **System Security Plan (SSP)** | Update architecture diagrams, component inventory, boundary definition, control implementations (CM, SC, AC families primarily) |
| **Security Control Assessment (SCA)** | New assessment procedures for Kubernetes, KubeVirt, Ceph; map to existing NIST 800-53 Rev 5 controls |
| **POA&M** | Track any gaps; e.g., if KubeVirt snapshot capability is less mature than VMware's, document timeline to remediation |
| **Hardware/Software Inventory** | Remove VMware components; add RKE2, KubeVirt, Ceph, Harbor, etc. |
| **Network Diagram** | Update to reflect new network architecture |
| **Data Flow Diagram** | Update to reflect new storage and inter-component flows |
| **Ports, Protocols, and Services (PPS)** | Update for Kubernetes API (6443), etcd (2379-2380), Kubelet (10250), Ceph (6789, 3300, 6800-7300), etc. |
| **Configuration Management Plan** | Document GitOps-based configuration management approach |
| **Incident Response Plan** | Update procedures for Kubernetes-specific incidents |
| **Contingency Plan** | Update backup and recovery procedures for KubeVirt |

### 11.3 Control Family Impact Assessment

| NIST 800-53 Family | Impact Level | Key Changes |
|--------------------|--------------|----|
| **AC (Access Control)** | Medium | New RBAC model (Kubernetes RBAC + Rancher); CAC integration via SAML/LDAP |
| **AU (Audit)** | Medium | Kubernetes audit logs; Ceph audit; new log pipeline |
| **CM (Configuration Management)** | High | Entirely new baseline; STIG for Kubernetes; GitOps model |
| **CP (Contingency Planning)** | Medium | New backup tooling (Velero); updated DR procedures |
| **IA (Identification and Authentication)** | Low | Same identity providers; new service account model |
| **IR (Incident Response)** | Low | Updated runbooks; same SIEM integration |
| **MA (Maintenance)** | Medium | New patching process (air-gap image updates); rolling upgrades |
| **PE (Physical)** | None | No change (same hardware/facility) |
| **PL (Planning)** | Low | Updated SSP |
| **RA (Risk Assessment)** | Medium | New risk assessment for Kubernetes/KubeVirt platform |
| **SA (System Acquisition)** | Low | New vendor relationships (SUSE/Rancher if support purchased) |
| **SC (System and Communications Protection)** | High | New encryption implementations; new boundary protections; new segmentation model |
| **SI (System and Information Integrity)** | Medium | New vulnerability management for containers; new STIG scanning |

### 11.4 Compliance Scanning and Continuous Monitoring

| Scan Type | Tool | Frequency |
|-----------|------|-----------|
| OS STIG compliance | OpenSCAP / SCAP Compliance Checker (SCC) | Weekly |
| Kubernetes STIG | Kube-bench + custom inspec profiles | Weekly |
| Container image CVE scan | Trivy (in Harbor) + NeuVector | On import + daily |
| CIS Kubernetes Benchmark | kube-bench | Weekly |
| Runtime anomaly detection | NeuVector | Continuous |
| Configuration drift | GitOps (Fleet/ArgoCD) reconciliation | Continuous |
| Network policy audit | Calico policy audit logs | Continuous |
| etcd encryption verification | RKE2 built-in | On startup + audit |

---

## 12. Operational Runbook Considerations

### 12.1 Day 2 Operations Comparison

| Operation | VMware (Current) | KubeVirt (Target) |
|-----------|-------------------|-------------------|
| **Start/Stop VM** | vCenter UI or PowerCLI | `virtctl start/stop <vm>` or Rancher UI |
| **Console access** | vCenter web console | `virtctl console <vm>` (serial) or `virtctl vnc <vm>` (graphical) |
| **Live migration** | vMotion (automatic DRS) | `virtctl migrate <vm>` or eviction-triggered |
| **Snapshot** | vCenter snapshot manager | `VirtualMachineSnapshot` CR |
| **Clone VM** | vCenter clone wizard | `VirtualMachine` manifest copy + CDI clone |
| **Resize (CPU/RAM)** | Edit VM settings (requires reboot for hot-add if not configured) | Edit VM spec, restart if needed (hotplug CPU/memory is limited) |
| **Add disk** | Add virtual disk in vCenter | Create PVC, add to VM spec, restart |
| **Monitoring** | vROps, esxtop | Prometheus + Grafana; `kubectl top`; KubeVirt metrics |
| **Alerting** | vROps alerts | Alertmanager rules |
| **Backup** | Veeam/Commvault | Velero + VolumeSnapshot |
| **Patching (host)** | ESXi patch via VUM | OS + RKE2 rolling upgrade; `kubectl drain` + upgrade + uncordon |
| **Patching (guest)** | Unchanged | Unchanged (WSUS/Satellite/Ansible) |

### 12.2 Monitoring Stack

Deploy Rancher Monitoring (Prometheus Operator + Grafana) with:

- **Node metrics:** node-exporter DaemonSet
- **Kubernetes metrics:** kube-state-metrics
- **KubeVirt metrics:** KubeVirt Prometheus rules (built-in to operator)
- **Ceph metrics:** Rook-Ceph Prometheus integration
- **Custom dashboards:**
  - VM resource utilization (CPU, memory, disk IOPS, network)
  - VM migration status
  - Storage capacity and performance
  - Cluster health overview

### 12.3 Backup and Disaster Recovery

| Component | Backup Method | RPO | RTO |
|-----------|---------------|-----|-----|
| **etcd (cluster state)** | RKE2 automatic etcd snapshots + Velero | 1 hour | 30 min |
| **VM disks** | Velero with CSI VolumeSnapshot | Per SLA (4-24 hours) | Per SLA (1-4 hours) |
| **VM configurations** | GitOps (all manifests in Git) | Near-zero (Git commit) | Minutes (apply manifests) |
| **Harbor registry** | Velero + PVC backup | Daily | 2 hours |
| **Ceph cluster** | Ceph RBD mirroring or Rook-Ceph backup | Per pool config | Varies |

### 12.4 Capacity Management

- **Cluster autoscaling:** Not applicable (bare metal); manual node addition process documented.
- **Resource overcommit:** KubeVirt supports memory overcommit but not recommended for IL5 production. Use guaranteed QoS class for all VMs.
- **Namespace quotas:** Enforce per-project resource limits to prevent noisy neighbors.
- **Capacity alerts:** Prometheus alerts when cluster reaches 70% and 85% utilization thresholds.

---

## 13. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|------------|--------|------------|
| R-01 | KubeVirt performance degradation vs. bare-metal ESXi for specific workloads | Medium | High | Run performance benchmarks during Wave 0 pilot; use SR-IOV or PCI passthrough for latency-sensitive workloads |
| R-02 | Windows VM boot failure due to missing VirtIO drivers | High | Medium | Mandatory pre-migration checklist; test with pilot Windows VMs first |
| R-03 | Storage performance insufficient during migration (dual-write to SAN) | Medium | High | Schedule migrations during low-utilization windows; stage disk copies in advance |
| R-04 | ATO revocation during transition | Low | Critical | File SCR early; maintain parallel environments; continuous communication with AO/ISSM |
| R-05 | Operational team lacks Kubernetes expertise | High | High | Training plan (see Section 14); SUSE/Rancher professional services; phased transition with extended parallel ops |
| R-06 | Live migration instability with Ceph RWX | Medium | Medium | Test extensively in pilot; fall back to cold migration if needed; Ceph tuning |
| R-07 | Air-gap content pipeline introduces vulnerability | Low | High | All images scanned on connected side AND inside enclave; signed images only |
| R-08 | Legacy OS (RHEL 6, Win 2012) incompatible with VirtIO | Medium | Medium | Use IDE/SATA bus emulation for legacy VMs; accept performance tradeoff |
| R-09 | Extended migration timeline impacts operations | Medium | Medium | Limit maintenance windows; parallel operation; rollback capability |
| R-10 | Ceph operational complexity exceeds team capacity | Medium | High | Consider Longhorn as simpler alternative; SUSE support contract; training |
| R-11 | License audit: Microsoft licensing on KVM | Low | Medium | Verify SPLA or Datacenter licensing covers KVM; consult Microsoft LAR |
| R-12 | Hardware failure during migration window | Low | High | Maintain N+2 spare capacity on both platforms during migration |

---

## 14. Resource Requirements

### 14.1 Team Composition

| Role | FTE | Duration | Responsibilities |
|------|-----|----------|-----------------|
| **Solution Architect** | 1 | Entire project | Architecture decisions, ATO documentation, technical lead |
| **Kubernetes Engineer (Senior)** | 2 | Entire project | Cluster deployment, KubeVirt configuration, automation |
| **Kubernetes Engineer (Mid)** | 2 | Migration waves | VM migration execution, troubleshooting |
| **Storage Engineer** | 1 | Months 1-6 | Ceph deployment and optimization |
| **Network Engineer** | 1 | Months 1-4 | VLAN setup, Multus config, firewall updates |
| **VMware Admin (existing)** | 1-2 | Entire project | VM export, VMware decommission, current-state SME |
| **Security/ISSO** | 1 | Entire project | STIG compliance, SSP updates, AO coordination |
| **Application Owners** | As needed | Per wave | Validation, testing, sign-off |
| **Project Manager** | 1 | Entire project | Schedule, risk, communications |

### 14.2 Training Plan

| Topic | Audience | Duration | Delivery |
|-------|----------|----------|----------|
| Kubernetes Fundamentals (CKA-level) | All engineering staff | 40 hours | Instructor-led or CBT |
| RKE2 Administration | Platform engineers | 16 hours | SUSE official training |
| KubeVirt Operations | Platform engineers | 16 hours | Hands-on lab |
| Ceph/Rook Administration | Storage engineers | 24 hours | Instructor-led |
| GitOps with Fleet/ArgoCD | All engineering staff | 8 hours | Workshop |
| Container Security with NeuVector | Security staff | 8 hours | SUSE official training |

### 14.3 Hardware Budget Estimate

| Item | Qty | Unit Cost (est.) | Total (est.) |
|------|-----|-------------------|-------------|
| Worker nodes (2U, 128 core, 1TB RAM, NVMe) | 14 | $35,000 | $490,000 |
| Control plane nodes (1U, 32 core, 128GB RAM) | 6 | $12,000 | $72,000 |
| Storage nodes (2U, 64 core, 256GB RAM, 10x 4TB NVMe) | 5 | $45,000 | $225,000 |
| 100GbE ToR switches | 4 | $25,000 | $100,000 |
| 25GbE NICs (dual-port per node) | 25 | $800 | $20,000 |
| Spare hardware (10%) | -- | -- | $90,700 |
| **Subtotal Hardware** | | | **$997,700** |
| SUSE Rancher support (optional, 3-year) | 1 | $150,000 | $150,000 |
| Training | -- | -- | $80,000 |
| Professional services (SUSE/partner) | -- | -- | $200,000 |
| **Total Estimated Budget** | | | **~$1,427,700** |

**Note:** Compare against VMware licensing renewal costs. Broadcom's per-core licensing for 200 VMs on 12-16 hosts typically runs $500K-$1.5M/year, making this migration cost-neutral within 1-2 years.

If existing server hardware is being reused (repurposing ESXi hosts), the hardware line items decrease significantly -- only storage nodes and additional NICs/switches may be needed.

---

## 15. Timeline & Milestones

| Phase | Duration | Milestone | Exit Criteria |
|-------|----------|-----------|---------------|
| **Phase 0: Discovery & Planning** | 6 weeks | Discovery complete, architecture approved | VM inventory, dependency map, architecture signoff, AO notified |
| **Phase 1: Infrastructure Build** | 6 weeks | Platform operational | RKE2 clusters deployed, KubeVirt operational, storage provisioned, Harbor populated, monitoring live |
| **Phase 2: Security Hardening & Validation** | 4 weeks | STIG compliant, AO briefed | All STIGs applied, SCC scans passing, SSP updated, SCR filed |
| **Phase 3: Wave 0 Pilot** | 3 weeks | Pilot VMs migrated and validated | 5-10 VMs running on KubeVirt, performance validated, runbooks drafted |
| **Phase 4: Waves 1-3 (Low/Med Risk)** | 8 weeks | ~75 VMs migrated | All low and medium risk VMs migrated with application owner signoff |
| **Phase 5: Waves 4-6 (Med/High Risk)** | 9 weeks | ~150 VMs migrated | Databases and infrastructure VMs migrated with extended soak |
| **Phase 6: Waves 7-8 (Special/Cleanup)** | 5 weeks | All VMs migrated | All 200 VMs on KubeVirt; VMware powered down |
| **Phase 7: VMware Decommission** | 3 weeks | VMware removed | ESXi hosts repurposed or decommissioned; licenses terminated |
| **Phase 8: ATO Delta Assessment** | 4 weeks | ATO reauthorized | Delta assessment complete; AO signs updated ATO |
| **Total** | **~48 weeks (~12 months)** | | |

### Gantt Chart (Simplified)

```
Month:     1    2    3    4    5    6    7    8    9    10   11   12
           |    |    |    |    |    |    |    |    |    |    |    |
Phase 0:   [=====]
Phase 1:        [========]
Phase 2:             [======]
Phase 3:                  [====]
Phase 4:                      [===============]
Phase 5:                                [================]
Phase 6:                                            [========]
Phase 7:                                                  [====]
Phase 8:                                                     [======]
```

---

## 16. Appendices

### Appendix A: Key Ports, Protocols, and Services

| Component | Port | Protocol | Direction | Purpose |
|-----------|------|----------|-----------|---------|
| RKE2 API Server | 6443 | TCP | Inbound | Kubernetes API |
| RKE2 Supervisor | 9345 | TCP | Inbound | Node join |
| etcd | 2379-2380 | TCP | Internal | etcd client + peer |
| Kubelet | 10250 | TCP | Internal | Kubelet API |
| Canal/Calico VXLAN | 4789 | UDP | Internal | Pod overlay network |
| Canal/Calico BGP | 179 | TCP | Internal | BGP peering (if used) |
| Canal/Calico Typha | 5473 | TCP | Internal | Calico Typha |
| CoreDNS | 53 | TCP/UDP | Internal | Cluster DNS |
| Harbor | 443 | TCP | Internal | Container registry |
| Ceph Monitor | 3300, 6789 | TCP | Internal | Ceph monitor |
| Ceph OSD | 6800-7300 | TCP | Internal | Ceph OSD communication |
| Prometheus | 9090 | TCP | Internal | Metrics |
| Grafana | 3000 | TCP | Internal | Dashboards |
| NodePort range | 30000-32767 | TCP | Internal | Kubernetes NodePort services |
| MetalLB | Configurable | TCP/UDP | Internal | Load balancer VIPs |
| KubeVirt VNC | 5900+ | TCP | Internal | VM console (via virtctl proxy) |
| Rancher UI | 443 | TCP | Inbound | Management UI |

### Appendix B: Sample KubeVirt VM Manifest (Linux)

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: app-server-01
  namespace: production-apps
  labels:
    app: myapp
    tier: frontend
    migration-wave: "2"
spec:
  running: true
  template:
    metadata:
      labels:
        app: myapp
        tier: frontend
        kubevirt.io/vm: app-server-01
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
        resources:
          requests:
            memory: 8Gi
          limits:
            memory: 8Gi
        devices:
          disks:
            - name: boot-disk
              disk:
                bus: virtio
            - name: cloudinit
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
            - name: app-network
              bridge: {}
          rng: {}
        machine:
          type: q35
      networks:
        - name: default
          pod: {}
        - name: app-network
          multus:
            networkName: production-apps/vlan50-app-network
      volumes:
        - name: boot-disk
          persistentVolumeClaim:
            claimName: app-server-01-boot
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              hostname: app-server-01
              ssh_authorized_keys:
                - ssh-rsa AAAA...
      evictionStrategy: LiveMigrate
      terminationGracePeriodSeconds: 180
```

### Appendix C: Sample KubeVirt VM Manifest (Windows)

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: win-server-01
  namespace: windows-workloads
spec:
  running: true
  template:
    spec:
      domain:
        clock:
          timer:
            hpet:
              present: false
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
            hyperv: {}
          utc: {}
        cpu:
          cores: 8
          sockets: 1
          threads: 1
        resources:
          requests:
            memory: 16Gi
          limits:
            memory: 16Gi
        features:
          acpi: {}
          apic: {}
          hyperv:
            relaxed: {}
            vapic: {}
            spinlocks:
              spinlocks: 8191
            vpindex: {}
            runtime: {}
            synic: {}
            stimer:
              direct: {}
            reset: {}
            frequencies: {}
            tlbflush: {}
            ipi: {}
        devices:
          disks:
            - name: boot-disk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
          tpm: {}
        machine:
          type: q35
      networks:
        - name: default
          pod: {}
      volumes:
        - name: boot-disk
          persistentVolumeClaim:
            claimName: win-server-01-boot
      evictionStrategy: LiveMigrate
      terminationGracePeriodSeconds: 3600
```

### Appendix D: Air-Gap Content Hauler Manifest

```yaml
apiVersion: content.hauler.cattle.io/v1alpha1
kind: Images
metadata:
  name: rke2-kubevirt-stack
spec:
  images:
    # RKE2 core (version-specific; update per release)
    - name: rancher/rke2-runtime:v1.28.5-rke2r1
    # KubeVirt
    - name: quay.io/kubevirt/virt-operator:v1.2.0
    - name: quay.io/kubevirt/virt-api:v1.2.0
    - name: quay.io/kubevirt/virt-controller:v1.2.0
    - name: quay.io/kubevirt/virt-handler:v1.2.0
    - name: quay.io/kubevirt/virt-launcher:v1.2.0
    # CDI
    - name: quay.io/kubevirt/cdi-operator:v1.58.0
    - name: quay.io/kubevirt/cdi-controller:v1.58.0
    - name: quay.io/kubevirt/cdi-importer:v1.58.0
    - name: quay.io/kubevirt/cdi-uploadproxy:v1.58.0
    - name: quay.io/kubevirt/cdi-apiserver:v1.58.0
    # Rook-Ceph
    - name: rook/ceph:v1.13.0
    - name: quay.io/ceph/ceph:v18.2
    # Multus
    - name: ghcr.io/k8snetworkplumbingwg/multus-cni:v4.0.2
    # Rancher
    - name: rancher/rancher:v2.8.2
    # Monitoring
    - name: rancher/mirrored-prometheus-operator:v0.70.0
    - name: grafana/grafana:10.2.3
    # NeuVector
    - name: neuvector/controller:5.3.0
    - name: neuvector/enforcer:5.3.0
    - name: neuvector/manager:5.3.0
    - name: neuvector/scanner:latest
    # Harbor
    - name: goharbor/harbor-core:v2.10.0
    - name: goharbor/harbor-portal:v2.10.0
    - name: goharbor/harbor-registryctl:v2.10.0
    # Velero
    - name: velero/velero:v1.13.0
```

### Appendix E: Glossary

| Term | Definition |
|------|-----------|
| **AO** | Authorizing Official |
| **ATO** | Authorization to Operate |
| **CDI** | Containerized Data Importer (KubeVirt component for importing disk images) |
| **CIS** | Center for Internet Security |
| **CMDB** | Configuration Management Database |
| **CNI** | Container Network Interface |
| **CUI** | Controlled Unclassified Information |
| **DISA** | Defense Information Systems Agency |
| **DRS** | Distributed Resource Scheduler (VMware) |
| **FIPS** | Federal Information Processing Standards |
| **IL5** | Impact Level 5 (DoD Cloud Computing SRG) |
| **ISSO** | Information System Security Officer |
| **KVM** | Kernel-based Virtual Machine |
| **NAD** | NetworkAttachmentDefinition (Multus) |
| **OCI** | Open Container Initiative |
| **OSD** | Object Storage Daemon (Ceph) |
| **POA&M** | Plan of Action and Milestones |
| **PVC** | PersistentVolumeClaim (Kubernetes) |
| **RBAC** | Role-Based Access Control |
| **RKE2** | Rancher Kubernetes Engine 2 |
| **RWX** | ReadWriteMany (storage access mode) |
| **SCR** | Significant Change Request |
| **SR-IOV** | Single Root I/O Virtualization |
| **SSP** | System Security Plan |
| **STIG** | Security Technical Implementation Guide |
| **VCSA** | vCenter Server Appliance |
| **VMDK** | Virtual Machine Disk (VMware format) |

---

*End of document.*
