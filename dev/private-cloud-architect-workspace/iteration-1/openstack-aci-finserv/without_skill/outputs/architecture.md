# Private Cloud Architecture for Financial Services
## OpenStack + Cisco ACI + Ceph | PCI-DSS & SOC 2 Compliant

**Version:** 1.0
**Date:** 2026-03-20
**Target Environment:** ~500 VMs, ~500 employees, 6-person ops team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Design Principles](#2-design-principles)
3. [Physical Infrastructure](#3-physical-infrastructure)
4. [Network Architecture (Cisco ACI)](#4-network-architecture-cisco-aci)
5. [Compute Architecture (OpenStack Nova)](#5-compute-architecture-openstack-nova)
6. [Storage Architecture (Ceph)](#6-storage-architecture-ceph)
7. [Identity & Access Management](#7-identity--access-management)
8. [Monitoring, Logging & Alerting](#8-monitoring-logging--alerting)
9. [Automation & Infrastructure as Code](#9-automation--infrastructure-as-code)
10. [Security & Compliance (PCI-DSS / SOC 2)](#10-security--compliance-pci-dss--soc-2)
11. [Disaster Recovery & Business Continuity](#11-disaster-recovery--business-continuity)
12. [Operational Procedures](#12-operational-procedures)
13. [Capacity Planning & Growth](#13-capacity-planning--growth)
14. [Risk Register](#14-risk-register)
15. [Bill of Materials Summary](#15-bill-of-materials-summary)

---

## 1. Executive Summary

This document describes the architecture for a private cloud platform built on **OpenStack** (compute orchestration), **Cisco ACI** (software-defined networking), and **Ceph** (distributed storage). The platform is designed to host approximately 500 virtual machines for a 500-employee financial services company while maintaining full compliance with **PCI-DSS v4.0** and **SOC 2 Type II**.

The architecture prioritizes:

- **Security by default** -- network microsegmentation, encryption at rest and in transit, least-privilege access
- **Operational simplicity** -- automated provisioning, centralized monitoring, GitOps workflows suitable for a 6-person ops team
- **Resilience** -- no single point of failure at any layer, N+1 or better redundancy
- **Compliance auditability** -- immutable logs, automated compliance scanning, clear separation of duties

### Key Technology Choices

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Compute orchestration | OpenStack (Caracal/2024.1+) | Mature, vendor-neutral, strong financial services adoption |
| Networking / SDN | Cisco ACI with Nexus 9000 | Already purchased; microsegmentation, policy-based networking |
| Storage | Ceph (Reef or later) | Unified block/object/file; no licensing cost; proven at scale |
| Identity | FreeIPA + Keycloak (OIDC/SAML) | Centralized RBAC, MFA, federation with existing AD if needed |
| Monitoring | Prometheus + Grafana + Loki + Alertmanager | Open-source, scalable, well-integrated |
| Automation | Ansible + Terraform + ArgoCD | GitOps, idempotent, auditable |
| Secrets | HashiCorp Vault | Dynamic secrets, PKI, encryption-as-a-service |

---

## 2. Design Principles

1. **Defense in depth** -- Every layer assumes the layer above it is compromised.
2. **Infrastructure as Code (IaC)** -- Nothing is configured manually in production. All changes go through version-controlled pipelines.
3. **Immutable infrastructure where possible** -- Base images are built in CI, VMs are redeployed rather than patched in place for stateless workloads.
4. **Least privilege everywhere** -- Service accounts, network policies, and user roles are scoped to the minimum required.
5. **Observable by default** -- Every component emits metrics, logs, and traces. Alerts are actionable.
6. **N+1 redundancy minimum** -- No single hardware failure causes service degradation.
7. **Separation of control and data planes** -- Management traffic, tenant traffic, storage replication traffic, and out-of-band management each travel on separate networks.
8. **Compliance as code** -- PCI-DSS and SOC 2 controls are validated automatically and continuously.

---

## 3. Physical Infrastructure

### 3.1 Data Center Requirements

- **Primary site:** Single data center with two independent power feeds (A+B)
- **Minimum rack space:** 8 racks (42U each)
- **Power:** Dual-fed 30A 208V circuits per rack, estimated 8-10 kW per rack
- **Cooling:** N+1 precision cooling, hot-aisle/cold-aisle containment
- **Physical security:** Mantrap entry, biometric + badge access, 24/7 CCTV with 90-day retention (PCI-DSS requirement)

### 3.2 Rack Layout

```
Rack 1-2:  Network & Management
  - Cisco Nexus 9500 Spine switches (2x)
  - Cisco APIC controllers (3x)
  - Out-of-band management switches (2x)
  - Console servers (2x)
  - Management servers (3x -- OpenStack control plane)
  - Patch panels and cable management

Rack 3-6:  Compute
  - 16x compute nodes (4 per rack)
  - Top-of-rack Nexus 9300 leaf switches (2 per rack)

Rack 7-8:  Storage (Ceph)
  - 8x Ceph OSD nodes (4 per rack)
  - 3x Ceph MON/MGR/MDS nodes (split across racks)
  - Top-of-rack Nexus 9300 leaf switches (2 per rack)
```

### 3.3 Server Specifications

#### Control Plane Nodes (3x)

| Component | Specification |
|-----------|--------------|
| Chassis | 1U rack-mount (e.g., Dell PowerEdge R660 or HPE DL360 Gen11) |
| CPU | 2x Intel Xeon Gold 6430 (32C/64T each) |
| RAM | 512 GB DDR5 ECC |
| Boot/OS | 2x 960 GB NVMe SSD (RAID 1) |
| Data | 2x 3.84 TB NVMe SSD (for MariaDB Galera, RabbitMQ, etcd) |
| NIC | 2x 25 GbE dual-port (4 ports total, bonded) |
| BMC | iDRAC/iLO with dedicated 1GbE OOB port |

#### Compute Nodes (16x)

| Component | Specification |
|-----------|--------------|
| Chassis | 2U rack-mount (e.g., Dell PowerEdge R760 or HPE DL380 Gen11) |
| CPU | 2x Intel Xeon Gold 6448Y (32C/64T each) or AMD EPYC 9454 (48C/96T) |
| RAM | 1 TB DDR5 ECC (supports ~30 VMs per node at avg 32 GB each) |
| Boot/OS | 2x 480 GB SATA SSD (RAID 1) |
| Local ephemeral | 2x 3.84 TB NVMe SSD (Nova local ephemeral, optional) |
| NIC | 2x 25 GbE dual-port (4 ports total) |
| BMC | Dedicated 1 GbE OOB port |

**Capacity math:** 16 nodes x 1 TB RAM = 16 TB total. At 1.5:1 memory overcommit ratio and 32 GB average per VM: ~750 VM capacity. This gives comfortable headroom above the 500 VM target with N+1 tolerance (can lose 1-2 nodes and still run all VMs).

#### Ceph OSD Nodes (8x)

| Component | Specification |
|-----------|--------------|
| Chassis | 2U rack-mount, 12-bay or 24-bay front-loading |
| CPU | 2x Intel Xeon Silver 4416+ (20C/40T each) |
| RAM | 256 GB DDR5 ECC (Ceph guidance: ~5 GB per OSD) |
| OS | 2x 480 GB SATA SSD (RAID 1) |
| OSD drives (data) | 12x 7.68 TB NVMe SSD per node |
| WAL/DB | 2x 1.6 TB NVMe SSD (partitioned for BlueStore WAL+DB) |
| NIC | 2x 25 GbE dual-port (4 ports: 2 for cluster/replication, 2 for public/client) |
| BMC | Dedicated 1 GbE OOB port |

**Capacity math:** 8 nodes x 12 drives x 7.68 TB = 737 TB raw. At 3x replication = ~245 TB usable. At 2x replication with EC pools for cold data, usable capacity can reach ~350 TB.

#### Ceph MON/MGR/MDS Nodes (3x)

| Component | Specification |
|-----------|--------------|
| Chassis | 1U rack-mount |
| CPU | 1x Intel Xeon Gold 6430 |
| RAM | 128 GB DDR5 ECC |
| OS + data | 2x 960 GB NVMe SSD (RAID 1) |
| NIC | 2x 25 GbE dual-port |

---

## 4. Network Architecture (Cisco ACI)

### 4.1 ACI Fabric Topology

The network follows a **leaf-spine** architecture managed by Cisco ACI (Application Policy Infrastructure Controller).

```
                    +-----------+     +-----------+
                    |  Spine 1  |     |  Spine 2  |
                    | N9500-FM  |     | N9500-FM  |
                    +-----+-----+     +-----+-----+
                          |                 |
          +-------+-------+-------+---------+-------+-------+
          |       |       |       |         |       |       |
       +--+--+ +--+--+ +--+--+ +--+--+  +--+--+ +--+--+ +--+--+
       |Leaf1| |Leaf2| |Leaf3| |Leaf4|  |Leaf5| |Leaf6| |Leaf7|
       |9300 | |9300 | |9300 | |9300 |  |9300 | |9300 | |9300 |
       +--+--+ +--+--+ +--+--+ +--+--+  +--+--+ +--+--+ +--+--+
         |       |       |       |         |       |       |
       Mgmt   Compute  Compute Compute  Compute  Ceph   Ceph
       Nodes  Rack 3   Rack 4  Rack 5   Rack 6   Rack7  Rack8
```

- **Spine switches:** 2x Cisco Nexus 9500 (with 100G line cards). Each spine connects to every leaf with 100 GbE uplinks.
- **Leaf switches:** 2x per rack (7 racks = 14 leaf switches), Nexus 9300-FX3 or GX2 with 48x 25GbE server ports + 6x 100GbE uplinks.
- **APIC cluster:** 3x Cisco APIC controllers (physical appliances), connected to management leaf switches.

### 4.2 ACI Logical Design

#### Tenants

| ACI Tenant | Purpose |
|------------|---------|
| `infra` | ACI internal fabric management (built-in) |
| `mgmt` | OpenStack control plane, monitoring, automation |
| `common` | Shared services (DNS, NTP, LDAP, Vault, PKI) |
| `prod` | Production workload VMs |
| `pci` | PCI-scoped workloads (CDE -- Cardholder Data Environment) |
| `dev` | Development/staging workloads |

#### VRFs and Bridge Domains

Each tenant gets its own VRF for complete routing isolation. Within each tenant, bridge domains map to functional subnets:

**Tenant: `mgmt`**
| Bridge Domain | Subnet | Purpose |
|---------------|--------|---------|
| BD-CTRL | 10.10.1.0/24 | OpenStack API endpoints |
| BD-CTRL-INTERNAL | 10.10.2.0/24 | Internal messaging (RabbitMQ, Galera) |
| BD-MGMT-OOB | 10.10.0.0/24 | Out-of-band / BMC |
| BD-MONITOR | 10.10.3.0/24 | Prometheus, Grafana, Loki |

**Tenant: `pci`**
| Bridge Domain | Subnet | Purpose |
|---------------|--------|---------|
| BD-PCI-APP | 10.20.1.0/24 | PCI application tier |
| BD-PCI-DB | 10.20.2.0/24 | PCI database tier |
| BD-PCI-WEB | 10.20.3.0/24 | PCI web/API tier |

**Tenant: `prod`**
| Bridge Domain | Subnet | Purpose |
|---------------|--------|---------|
| BD-PROD-APP | 10.30.1.0/23 | General production applications |
| BD-PROD-DB | 10.30.4.0/24 | Production databases |
| BD-PROD-WEB | 10.30.6.0/24 | Production web tier |

#### Contracts and Filters (Microsegmentation)

ACI contracts enforce allow-list-only communication between EPGs (Endpoint Groups):

```
EPG: PCI-Web  --[contract: allow-https]--> EPG: PCI-App
EPG: PCI-App  --[contract: allow-db]----> EPG: PCI-DB
EPG: PCI-DB   --[contract: deny-all]----> (everything else)

EPG: Monitoring --[contract: allow-metrics]--> EPG: All-Servers (tcp/9090,9100,9093)
EPG: All-Servers --[contract: allow-syslog]--> EPG: Log-Collectors (tcp/514, tcp/5140)
```

**PCI-DSS requirement 1 (network segmentation):** The `pci` tenant VRF is completely isolated. Inter-VRF routing to `pci` is permitted only through a defined L4-L7 service graph with an inline firewall (see 4.4). All ACI contracts for PCI EPGs default to **deny-all** with explicit allow rules.

### 4.3 VLAN and Overlay Design

ACI uses VXLAN internally across the fabric. Server-facing ports use either:

- **Bare-metal / OpenStack integration:** ACI opflex plugin for OpenStack Neutron (the `aci-neutron` mechanism driver) provisions EPGs dynamically when OpenStack networks/ports are created.
- **VLAN pools:** Static VLAN pools for infrastructure services that sit outside OpenStack (BMC, APIC management, storage).

| VLAN Pool | Range | Purpose |
|-----------|-------|---------|
| OOB-MGMT | 100-109 | BMC/iDRAC/iLO |
| INFRA-STATIC | 110-149 | Ceph public, Ceph cluster, OpenStack internal |
| OPENSTACK-DYNAMIC | 200-999 | Dynamically assigned by ACI Neutron plugin |

### 4.4 External Connectivity and Firewalling

```
Internet
    |
[Border Router -- BGP peering with ISP]
    |
[Perimeter Firewall Pair -- Palo Alto PA-3400 HA pair]
    |
[ACI L3Out on Spine -- External EPG]
    |
[ACI Fabric]
```

- **Border routers:** 2x routers with BGP to ISP for public IP space. These sit outside the ACI fabric.
- **Perimeter firewall:** HA pair of next-gen firewalls. Terminates VPN, performs IDS/IPS, enforces north-south policy. This is also the PCI-DSS "firewall at each Internet connection" requirement.
- **ACI L3Out:** Configured on spine switches to peer with the perimeter firewall via OSPF or BGP. External EPGs classify inbound traffic and apply contracts.
- **East-west PCI firewall:** ACI L4-L7 service graph inserts the firewall inline for any traffic entering/leaving the PCI tenant VRF. This satisfies PCI-DSS requirement for segmentation controls between CDE and non-CDE networks.

### 4.5 OpenStack Neutron Integration with ACI

The **Cisco ACI Neutron plugin** (`networking-aci` or `apic_aim` mechanism driver) replaces the default OVS/OVN networking:

- OpenStack networks map to ACI EPGs
- OpenStack security groups map to ACI contracts
- OpenStack routers map to ACI VRF inter-BD routing
- Floating IPs map to ACI external NAT on the L3Out

**Configuration approach:**

```
# /etc/neutron/plugins/ml2/ml2_conf.ini (relevant sections)
[ml2]
type_drivers = opflex, local, vlan
mechanism_drivers = apic_aim
tenant_network_types = opflex

[ml2_apic_aim]
apic_hosts = 10.10.0.11,10.10.0.12,10.10.0.13
apic_username = neutron-svc
apic_password_file = /etc/neutron/apic_password  # or Vault reference
```

The opflex agent runs on each compute node and programs the local OVS bridge to apply ACI policy locally. This means policy enforcement is distributed, not centralized through the spine/leaf.

### 4.6 Network Services

| Service | Implementation | Notes |
|---------|---------------|-------|
| DNS | 2x PowerDNS (authoritative) + Unbound (recursive) | Internal zones: `cloud.internal`, `pci.internal` |
| NTP | 3x Chrony servers synced to stratum-1 GPS or upstream pool | All hosts sync to internal NTP. PCI-DSS req 10.6. |
| Load balancing | OpenStack Octavia (HAProxy-based LBaaS) | For tenant-facing LB. Perimeter LB on firewall or dedicated F5. |
| VPN | WireGuard or IPSec site-to-site on perimeter FW | Remote admin access via VPN only |

---

## 5. Compute Architecture (OpenStack Nova)

### 5.1 OpenStack Release and Deployment

- **Release:** OpenStack 2025.1 (or latest stable at time of deployment)
- **Deployment tool:** **Kolla-Ansible** -- containerized OpenStack services running in Docker on each node, deployed and upgraded via Ansible playbooks.
- **Reason for Kolla-Ansible:** Simplifies upgrades (container image swap), well-suited for teams with Ansible experience, good ACI plugin support, active upstream maintenance.

### 5.2 Control Plane Architecture

Three control plane nodes in an active-active-active configuration:

```
         +--- ctrl-01 ---+    +--- ctrl-02 ---+    +--- ctrl-03 ---+
         | MariaDB Galera |    | MariaDB Galera |    | MariaDB Galera |
         | RabbitMQ       |    | RabbitMQ       |    | RabbitMQ       |
         | Memcached      |    | Memcached      |    | Memcached      |
         | Keystone       |    | Keystone       |    | Keystone       |
         | Nova API       |    | Nova API       |    | Nova API       |
         | Nova Scheduler |    | Nova Scheduler |    | Nova Scheduler |
         | Nova Conductor |    | Nova Conductor |    | Nova Conductor |
         | Neutron Server |    | Neutron Server |    | Neutron Server |
         | Glance API     |    | Glance API     |    | Glance API     |
         | Cinder API     |    | Cinder API     |    | Cinder API     |
         | Cinder Sched.  |    | Cinder Sched.  |    | Cinder Sched.  |
         | Horizon        |    | Horizon        |    | Horizon        |
         | Heat           |    | Heat           |    | Heat           |
         | Barbican       |    | Barbican       |    | Barbican       |
         | Octavia API    |    | Octavia API    |    | Octavia API    |
         +-----------+----+    +-------+--------+    +----+-----------+
                     |                 |                   |
                     +--------+--------+-------------------+
                              |
                        HAProxy/keepalived
                        VIP: 10.10.1.10
```

- **HAProxy + keepalived** runs on all three control nodes, providing a floating VIP for API endpoints.
- **MariaDB Galera Cluster** for database HA (synchronous multi-master replication).
- **RabbitMQ** in a 3-node quorum queue cluster for message bus HA.
- **Memcached** on each node for Keystone token caching.

### 5.3 OpenStack Services Deployed

| Service | Project | Purpose |
|---------|---------|---------|
| Keystone | Identity | Authentication, authorization, service catalog |
| Nova | Compute | VM lifecycle management |
| Neutron | Networking | SDN (with ACI plugin) |
| Glance | Image | VM image registry (backed by Ceph RBD) |
| Cinder | Block Storage | Persistent volumes (backed by Ceph RBD) |
| Horizon | Dashboard | Web UI |
| Heat | Orchestration | Stack templates (IaC for tenants) |
| Barbican | Key Management | Secret storage, certificate management (integrates with Vault) |
| Octavia | Load Balancing | LBaaS v2 |
| Designate | DNS | DNSaaS (backed by PowerDNS) |
| Ironic | Bare Metal | Optional -- for future bare-metal provisioning |

### 5.4 Compute Node Configuration

Each compute node runs:

- **Nova compute** (libvirt/KVM hypervisor)
- **Opflex agent** (ACI network policy enforcement)
- **Ceph client libraries** (for RBD access to Ceph storage)
- **Node exporter** (Prometheus metrics)
- **Filebeat/Promtail** (log forwarding)
- **AIDE** (file integrity monitoring -- PCI-DSS requirement 11.5)
- **CrowdSec or OSSEC agent** (host-based IDS)

#### Hypervisor Tuning

```
# /etc/nova/nova.conf (key settings)
[DEFAULT]
cpu_allocation_ratio = 4.0       # Conservative for financial workloads
ram_allocation_ratio = 1.5       # Slight overcommit, adjust based on usage
disk_allocation_ratio = 1.0      # No disk overcommit

[libvirt]
virt_type = kvm
cpu_mode = host-passthrough      # Best performance, live migration within same CPU family
images_type = rbd                # Boot from Ceph
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
```

### 5.5 Host Aggregates and Availability Zones

| Availability Zone | Hosts | Purpose |
|-------------------|-------|---------|
| AZ-1 | compute-01 through compute-08 (Racks 3-4) | Primary zone |
| AZ-2 | compute-09 through compute-16 (Racks 5-6) | Secondary zone |

| Host Aggregate | Hosts | Properties |
|----------------|-------|-----------|
| `pci-compliant` | compute-01 to compute-04 | `pci=true` -- hardened, CDE-only workloads |
| `general` | compute-05 to compute-16 | `pci=false` -- general workloads |

PCI workloads are scheduled only on `pci-compliant` hosts via Nova flavor extra-specs:

```
openstack flavor set m1.pci-medium --property aggregate_instance_extra_specs:pci=true
```

This satisfies the PCI-DSS requirement for dedicated, hardened hosts within the CDE.

### 5.6 VM Image Pipeline

```
Git repo (Packer templates)
    |
CI Pipeline (GitLab CI / Jenkins)
    |
Packer + QEMU builds golden image
    |
Automated security scan (OpenSCAP, Trivy)
    |
Upload to Glance (Ceph-backed)
    |
Tag as "approved" in Glance metadata
    |
Nova only boots images with "approved=true" (image property filter)
```

Base images: Ubuntu 22.04 LTS, Rocky Linux 9, Windows Server 2022 (if needed). All images include:
- CIS benchmark Level 1 hardening
- Pre-installed monitoring agents
- SSH key-only auth (no password login)
- Automatic security updates enabled

---

## 6. Storage Architecture (Ceph)

### 6.1 Ceph Cluster Design

```
                    +----------+     +----------+     +----------+
                    |  MON/MGR |     |  MON/MGR |     |  MON/MGR |
                    |  ceph-01 |     |  ceph-02 |     |  ceph-03 |
                    +----+-----+     +----+-----+     +----+-----+
                         |                |                |
        +------+---------+--------+-------+--------+------+-------+
        |      |         |        |       |        |      |       |
     +--+--+ +-+---+ +--+--+ +--+--+ +--+--+ +--+--+ +--+--+ +--+--+
     |OSD-1| |OSD-2| |OSD-3| |OSD-4| |OSD-5| |OSD-6| |OSD-7| |OSD-8|
     |12drv| |12drv| |12drv| |12drv| |12drv| |12drv| |12drv| |12drv|
     +-----+ +-----+ +-----+ +-----+ +-----+ +-----+ +-----+ +-----+
       Rack 7   Rack 7  Rack 7  Rack 7  Rack 8  Rack 8  Rack 8  Rack 8
```

- **Ceph release:** Reef (v18.2.x) or Squid (v19.x) -- latest stable
- **Deployment tool:** `cephadm` (official orchestrator, container-based)
- **MON nodes:** 3 (quorum requires majority -- tolerates 1 failure)
- **MGR nodes:** Co-located with MONs, active/standby
- **OSD nodes:** 8 nodes, 12 OSDs each = **96 OSDs total**
- **MDS nodes:** Co-located with MONs (only needed if CephFS is used)

### 6.2 Network Separation

Ceph uses two distinct networks:

| Network | Subnet | Purpose | Interface |
|---------|--------|---------|-----------|
| Public (client) | 10.40.1.0/24 | Client-to-OSD traffic (compute nodes access this) | bond0 (2x 25GbE) |
| Cluster (replication) | 10.40.2.0/24 | OSD-to-OSD replication, recovery, rebalancing | bond1 (2x 25GbE) |

This separation is critical for performance: replication/recovery traffic does not compete with client I/O.

### 6.3 Pool Design

| Pool | Type | Replication | Use Case |
|------|------|-------------|----------|
| `vms` | RBD | 3x replicated | Nova ephemeral disks, Glance images |
| `volumes` | RBD | 3x replicated | Cinder persistent volumes |
| `volumes-ssd` | RBD | 3x replicated | High-IOPS Cinder tier (NVMe-only OSD class) |
| `backups` | RBD | EC 4+2 | Cinder backup target |
| `rgw-data` | RGW | EC 4+2 | Object storage (S3 API) for backups, logs |
| `cephfs-data` | CephFS | 3x replicated | Shared filesystem (optional) |
| `cephfs-metadata` | CephFS | 3x replicated | CephFS metadata |

#### CRUSH Rules

CRUSH rules ensure replicas are spread across failure domains:

```
# Ensure replicas land in different racks
rule replicated_rack {
    id 1
    type replicated
    step take default
    step chooseleaf firstn 0 type rack
    step emit
}
```

This means a full rack failure does not lose any data.

### 6.4 Ceph Integration with OpenStack

**Glance (images):**
```ini
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = vms
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

**Cinder (block volumes):**
```ini
[DEFAULT]
enabled_backends = ceph-hdd, ceph-ssd

[ceph-hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_user = cinder
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_secret_uuid = <generated-uuid>

[ceph-ssd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes-ssd
rbd_user = cinder
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_secret_uuid = <generated-uuid>
volume_backend_name = ssd
```

**Nova (ephemeral disks):**
Nova boots VMs directly from Ceph RBD using copy-on-write clones of Glance images. This gives near-instant boot times and enables live migration without shared filesystem.

### 6.5 Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| 4K random read IOPS (cluster) | >500,000 | NVMe-only cluster |
| 4K random write IOPS (cluster) | >200,000 | 3x replication write amplification |
| Sequential throughput | >10 GB/s aggregate | |
| Single volume latency | <1 ms (p99) | For SSD tier |
| Recovery time (single OSD failure) | <2 hours | Tune `osd_recovery_max_active` |

### 6.6 Encryption at Rest

Ceph supports per-OSD encryption using LUKS (dm-crypt). This is enabled at OSD creation time via `cephadm`:

```yaml
# ceph.conf or cephadm service spec
service_type: osd
service_id: default_drive_group
encrypted: true
```

Keys are managed by Ceph's built-in key management or integrated with Vault via the `dmcrypt` key storage backend. This satisfies PCI-DSS requirement 3.4 (render PAN unreadable anywhere it is stored).

---

## 7. Identity & Access Management

### 7.1 Architecture Overview

```
                  +------------------+
                  |   External AD    |  (if existing)
                  +--------+---------+
                           |
                    LDAP / Kerberos trust
                           |
                  +--------+---------+
                  |     FreeIPA      |  (2x replicas)
                  |  LDAP + Kerberos |
                  |  DNS + CA        |
                  +--------+---------+
                           |
              +------------+------------+
              |                         |
    +---------+---------+    +----------+----------+
    |     Keycloak      |    |  OpenStack Keystone  |
    | (OIDC/SAML IdP)   |    |  (Identity v3)       |
    | MFA enforcement   |    |  LDAP backend for     |
    | SSO portal        |    |  user auth            |
    +-------------------+    +----------------------+
```

### 7.2 Component Roles

| Component | Role |
|-----------|------|
| **FreeIPA** (2 replicas) | Central LDAP directory, Kerberos KDC, internal CA, host enrollment, sudo rules, HBAC policies |
| **Keycloak** (2 instances, HA) | OIDC/SAML identity provider; provides MFA (TOTP), SSO for Horizon, Grafana, Vault, ArgoCD; federates with external AD if needed |
| **OpenStack Keystone** | OpenStack-native identity; configured with LDAP backend (FreeIPA) for user auth; local service accounts for OpenStack-to-OpenStack communication |
| **HashiCorp Vault** | Secrets engine, dynamic database credentials, PKI/TLS certificate issuance, encryption-as-a-service |

### 7.3 RBAC Model

#### OpenStack Roles

| Keystone Role | Permissions | Assigned To |
|---------------|------------|-------------|
| `cloud-admin` | Full admin across all projects | Ops team leads (2 people) |
| `project-admin` | Admin within a single project | Team leads per business unit |
| `member` | Create/manage own VMs and volumes | Developers, analysts |
| `reader` | Read-only access | Auditors, compliance team |
| `pci-admin` | Admin within PCI project only | PCI-authorized ops (3 people) |

#### Infrastructure Access

| Access Level | Mechanism | MFA Required | Who |
|-------------|-----------|--------------|-----|
| SSH to hypervisors | FreeIPA + SSH certificates (short-lived, Vault-signed) | Yes | Ops team only |
| SSH to VMs | Tenant-managed keys, injected via cloud-init | Per tenant policy | VM owners |
| APIC admin | Local APIC account + TACACS+ to FreeIPA | Yes | 2 designated network ops |
| Ceph admin | `ceph` CLI via FreeIPA-enrolled hosts, sudo rules | Yes | 2 designated storage ops |
| Vault admin | OIDC via Keycloak | Yes | 2 designated ops |
| BMC/IPMI | Isolated OOB network, local accounts, rotated quarterly | N/A (physical) | All ops |

### 7.4 MFA Enforcement

- **Keycloak** enforces TOTP (Google Authenticator / FreeOTP) for all administrative logins.
- **SSH** uses Vault-issued short-lived certificates (TTL: 8 hours). To get a cert, the operator authenticates to Vault via OIDC (which triggers MFA through Keycloak).
- **PCI-DSS requirement 8.3:** MFA is required for all non-console administrative access to CDE systems.

### 7.5 Service Account Management

- OpenStack service accounts (nova, neutron, cinder, etc.) are **local to Keystone** -- not in LDAP.
- Ceph client keyrings are distributed via Ansible with minimum required capabilities:
  ```
  # Example: cinder user can only access volumes pool
  ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=volumes-ssd'
  ```
- All service account credentials are stored in **HashiCorp Vault** and rotated automatically every 90 days.

---

## 8. Monitoring, Logging & Alerting

### 8.1 Monitoring Stack

```
+-------------------+     +------------------+     +-------------------+
| Prometheus (2x)   |<----| Node Exporter    |     | Alertmanager (2x) |
| (HA pair)         |<----| Ceph Exporter    |---->| PagerDuty / email |
|                   |<----| OpenStack Export. |     | Slack / Teams     |
|                   |<----| SNMP Exporter    |     +-------------------+
|                   |<----| Blackbox Exporter |
+--------+----------+     +------------------+
         |
+--------+----------+
|   Grafana (2x)    |
|  (HA, PostgreSQL  |
|   backend)        |
+-------------------+
```

### 8.2 Metrics Collection

| Exporter | Metrics | Scrape Interval |
|----------|---------|----------------|
| `node_exporter` | CPU, RAM, disk, network per host | 15s |
| `ceph_exporter` (built-in MGR module) | Cluster health, OSD status, pool I/O, PG states | 15s |
| `openstack-exporter` | Nova instances, Neutron agents, Cinder volumes, API latency | 30s |
| `snmp_exporter` | Cisco Nexus interface counters, errors, ACI health scores | 30s |
| `blackbox_exporter` | API endpoint availability (HTTP probes), ICMP probes | 10s |
| `haproxy_exporter` | Control plane LB stats | 15s |
| `libvirt_exporter` | Per-VM CPU, memory, disk, network | 30s |

**Retention:** Prometheus local storage retains 30 days of full-resolution data. For long-term storage, Thanos or VictoriaMetrics can be added to ship data to Ceph RGW (S3-compatible).

### 8.3 Logging Stack

```
All Hosts                          Log Aggregation                   Storage
+----------+     +-----------+     +-------------------+     +----------------+
| Promtail |---->|           |     |                   |     |                |
| (or      |     |  Loki     |---->|  Ceph RGW (S3)    |     |  Grafana       |
| Filebeat)|     |  (3 nodes)|     |  Long-term store  |     |  (Log viewer)  |
+----------+     +-----------+     +-------------------+     +----------------+
```

| Log Source | Collection Method | Retention |
|------------|------------------|-----------|
| Syslog (all hosts) | Promtail with syslog target | 90 days hot, 1 year cold (S3) |
| OpenStack service logs | Promtail scraping container logs | 90 days hot, 1 year cold |
| ACI fabric logs | Syslog forwarding from APIC to Loki | 90 days hot, 1 year cold |
| Firewall logs | Syslog from Palo Alto to Loki | 1 year hot (PCI-DSS req 10.7) |
| Authentication logs | Promtail + FreeIPA audit log | 1 year hot (PCI-DSS/SOC 2) |
| Ceph cluster logs | Promtail on MON/OSD nodes | 90 days hot |
| Vault audit logs | File audit device -> Promtail | 1 year hot (immutable) |

**PCI-DSS requirement 10:** All log data is:
- Collected centrally
- Tamper-evident (write-once S3 buckets with object lock)
- Reviewed daily (automated anomaly detection alerts)
- Retained for at least 1 year, with 3 months immediately accessible

### 8.4 Key Dashboards (Grafana)

| Dashboard | Content |
|-----------|---------|
| Cloud Overview | Total VMs, total vCPU/RAM usage, API error rates, cluster health |
| Ceph Health | OSD status, pool utilization, IOPS, throughput, recovery progress |
| Network / ACI | Fabric health score, top talkers, dropped packets, contract hit counts |
| Compute Nodes | Per-node CPU, memory, VM count, libvirt metrics |
| PCI CDE Status | CDE host compliance state, access logs, segmentation test results |
| Capacity Forecast | Trending CPU/RAM/storage utilization with projections |
| Security Events | Failed logins, privilege escalations, FIM alerts, IDS events |

### 8.5 Alerting Rules (Critical Subset)

| Alert | Condition | Severity | Notification |
|-------|-----------|----------|-------------|
| `CephHealthError` | `ceph_health_status == 2` | Critical | PagerDuty + Slack |
| `CephOSDDown` | Any OSD down > 5 min | Warning | Slack |
| `CephNearFull` | Pool usage > 80% | Warning | Slack + Email |
| `ControlPlaneAPIDown` | Blackbox probe fails for any OpenStack API > 2 min | Critical | PagerDuty |
| `ComputeNodeDown` | `nova-compute` agent down > 5 min | Critical | PagerDuty |
| `NeutronAgentDown` | Any Neutron/opflex agent down > 5 min | Critical | PagerDuty |
| `ACIFaultCritical` | ACI health score < 80 or critical fault raised | Critical | PagerDuty |
| `HighAPILatency` | p99 API response > 5s for 5 min | Warning | Slack |
| `DiskPredictiveFailure` | SMART pre-fail attribute triggered | Warning | Slack + Email |
| `SecurityAuthFailure` | > 10 failed auth attempts in 5 min from single IP | Warning | Slack + Security team |
| `FIMChange` | AIDE detects unauthorized file change on CDE host | Critical | PagerDuty + Security |

### 8.6 On-Call Rotation

With a 6-person ops team, implement a weekly rotation:

- **Primary on-call:** 1 person, receives PagerDuty alerts, 15-min SLA for Critical
- **Secondary on-call:** 1 person, escalation after 30 min
- **Rotation:** Weekly, 6-week cycle, each person is primary ~8 weeks/year

---

## 9. Automation & Infrastructure as Code

### 9.1 Toolchain

```
+---------------+     +-----------+     +----------------+     +-----------+
| Git (GitLab)  |---->| CI/CD     |---->| Ansible        |---->| Target    |
| IaC repos     |     | Pipeline  |     | (config mgmt)  |     | Systems   |
+---------------+     +-----------+     +----------------+     +-----------+
                           |
                      +----+----+
                      |Terraform|  (OpenStack provider)
                      +---------+
                           |
                      +----+----+
                      | ArgoCD  |  (GitOps for k8s apps if applicable)
                      +---------+
```

### 9.2 Repository Structure

```
infrastructure/
  +-- ansible/
  |     +-- inventory/
  |     |     +-- production/
  |     |     +-- staging/
  |     +-- playbooks/
  |     |     +-- site.yml              # Full deployment
  |     |     +-- openstack.yml         # Kolla-Ansible wrapper
  |     |     +-- ceph.yml              # Ceph deployment
  |     |     +-- monitoring.yml        # Prometheus/Grafana/Loki
  |     |     +-- security-hardening.yml# CIS benchmarks
  |     |     +-- compliance-scan.yml   # OpenSCAP runs
  |     +-- roles/
  |           +-- base-hardening/
  |           +-- ntp-client/
  |           +-- monitoring-agent/
  |           +-- ceph-client/
  |           +-- vault-agent/
  +-- terraform/
  |     +-- modules/
  |     |     +-- openstack-project/    # Standardized project setup
  |     |     +-- openstack-network/    # Network with ACI integration
  |     |     +-- openstack-vm/         # VM with standard config
  |     |     +-- pci-environment/      # PCI-compliant project template
  |     +-- environments/
  |           +-- pci-prod/
  |           +-- general-prod/
  |           +-- dev/
  +-- packer/
  |     +-- ubuntu-2204-base/
  |     +-- rocky9-base/
  |     +-- pci-hardened-ubuntu/
  +-- policies/
        +-- opa/                        # Open Policy Agent policies
        +-- sentinel/                   # (if using Terraform Enterprise)
```

### 9.3 Deployment Workflow

#### Initial Platform Deployment

1. **Bare metal provisioning:** PXE boot + Foreman/MAAS provisions OS on all servers
2. **Base hardening:** Ansible `base-hardening` role applies CIS Level 1 + custom hardening
3. **Ceph deployment:** `cephadm bootstrap` + Ansible orchestration for OSD deployment
4. **OpenStack deployment:** Kolla-Ansible deploys all OpenStack services
5. **ACI integration:** Ansible configures ACI tenants, VRFs, BDs, contracts; deploys Neutron ACI plugin
6. **Monitoring deployment:** Ansible deploys Prometheus, Grafana, Loki, Alertmanager
7. **Identity setup:** Ansible deploys FreeIPA, Keycloak, configures Keystone LDAP backend
8. **Validation:** Automated test suite (Tempest for OpenStack, custom scripts for end-to-end)

#### Day-2 Operations (GitOps)

```
Developer/Ops pushes change to Git
    |
CI Pipeline runs:
    +-- Lint (ansible-lint, terraform validate, yamllint)
    +-- Dry-run (ansible --check --diff, terraform plan)
    +-- Security scan (checkov, tfsec)
    +-- Policy check (OPA/conftest)
    |
Merge to main branch
    |
CD Pipeline runs:
    +-- Ansible apply (with rolling update strategy)
    +-- Terraform apply (with state locking)
    +-- Post-deploy validation tests
    +-- Notification to Slack
```

### 9.4 Key Automation Playbooks

| Playbook | Purpose | Trigger |
|----------|---------|---------|
| `rolling-os-update.yml` | Patch hypervisors one at a time with live migration | Weekly (scheduled) |
| `openstack-upgrade.yml` | Kolla-Ansible container image upgrade | Per release cycle |
| `ceph-upgrade.yml` | Rolling Ceph daemon upgrade | Per release cycle |
| `add-compute-node.yml` | Onboard new compute node (enroll in FreeIPA, configure networking, join Nova) | On demand |
| `decommission-node.yml` | Drain VMs, remove from schedulers, cleanup | On demand |
| `compliance-scan.yml` | OpenSCAP scan against PCI-DSS/CIS profile, report to Grafana | Daily (automated) |
| `certificate-rotation.yml` | Rotate TLS certificates via Vault PKI | Monthly (automated) |
| `backup-control-plane.yml` | Dump MariaDB, backup Keystone/Glance metadata, etcd snapshot | Daily (automated) |

### 9.5 Secrets Management with Vault

```
+------------------+
|  HashiCorp Vault |  (3-node HA cluster with Raft storage)
|                  |
|  Secrets Engines:|
|  - KV v2         |  Static secrets (API keys, passwords)
|  - PKI           |  Internal TLS certificates (auto-renewal)
|  - Database      |  Dynamic MariaDB/PostgreSQL credentials
|  - SSH           |  Signed SSH certificates (short-lived)
|  - Transit       |  Encryption-as-a-service
+------------------+
```

- All Ansible playbooks retrieve secrets from Vault at runtime (via `hashi_vault` lookup plugin).
- No secrets are stored in Git, ever.
- Vault audit log captures every secret access (who, what, when).
- Vault is unsealed using auto-unseal with a cloud KMS or Shamir shares held by 3 of 6 ops team members.

---

## 10. Security & Compliance (PCI-DSS / SOC 2)

### 10.1 PCI-DSS v4.0 Control Mapping

| PCI-DSS Requirement | Implementation |
|---------------------|---------------|
| **1. Network Security Controls** | ACI microsegmentation with explicit contracts; perimeter firewall HA pair; CDE in isolated VRF/tenant; ACI service graph with inline FW for CDE ingress/egress |
| **2. Secure Configurations** | CIS-hardened base images; Ansible-enforced configuration; no default passwords; unnecessary services disabled |
| **3. Protect Stored Account Data** | Ceph encryption at rest (dm-crypt/LUKS); Barbican for application-level key management; Vault transit engine for field-level encryption |
| **4. Encrypt Transmissions** | TLS 1.2+ everywhere; OpenStack internal APIs use TLS; Ceph messenger v2 with encryption; ACI fabric uses MACsec (if supported on hardware) |
| **5. Malware Protection** | ClamAV on CDE hosts; container image scanning in CI; no direct Internet access from CDE |
| **6. Secure Systems and Software** | Automated patching pipeline; vulnerability scanning (OpenVAS/Nessus); golden image pipeline with security gates |
| **7. Restrict Access** | Keystone RBAC; FreeIPA HBAC policies; ACI EPG-based access; need-to-know enforcement |
| **8. Identify Users and Authenticate** | FreeIPA + Keycloak; MFA for all admin access; unique IDs; 90-day password rotation; Vault SSH certs (8-hour TTL) |
| **9. Physical Access** | Data center badge + biometric; CCTV 90-day retention; visitor logs; hardware tamper detection |
| **10. Log and Monitor** | Centralized logging (Loki); 1-year retention; daily log review (automated alerts); tamper-proof storage (S3 object lock) |
| **11. Regular Security Testing** | Quarterly ASV scans; annual penetration test; automated internal vulnerability scans (weekly); AIDE file integrity monitoring; network segmentation testing (ACI contract verification) |
| **12. Organizational Policies** | Security policies documented; incident response plan; annual security awareness training; risk assessment |

### 10.2 SOC 2 Type II Alignment

SOC 2 trust service criteria overlap significantly with PCI-DSS. Additional considerations:

| Criteria | Implementation |
|----------|---------------|
| **Security (CC6)** | All PCI-DSS controls above; change management via Git (auditable) |
| **Availability (A1)** | N+1 redundancy at every layer; SLA monitoring via Prometheus; DR plan documented |
| **Processing Integrity (PI1)** | Input validation at application layer; automated testing; monitoring for anomalous processing |
| **Confidentiality (C1)** | Encryption at rest and in transit; network segmentation; DLP on perimeter firewall |
| **Privacy** | Data classification tagging in OpenStack metadata; access logging; data retention automation |

### 10.3 CDE Network Segmentation Verification

ACI provides built-in tools for verifying segmentation:

```bash
# Verify no unexpected traffic flows between CDE and non-CDE
# Run from APIC CLI or via API:
apic-contract-audit --tenant pci --verify-isolation

# Automated test (run weekly):
# 1. Deploy test VM in non-CDE network
# 2. Attempt connection to CDE subnets on all ports
# 3. Verify 100% deny
# 4. Log results and alert on any unexpected allow
```

This is scripted and runs as a weekly automated job, with results pushed to the compliance dashboard.

### 10.4 Hardening Standards

All systems are hardened per:

- **CIS Benchmark Level 1** (at minimum) for the OS
- **STIG profiles** for government-adjacent requirements
- **OpenSCAP** automated scanning with PCI-DSS and CIS content

Ansible role `base-hardening` enforces:

```yaml
# Key hardening items (non-exhaustive)
- Disable unused filesystems (cramfs, freevxfs, jffs2, hfs, hfsplus, udf)
- Set /tmp noexec,nosuid,nodev
- Disable core dumps
- Enable ASLR (kernel.randomize_va_space = 2)
- Configure auditd for file access, privilege escalation, time changes
- Set password complexity (minlen=14, complexity classes)
- Configure fail2ban (5 failures = 30 min lockout)
- Disable root SSH login
- Enable SELinux/AppArmor enforcing mode
- Remove unnecessary packages
- Configure NTP (PCI-DSS 10.6)
- Enable TCP SYN cookies
- Disable ICMP redirects
- Configure rsyslog forwarding to central Loki
```

### 10.5 Vulnerability Management

| Activity | Frequency | Tool | Scope |
|----------|-----------|------|-------|
| Internal vulnerability scan | Weekly | OpenVAS / Nessus | All hosts |
| External ASV scan | Quarterly | Approved Scanning Vendor | Public-facing IPs |
| Container image scan | Every build | Trivy | Kolla images, golden VM images |
| Dependency scan | Every build | Dependabot / Snyk | IaC repos, application code |
| Penetration test | Annual | Third-party firm | Full scope including CDE |
| Configuration audit | Daily | OpenSCAP | All hosts |

### 10.6 Incident Response

An incident response plan is maintained with these phases:

1. **Detection** -- Automated alerts (Prometheus/Alertmanager), SIEM correlation, user reports
2. **Triage** -- On-call ops classifies severity (P1-P4), engages security team for P1/P2
3. **Containment** -- ACI contracts can isolate compromised segments within minutes; Ansible playbook to quarantine hosts
4. **Eradication** -- Re-image compromised hosts from golden images; rotate all credentials via Vault
5. **Recovery** -- Restore from Ceph snapshots/backups; gradual reconnection with monitoring
6. **Post-mortem** -- Blameless post-incident review; update runbooks; compliance notification if PCI data affected

---

## 11. Disaster Recovery & Business Continuity

### 11.1 Backup Strategy

| Data | Method | Frequency | Retention | Storage Target |
|------|--------|-----------|-----------|---------------|
| OpenStack DB (MariaDB) | `mysqldump` via Ansible | Every 6 hours | 30 days | Ceph RGW (S3) |
| Keystone/Glance metadata | DB dump + Glance image list export | Daily | 30 days | Ceph RGW |
| Ceph RBD snapshots | Scheduled RBD snapshots | Daily | 7 daily, 4 weekly | Same Ceph cluster (different pool) |
| Ceph RBD offsite | `rbd export` to remote S3 or tape | Weekly | 90 days | Offsite location |
| VM-level backups | Cinder backup service | Daily for critical VMs | 30 days | Ceph RGW (EC pool) |
| Configuration (IaC) | Git repository | Every commit | Indefinite | GitLab (replicated) |
| Vault data | Raft snapshots | Daily | 30 days | Offsite encrypted storage |
| ACI configuration | APIC configuration export | Daily + pre-change | 30 days | Ceph RGW |

### 11.2 Recovery Targets

| Tier | RTO | RPO | Scope |
|------|-----|-----|-------|
| Tier 1 (PCI/Critical) | 4 hours | 1 hour | Payment processing, core banking APIs |
| Tier 2 (Important) | 8 hours | 6 hours | Internal applications, databases |
| Tier 3 (Standard) | 24 hours | 24 hours | Development, testing environments |

### 11.3 Failure Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| Single compute node failure | VMs on that node restart on other nodes (Nova evacuate) | Automatic (5-10 min) |
| Single OSD failure | Ceph rebalances data automatically; no data loss | Automatic (1-2 hours) |
| Full OSD node failure (12 OSDs) | Ceph rebalances; increased latency during recovery | Automatic (4-8 hours) |
| Single control plane node | HAProxy routes to remaining 2 nodes; no service impact | Automatic (seconds) |
| Full rack loss | VMs in that AZ restart in other AZ; Ceph survives (CRUSH rack rule) | Semi-automatic (15-30 min) |
| ACI spine failure | Remaining spine handles all traffic (reduced bandwidth) | Automatic (sub-second failover) |
| Both spines fail | Total network outage; all VMs unreachable | Manual (hardware replacement) |
| Full data center loss | Total outage; restore from offsite backups | Manual (RTO depends on DR site availability) |

### 11.4 DR Site Considerations

For a full DR site (future phase), consider:

- A second, smaller cluster (50% capacity) in a geographically separate data center
- Ceph RBD mirroring (async) for critical volumes
- OpenStack database replication to standby control plane
- ACI Multi-Site for network policy synchronization
- Estimated additional cost: 40-60% of primary site

---

## 12. Operational Procedures

### 12.1 Day-1 Deployment Checklist

- [ ] Rack and cable all hardware per rack diagram
- [ ] Verify power redundancy (both feeds active)
- [ ] Configure BMC/IPMI on OOB network
- [ ] PXE boot and install base OS on all nodes
- [ ] Run Ansible base-hardening playbook
- [ ] Bootstrap Ceph cluster (cephadm)
- [ ] Deploy OSD nodes and create pools
- [ ] Deploy OpenStack via Kolla-Ansible
- [ ] Configure ACI fabric (tenants, VRFs, BDs, contracts)
- [ ] Deploy Neutron ACI plugin and verify integration
- [ ] Deploy FreeIPA and enroll all hosts
- [ ] Deploy Keycloak and configure OIDC for Horizon
- [ ] Deploy Vault and configure secrets engines
- [ ] Deploy monitoring stack (Prometheus, Grafana, Loki)
- [ ] Run OpenStack Tempest tests
- [ ] Run compliance scan (OpenSCAP)
- [ ] Run segmentation verification tests
- [ ] Document as-built configuration
- [ ] Conduct ops team training on runbooks

### 12.2 Routine Maintenance Schedule

| Task | Frequency | Method | Downtime |
|------|-----------|--------|----------|
| OS security patches | Weekly | Ansible rolling update with live migration | Zero (rolling) |
| OpenStack minor upgrade | Quarterly | Kolla-Ansible container image update | Zero (rolling) |
| OpenStack major upgrade | Annually | Kolla-Ansible with staged rollout | Brief API downtime (<15 min) |
| Ceph minor upgrade | Quarterly | cephadm rolling upgrade | Zero |
| ACI firmware upgrade | Semi-annually | APIC-orchestrated rolling upgrade | Zero (if dual-spine) |
| TLS certificate rotation | Monthly | Vault PKI auto-renewal | Zero |
| Credential rotation | Quarterly | Vault dynamic secrets + Ansible | Zero |
| Compliance scan | Daily | OpenSCAP via Ansible | Zero |
| Backup verification (restore test) | Monthly | Restore random VM from backup | N/A (test environment) |
| DR failover test | Semi-annually | Simulate component failures | Planned maintenance window |

### 12.3 Runbook Index

Each runbook should be a versioned document in Git:

1. `runbook-compute-node-failure.md` -- Evacuate VMs, diagnose hardware, RMA, re-provision
2. `runbook-ceph-osd-failure.md` -- Identify failed OSD, replace disk, verify rebalance
3. `runbook-control-plane-recovery.md` -- Recover from 1 or 2 control node failures, DB recovery
4. `runbook-aci-fault-triage.md` -- Interpret ACI faults, common contract misconfigs
5. `runbook-security-incident.md` -- Incident response steps, isolation procedures, evidence preservation
6. `runbook-capacity-expansion.md` -- Add compute/storage nodes
7. `runbook-openstack-upgrade.md` -- Step-by-step upgrade procedure with rollback plan
8. `runbook-tenant-onboarding.md` -- Create project, quotas, networks, initial access

### 12.4 Team Skill Requirements

With a 6-person team, recommended specialization (with cross-training):

| Role | Primary | Backup |
|------|---------|--------|
| OpenStack / Compute | Person A | Person B |
| Ceph / Storage | Person C | Person D |
| Networking / ACI | Person E | Person A |
| Security / Compliance | Person B | Person F |
| Automation / CI/CD | Person D | Person C |
| Monitoring / On-call Lead | Person F | Person E |

Every team member should be able to perform basic triage across all domains.

---

## 13. Capacity Planning & Growth

### 13.1 Current Sizing vs. Target

| Resource | Provisioned | Day-1 Target (500 VMs) | Utilization | Headroom |
|----------|-------------|----------------------|-------------|----------|
| vCPU | 16 nodes x 64 cores x 4:1 = 4,096 vCPUs | ~2,000 vCPUs | 49% | 51% |
| RAM | 16 TB (1.5:1 overcommit = 24 TB effective) | ~16 TB | 67% | 33% |
| Storage (usable) | ~245 TB (3x replication) | ~100 TB | 41% | 59% |
| Network (per leaf) | 48x 25GbE ports | ~20 ports per leaf | 42% | 58% |

### 13.2 Growth Triggers

| Resource | Trigger Threshold | Action |
|----------|------------------|--------|
| Compute (RAM) | 75% sustained for 30 days | Add 2 compute nodes (1 rack has capacity for 4 more) |
| Storage | 70% pool utilization | Add 2 OSD nodes or expand with additional drives |
| Network | 80% uplink utilization on any leaf pair | Add uplinks or upgrade to 100GbE server connections |
| Control plane | API p99 latency > 3s consistently | Tune, then consider dedicated API/DB nodes |

### 13.3 Scaling Path

**Phase 1 (current):** 16 compute, 8 storage -- supports 500+ VMs
**Phase 2 (12-18 months):** Add 4 compute nodes, 2 storage nodes -- supports 700+ VMs
**Phase 3 (24-36 months):** Second rack pair for compute, consider second site for DR

Each compute node addition is automated via Ansible playbook and takes approximately 2 hours from rack-to-production.

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Ceph data loss (triple failure) | Very Low | Critical | CRUSH rack-level failure domain; monitoring; offsite backups |
| OpenStack control plane outage | Low | High | 3-node HA; tested failover; DB backups every 6 hours |
| ACI fabric partition | Very Low | Critical | Dual spine; redundant leaf uplinks; out-of-band management |
| Ransomware/compromise | Medium | Critical | Network segmentation; immutable backups (S3 object lock); incident response plan |
| Key person dependency | Medium | High | Cross-training; documented runbooks; automation reduces tribal knowledge |
| Vendor lock-in (ACI) | Medium | Medium | ACI is the most coupled component; OpenStack Neutron abstracts some; document migration path to OVN |
| Compliance audit failure | Low | High | Automated daily compliance scans; pre-audit preparation checklist |
| Hardware supply chain delay | Medium | Medium | Maintain 1 spare compute node and spare disks on-site |
| Power/cooling failure | Low | Critical | Dual power feeds; UPS; generator; monitoring with 15-min alert |
| Ops team burnout | Medium | High | Automation reduces toil; on-call rotation; invest in self-healing |

---

## 15. Bill of Materials Summary

This is an estimated BOM for budgeting. Exact pricing depends on vendor negotiations.

| Category | Item | Qty | Estimated Unit Cost | Total |
|----------|------|-----|-------------------|-------|
| **Network** | Cisco Nexus 9500 Spine | 2 | (already purchased) | -- |
| | Cisco Nexus 9300 Leaf | 14 | (already purchased) | -- |
| | Cisco APIC | 3 | (already purchased) | -- |
| | Palo Alto PA-3400 (HA pair) | 2 | $35,000 | $70,000 |
| | 100GbE optics/cables (spine-leaf) | 56 | $300 | $16,800 |
| | 25GbE optics/cables (server) | 200 | $50 | $10,000 |
| **Compute** | Control plane nodes | 3 | $25,000 | $75,000 |
| | Compute nodes (1TB RAM) | 16 | $35,000 | $560,000 |
| **Storage** | Ceph OSD nodes (12x 7.68TB NVMe) | 8 | $65,000 | $520,000 |
| | Ceph MON/MGR nodes | 3 | $15,000 | $45,000 |
| **Infrastructure** | OOB management switches | 2 | $3,000 | $6,000 |
| | Console servers | 2 | $4,000 | $8,000 |
| | UPS (per rack) | 8 | $5,000 | $40,000 |
| | PDUs (dual per rack) | 16 | $1,500 | $24,000 |
| | Racks (42U) | 8 | $2,000 | $16,000 |
| | Cabling and patch panels | -- | -- | $15,000 |
| **Software** | OpenStack (open source) | -- | $0 | $0 |
| | Ceph (open source) | -- | $0 | $0 |
| | Cisco ACI licensing (Essentials) | -- | $50,000/yr | $50,000/yr |
| | HashiCorp Vault (Enterprise, optional) | -- | $30,000/yr | $30,000/yr |
| | Monitoring stack (open source) | -- | $0 | $0 |
| **Services** | Professional services (deployment) | -- | -- | $100,000-200,000 |
| | PCI-DSS QSA assessment | -- | -- | $30,000-50,000/yr |
| | SOC 2 Type II audit | -- | -- | $30,000-50,000/yr |
| | Annual penetration test | -- | -- | $20,000-40,000/yr |
| | **Hardware subtotal** | | | **~$1,406,800** |
| | **Annual recurring** | | | **~$180,000-290,000/yr** |

**Note:** This excludes data center colocation costs, power/cooling, and staff salaries. Total first-year cost (hardware + software + services) is estimated at **$1.6M - $1.9M**. Compare against current AWS spend to validate ROI. Typical break-even for builds of this size is 18-30 months.

---

## Appendix A: Architecture Diagram (Logical)

```
+===========================================================================+
|                          EXTERNAL NETWORK                                  |
|   [Internet] --- [BGP Router x2] --- [Palo Alto FW HA] --- [ACI L3Out]   |
+===========================================================================+
         |
+===========================================================================+
|                         CISCO ACI FABRIC                                   |
|   +-------------------------------------------------------------------+   |
|   |  Spine 1 (N9500)  <=========100G=========>  Spine 2 (N9500)      |   |
|   +-------------------------------------------------------------------+   |
|          |          |          |          |          |          |          |
|   +------+   +------+   +------+   +------+   +------+   +------+        |
|   |Leaf 1|   |Leaf 3|   |Leaf 5|   |Leaf 7|   |Leaf 9|   |Leaf11|        |
|   |Leaf 2|   |Leaf 4|   |Leaf 6|   |Leaf 8|   |Leaf10|   |Leaf12|        |
|   +------+   +------+   +------+   +------+   +------+   +------+        |
+===========================================================================+
      |              |                   |              |
+============+ +==========================+ +======================+
| MGMT PLANE | |     COMPUTE PLANE        | |    STORAGE PLANE     |
|            | |                          | |                      |
| ctrl-01    | | compute-01 ... -16       | | osd-01 ... osd-08    |
| ctrl-02    | |  +-- Nova (KVM)          | |  +-- Ceph OSD x12   |
| ctrl-03    | |  +-- Opflex agent        | |  +-- BlueStore       |
|            | |  +-- Ceph RBD client     | |  +-- dm-crypt        |
| OpenStack  | |  +-- Node exporter       | |                      |
| APIs       | |  +-- Promtail            | | mon-01, mon-02,      |
| MariaDB    | |  +-- AIDE/FIM            | | mon-03               |
| RabbitMQ   | |                          | |  +-- Ceph MON/MGR    |
| Keystone   | | AZ-1: compute-01 to -08  | |  +-- Ceph MDS        |
| Horizon    | | AZ-2: compute-09 to -16  | |  +-- RGW (S3 API)   |
+============+ +==========================+ +======================+
      |              |                   |
+===========================================================================+
|                        SHARED SERVICES                                     |
|   [FreeIPA x2] [Keycloak x2] [Vault x3] [Prometheus x2] [Grafana x2]    |
|   [Loki x3] [GitLab] [PowerDNS x2] [NTP x3] [Alertmanager x2]           |
+===========================================================================+
```

---

## Appendix B: IP Address Allocation Summary

| Network | CIDR | VLAN | Purpose |
|---------|------|------|---------|
| OOB Management | 10.10.0.0/24 | 100 | BMC/IPMI |
| Control Plane API | 10.10.1.0/24 | 110 | OpenStack public API |
| Control Internal | 10.10.2.0/24 | 111 | Galera, RabbitMQ, internal API |
| Monitoring | 10.10.3.0/24 | 112 | Prometheus, Grafana, Loki |
| Shared Services | 10.10.4.0/24 | 113 | FreeIPA, Keycloak, Vault, DNS, NTP |
| PCI Application | 10.20.1.0/24 | 201 | CDE application tier |
| PCI Database | 10.20.2.0/24 | 202 | CDE database tier |
| PCI Web | 10.20.3.0/24 | 203 | CDE web/API tier |
| Prod Application | 10.30.1.0/23 | 301 | General production apps |
| Prod Database | 10.30.4.0/24 | 302 | Production databases |
| Prod Web | 10.30.6.0/24 | 303 | Production web tier |
| Dev/Staging | 10.50.0.0/16 | 500-599 | Development workloads |
| Ceph Public | 10.40.1.0/24 | 401 | Ceph client traffic |
| Ceph Cluster | 10.40.2.0/24 | 402 | Ceph replication traffic |
| Tenant overlay | (VXLAN) | 200-999 | ACI-managed dynamic tenant networks |

---

## Appendix C: Compliance Audit Preparation Checklist

Before any PCI-DSS QSA or SOC 2 auditor visit:

- [ ] Run full OpenSCAP compliance scan and remediate findings
- [ ] Verify network segmentation tests passed (weekly automated + manual verification)
- [ ] Confirm all systems patched within SLA (30 days critical, 90 days others)
- [ ] Verify log retention (1 year available, 3 months immediately accessible)
- [ ] Confirm MFA enabled for all admin accounts (pull report from Keycloak)
- [ ] Verify encryption at rest (Ceph dm-crypt) and in transit (TLS) active
- [ ] Review and update network diagrams (this document)
- [ ] Review and update data flow diagrams
- [ ] Confirm backup restore test completed within last quarter
- [ ] Verify vulnerability scan results and remediation status
- [ ] Confirm incident response plan reviewed and updated
- [ ] Gather evidence of change management (Git history, CI/CD pipeline logs)
- [ ] Confirm user access reviews completed (quarterly)
- [ ] Verify clock synchronization across all systems (NTP audit)

---

*This document should be reviewed and updated quarterly, or after any significant infrastructure change. Store in version control alongside the IaC repository.*
