# Private Cloud Architecture: Financial Services Platform

## Document Control

| Field | Value |
|-------|-------|
| Organization | Mid-size Financial Services Company |
| Employees | ~500 |
| Target VM Count | ~500 |
| Ops Team Size | 6 engineers |
| Compliance | PCI-DSS, SOC2 |
| Date | 2026-03-20 |
| Status | Architecture Design - Iteration 1 |

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform to replace the organization's AWS footprint. The platform is built on OpenStack for IaaS, Cisco ACI for network fabric, and Ceph for unified software-defined storage. The design prioritizes PCI-DSS and SOC2 compliance, 99.99% availability, operational simplicity for a 6-person team, and a target TCO reduction of 35-45% versus the current AWS spend over a 5-year horizon.

The architecture follows defense-in-depth security principles, automates infrastructure lifecycle with Ansible and OpenTofu, and provides self-service VM provisioning through OpenStack Horizon and API. Every design decision considers the constraint that 6 engineers must operate the entire stack, so operational simplicity and automation coverage are paramount.

---

## 2. Design Principles

1. **Compliance-first**: PCI-DSS and SOC2 requirements drive every architectural decision. The Cardholder Data Environment (CDE) is isolated at the network, compute, and storage layers.
2. **Automate everything**: A 6-person team cannot operate 500 VMs manually. Every provisioning, configuration, and day-2 operation must be automated and idempotent.
3. **Defense in depth**: Security is layered -- physical, network (ACI microsegmentation), host (hardened OS), application (WAF, runtime security), and data (encryption at rest and in transit).
4. **Failure domains**: The architecture tolerates the loss of any single rack, any single server, any single storage OSD, and any single network switch without service interruption.
5. **FLOSS where equivalent**: Prefer open-source components (Ceph, OpenStack, Prometheus, Keycloak) to minimize licensing costs and avoid vendor lock-in, while keeping Cisco ACI where its policy model provides unique value.
6. **Operational simplicity over elegance**: Choose boring, well-understood technology. The team must be able to troubleshoot at 3 AM.

---

## 3. Physical Infrastructure

### 3.1 Data Center Layout

Deploy across two availability zones (AZs) within a single data center facility, with each AZ in a separate fire zone / power domain. If a second facility is available, use it for DR (see Section 12).

| Zone | Purpose | Racks |
|------|---------|-------|
| AZ-A | Production workloads (primary) | 4 racks |
| AZ-B | Production workloads (secondary) | 4 racks |
| Management | OpenStack control plane, APIC, monitoring | 1 rack |
| Network | Spine switches, border routers, firewalls | 1 rack (top of each AZ) |

Total: ~10 racks.

### 3.2 Power and Cooling

- Dual power feeds (A+B) to every rack from independent UPS and PDU paths.
- Target PUE: 1.4 or better.
- N+1 cooling for each fire zone.
- 8-10 kW per rack average, 15 kW max for high-density compute racks.
- Generator backup with 48-hour fuel capacity.

### 3.3 Rack Layout (per compute rack)

| U Position | Equipment |
|-----------|-----------|
| U42-U41 | Cisco Nexus 9300 leaf switches (2x, ToR) |
| U40 | Patch panel / cable management |
| U1-U39 | Compute and storage nodes (see below) |

---

## 4. Network Architecture

### 4.1 Cisco ACI Fabric Design

The network is a Cisco ACI fabric in spine-leaf topology using the Nexus 9000 switches already procured.

#### Spine Layer
- **2x Cisco Nexus 9500 or 9336C-FX2** spine switches (one per AZ for redundancy).
- 400G or 100G uplinks between spines and leaves.
- Each spine connects to every leaf switch (full mesh).

#### Leaf Layer
- **8x Cisco Nexus 9300-EX/FX/GX** leaf switches, 2 per rack (ToR pair).
- 25G or 10G server-facing ports.
- 100G uplinks to each spine.
- vPC (virtual Port Channel) pairs at every rack for server dual-homing.

#### APIC Cluster
- **3x Cisco APIC controllers** (one per AZ + one in management rack).
- APIC manages the entire fabric policy model declaratively.
- All APIC nodes in a cluster for consensus (odd number required).

#### Border / External Connectivity
- **2x Cisco Nexus 9300 border leaf switches** connecting to:
  - Corporate WAN / MPLS.
  - Internet edge (through firewalls).
  - DR site interconnect.
- BGP peering with upstream provider / WAN edge.
- L3Out configured in ACI for external routing.

### 4.2 ACI Logical Architecture

```
ACI Tenant Model
================

Tenant: finserv-prod
  |
  +-- VRF: cde-vrf              (PCI Cardholder Data Environment - isolated)
  |     |
  |     +-- BD: cde-app-bd       (Bridge Domain for CDE application tier)
  |     |     +-- Subnet: 10.100.1.0/24
  |     |     +-- EPG: epg-cde-app
  |     |
  |     +-- BD: cde-db-bd        (Bridge Domain for CDE database tier)
  |     |     +-- Subnet: 10.100.2.0/24
  |     |     +-- EPG: epg-cde-db
  |     |
  |     +-- BD: cde-web-bd       (Bridge Domain for CDE web tier)
  |           +-- Subnet: 10.100.3.0/24
  |           +-- EPG: epg-cde-web
  |
  +-- VRF: corporate-vrf         (Non-CDE corporate workloads)
  |     |
  |     +-- BD: corp-app-bd
  |     |     +-- Subnet: 10.200.1.0/24
  |     |     +-- EPG: epg-corp-app
  |     |
  |     +-- BD: corp-db-bd
  |     |     +-- Subnet: 10.200.2.0/24
  |     |     +-- EPG: epg-corp-db
  |     |
  |     +-- BD: corp-web-bd
  |           +-- Subnet: 10.200.3.0/24
  |           +-- EPG: epg-corp-web
  |
  +-- VRF: dmz-vrf               (Internet-facing services)
        |
        +-- BD: dmz-bd
              +-- Subnet: 10.250.1.0/24
              +-- EPG: epg-dmz

Tenant: finserv-mgmt
  |
  +-- VRF: mgmt-vrf
        |
        +-- BD: infra-mgmt-bd    (OpenStack control plane, monitoring)
        |     +-- Subnet: 10.10.1.0/24
        |     +-- EPG: epg-infra-mgmt
        |
        +-- BD: oob-mgmt-bd      (Out-of-band management / IPMI / CIMC)
              +-- Subnet: 10.10.2.0/24
              +-- EPG: epg-oob-mgmt
```

### 4.3 ACI Contracts and Microsegmentation

Contracts enforce a whitelist security model (deny-all default). This is critical for PCI-DSS Requirement 1 (network segmentation).

| Contract | Provider EPG | Consumer EPG | Filters (Protocols/Ports) |
|----------|-------------|--------------|---------------------------|
| cde-web-to-app | epg-cde-app | epg-cde-web | TCP/443, TCP/8443 |
| cde-app-to-db | epg-cde-db | epg-cde-app | TCP/5432 (PostgreSQL), TCP/3306 (MySQL) |
| corp-web-to-app | epg-corp-app | epg-corp-web | TCP/443, TCP/8080 |
| corp-app-to-db | epg-corp-db | epg-corp-app | TCP/5432, TCP/3306 |
| mgmt-to-all | All EPGs | epg-infra-mgmt | TCP/22 (SSH), ICMP, TCP/9090-9100 (Prometheus) |
| cde-to-external | L3Out | epg-cde-web | TCP/443 (inbound, via LB) |
| monitoring | All EPGs | epg-infra-mgmt | TCP/9090-9100, UDP/514 (syslog) |

**Key PCI-DSS network controls:**
- The CDE VRF has **no direct route** to the corporate VRF. Any traffic between them must traverse a firewall with explicit rules.
- Inter-VRF routing is handled through an ACI shared L3Out with a firewall service graph (Cisco Firepower or OPNsense in HA).
- Vzany (catch-all) contracts are **not used** -- every flow is explicitly permitted.
- ACI contract logging is enabled for all CDE contracts, forwarding to the SIEM.

### 4.4 ACI-OpenStack Integration

- Deploy the **Cisco ACI Neutron plugin** (ML2 mechanism driver + GBP) to integrate OpenStack Neutron with ACI.
- OpenStack networks map to ACI EPGs automatically.
- OpenStack security groups are enforced through ACI contracts.
- VLAN or VXLAN encapsulation on the host-facing side; VXLAN within the ACI fabric.
- OpFlex agent on each compute node for policy download from APIC.

### 4.5 IP Address Management (IPAM)

- **NetBox** as the source of truth for all IP allocations, VLAN assignments, rack layouts, and device inventory.
- NetBox integrates with OpenStack and Ansible as a dynamic inventory source.
- DNS: **PowerDNS** with PostgreSQL backend, integrated with OpenStack Designate for automatic DNS record creation.
- DHCP: Managed by Neutron DHCP agent (dnsmasq) for tenant networks; ISC Kea for management networks.

### 4.6 Load Balancing

- **OpenStack Octavia** with HAProxy amphora for tenant-facing load balancers.
- Dedicated HA pair of **HAProxy** nodes (bare-metal or VM) for external ingress to the CDE and DMZ.
- Health checks, TLS termination, and connection logging on all load balancers.

### 4.7 Firewall Architecture

- **Perimeter firewall**: HA pair at the border leaf (Cisco Firepower or OPNsense in HA with CARP). Stateful inspection, IDS/IPS (Suricata rules).
- **Inter-VRF firewall**: ACI service graph inserts a firewall between CDE and corporate VRFs.
- **Host firewall**: nftables on every compute node and control plane node, managed by Ansible.
- PCI-DSS requires explicit segmentation between CDE and all other networks. This is enforced at three layers: ACI contracts, firewall rules, and host-level nftables.

---

## 5. Compute Architecture

### 5.1 Hardware Specification

Standardize on a single server model to simplify spares inventory and firmware management.

#### Compute Nodes (for VM hosting)

| Spec | Value |
|------|-------|
| Quantity | 16 nodes (8 per AZ) |
| CPU | 2x AMD EPYC 9454 (48C/96T each), or Intel Xeon 6 equivalent |
| RAM | 512 GB DDR5 ECC per node (8,192 GB total) |
| Local Storage | 2x 960GB NVMe SSD (OS mirror, RAID1) |
| Network | 2x 25GbE (fabric) + 2x 1GbE (IPMI/OOB management) |
| Form Factor | 1U or 2U rackmount |

**Capacity math for 500 VMs:**
- Total vCPUs: 16 nodes x 96 cores = 1,536 physical cores (3,072 hyperthreads).
- At 4:1 vCPU overcommit: 6,144 vCPUs available. At an average of 4 vCPUs per VM = 1,536 VMs max capacity.
- Total RAM: 8,192 GB. At 1.2:1 memory overcommit (conservative for finserv): ~9,830 GB available. At an average of 8 GB per VM = 1,228 VMs max capacity.
- **Headroom**: The platform supports ~500 VMs at roughly 33-40% utilization, leaving comfortable room for failover (lose an entire AZ and still run all VMs in the surviving AZ at ~70% utilization).

#### Storage Nodes (Ceph OSD nodes)

| Spec | Value |
|------|-------|
| Quantity | 6 nodes (3 per AZ) |
| CPU | 1x AMD EPYC 9254 (24C/48T) |
| RAM | 128 GB DDR5 ECC |
| OSD Storage | 8x 3.84TB NVMe SSD (for all-flash) or mixed NVMe+SSD |
| WAL/DB | 2x 800GB NVMe (dedicated, for BlueStore WAL/DB) |
| OS | 2x 480GB SSD (RAID1) |
| Network | 2x 25GbE (Ceph public) + 2x 25GbE (Ceph cluster/replication) + 2x 1GbE (OOB) |

**Storage capacity math:**
- Raw: 6 nodes x 8 drives x 3.84 TB = 184.32 TB raw.
- Usable at 3x replication: ~61 TB usable.
- Usable at erasure coding (4+2): ~122 TB usable.
- Strategy: Use 3x replication for RBD (VM block storage) -- the write performance and recovery speed matter for financial workloads. Use EC for cold object storage.

#### Control Plane Nodes

| Spec | Value |
|------|-------|
| Quantity | 3 nodes (spread across AZs + management rack) |
| CPU | 1x AMD EPYC 9254 (24C/48T) |
| RAM | 256 GB DDR5 ECC |
| Storage | 2x 960GB NVMe SSD (RAID1) |
| Network | 2x 25GbE (fabric) + 2x 1GbE (OOB) |

These nodes run all OpenStack control services (containerized via Kolla), Ceph MON/MGR, and core infrastructure services.

### 5.2 Firmware and Bare-Metal Management

- **Out-of-band management**: IPMI / Redfish (or Cisco CIMC if using UCS hardware) on a dedicated OOB management network (10.10.2.0/24).
- **Bare-metal provisioning**: MAAS (Metal as a Service) for initial OS deployment, or OpenStack Ironic for bare-metal lifecycle within OpenStack.
- **Firmware updates**: Automated via Ansible using vendor-provided tools (e.g., `fwupd`, vendor BMC APIs). Scheduled quarterly during maintenance windows.
- **OS base image**: Ubuntu 24.04 LTS (or RHEL 9 if support contracts are preferred). CIS Level 1 hardened, built by Packer, deployed via MAAS.
- **BIOS settings**: Standardized via Ansible/Redfish -- performance profile, virtualization extensions enabled, NUMA interleaving disabled (NUMA-aware scheduling in Nova instead).

### 5.3 Compute Overcommit Policy

| Resource | Overcommit Ratio | Rationale |
|----------|-----------------|-----------|
| CPU | 4:1 | Financial services workloads are often I/O-bound, not CPU-bound. Monitor and adjust. |
| Memory | 1.2:1 | Conservative. Memory overcommit is risky for financial workloads -- OOM kills are unacceptable. |
| Disk I/O | N/A | Ceph provides shared storage; local disk is only for ephemeral/swap. |

### 5.4 NUMA and Performance Tuning

- Nova is configured with NUMA-aware scheduling (`hw:numa_nodes` flavor extra spec).
- Hugepages (1GB) enabled for database VMs and latency-sensitive workloads.
- CPU pinning available for dedicated-core flavors (e.g., `m1.dedicated.4xlarge`).
- `tuned` profile set to `throughput-performance` on all compute nodes.

---

## 6. Storage Architecture (Ceph)

### 6.1 Ceph Cluster Design

```
Ceph Cluster Topology
=====================

              +------------------+
              |   Ceph Monitors  |  (3x, on control plane nodes)
              |   Ceph Managers  |  (3x, colocated with MONs)
              +--------+---------+
                       |
           +-----------+-----------+
           |                       |
     +-----+------+        +------+-----+
     |   AZ-A     |        |    AZ-B    |
     | OSD Nodes  |        |  OSD Nodes |
     | (3 nodes)  |        |  (3 nodes) |
     | 24 OSDs    |        |  24 OSDs   |
     +------------+        +------------+

  Ceph Public Network:   10.20.1.0/24 (client traffic)
  Ceph Cluster Network:  10.20.2.0/24 (replication traffic)
```

### 6.2 Ceph Configuration

| Parameter | Value |
|-----------|-------|
| Ceph Release | Squid (latest LTS) or Reef |
| OSD Backend | BlueStore |
| Replication | 3x for RBD pools (block storage) |
| Erasure Coding | 4+2 for RGW pools (object storage, if needed) |
| CRUSH Rules | Rack-aware failure domain (min_size=2 ensures writes succeed even with one AZ degraded) |
| Monitors | 3 (one per control plane node) |
| Managers | 3 (active-standby, colocated with MON) |
| MDS | 2 (active-standby, only if CephFS is used for Manila) |
| Dashboard | Ceph MGR dashboard enabled for ops visibility |

### 6.3 Ceph Pools

| Pool | Type | Replication | Purpose |
|------|------|-------------|---------|
| rbd-vms | RBD | 3x | Cinder volumes (VM block devices) |
| rbd-ephemeral | RBD | 3x | Nova ephemeral disks |
| rbd-images | RBD | 3x | Glance images |
| rbd-backups | RBD | 3x | Cinder backups |
| rgw-default | EC 4+2 | N/A | Object storage (Swift API) |
| cephfs-data | CephFS | 3x | Manila shared file systems (if needed) |

### 6.4 Ceph-OpenStack Integration

- **Cinder** uses the RBD backend for block volumes. Direct librbd access from compute nodes (no iSCSI gateway needed).
- **Glance** stores images in Ceph RBD (copy-on-write cloning for fast VM boot).
- **Nova** uses Ceph for ephemeral disks (boot-from-volume preferred to maximize live migration reliability).
- **Manila** (optional) uses CephFS for shared file systems.
- All compute nodes have `ceph.conf` and Cephx keyrings deployed via Ansible.

### 6.5 Storage Performance Targets

| Metric | Target |
|--------|--------|
| 4K random read IOPS (per OSD) | >50,000 (NVMe) |
| 4K random write IOPS (per OSD) | >20,000 (NVMe) |
| Sequential throughput (per OSD) | >1 GB/s |
| Aggregate cluster IOPS | >500,000 (read), >200,000 (write) |
| Tail latency (P99) | <2ms for 4K random read |

### 6.6 Storage Encryption (PCI-DSS Requirement 3)

- **Encryption at rest**: All Ceph OSDs use dm-crypt with LUKS2 encryption. Keys managed by Ceph's built-in dm-crypt support (keys stored encrypted in MON database).
- For higher assurance: integrate with HashiCorp Vault or KMIP-compliant key manager for OSD encryption keys.
- **Encryption in transit**: Ceph messenger v2 with TLS encryption enabled for both public and cluster networks (`ms_cluster_mode = secure`, `ms_service_mode = secure`, `ms_client_mode = secure`).

### 6.7 Data Protection and Backup

- **VM backup**: Ceph RBD snapshots + incremental backup to a secondary Ceph cluster (or to an off-site target via rbd-mirror).
- **Backup tool**: Bareos (FLOSS, enterprise-grade) or Restic for file-level backups. Consider Veeam if the team prefers commercial tooling.
- **Retention**: Daily snapshots retained for 30 days, weekly for 90 days, monthly for 1 year (align with PCI-DSS and SOC2 retention requirements).
- **RBD mirroring**: Asynchronous mirroring from AZ-A to DR site for critical CDE volumes (RPO < 15 minutes).

---

## 7. OpenStack Platform

### 7.1 Deployment Method

**Kolla-Ansible** is the deployment tool. It deploys all OpenStack services as Docker containers, simplifying upgrades and rollback.

- All OpenStack services run in containers on the 3 control plane nodes behind HAProxy/keepalived for HA.
- MariaDB Galera cluster (3 nodes) for the database.
- RabbitMQ cluster (3 nodes, quorum queues) for messaging.
- Memcached (3 nodes) for token caching.

### 7.2 OpenStack Services

| Service | Component | Purpose |
|---------|-----------|---------|
| **Keystone** | Identity | Authentication, authorization, federation |
| **Nova** | Compute | VM lifecycle management |
| **Neutron** | Networking | SDN, integrated with Cisco ACI plugin |
| **Cinder** | Block Storage | Persistent volumes (Ceph RBD backend) |
| **Glance** | Image | VM image management (Ceph RBD backend) |
| **Horizon** | Dashboard | Web UI for self-service |
| **Octavia** | Load Balancer | LBaaS v2 with HAProxy amphora |
| **Designate** | DNS | DNSaaS, integrated with PowerDNS |
| **Heat** | Orchestration | Stack-based VM orchestration |
| **Barbican** | Key Management | Secret storage, TLS cert management |
| **Manila** | Shared Filesystem | CephFS-backed NFS shares (if needed) |
| **Ironic** | Bare Metal | Bare-metal provisioning (future use) |
| **Placement** | Resource Tracking | Resource inventory and allocation |

### 7.3 OpenStack HA Architecture

```
OpenStack Control Plane HA
==========================

              +-----------+
              |  Virtual  |
              |    IP     |  (keepalived VRRP)
              +-----+-----+
                    |
          +---------+---------+
          |    HAProxy (x3)   |  (active/passive with keepalived)
          +---------+---------+
                    |
     +--------------+--------------+
     |              |              |
+----+----+   +----+----+   +----+----+
| ctrl-01 |   | ctrl-02 |   | ctrl-03 |
| AZ-A    |   | AZ-B    |   | Mgmt    |
+---------+   +---------+   +---------+

Each control node runs:
  - All OpenStack API services (containerized)
  - MariaDB Galera member
  - RabbitMQ cluster member
  - Memcached
  - HAProxy + keepalived
  - Ceph MON + MGR
```

### 7.4 OpenStack Project (Tenant) Structure

| Project | Purpose | VRF |
|---------|---------|-----|
| cde-production | Cardholder Data Environment workloads | cde-vrf |
| cde-staging | CDE staging/pre-prod | cde-vrf |
| corporate-production | Non-CDE business applications | corporate-vrf |
| corporate-staging | Non-CDE staging | corporate-vrf |
| dmz | Internet-facing reverse proxies, WAFs | dmz-vrf |
| infrastructure | Monitoring, logging, automation tooling | mgmt-vrf |
| sandbox | Developer experimentation (no CDE access) | corporate-vrf |

### 7.5 Flavor Design

| Flavor | vCPUs | RAM | Disk | Use Case |
|--------|-------|-----|------|----------|
| m1.small | 2 | 4 GB | 40 GB | Small services, jump hosts |
| m1.medium | 4 | 8 GB | 80 GB | Standard application servers |
| m1.large | 8 | 16 GB | 160 GB | Medium application servers |
| m1.xlarge | 16 | 32 GB | 320 GB | Large application servers |
| m1.2xlarge | 32 | 64 GB | 640 GB | Database servers, heavy workloads |
| c1.large | 16 | 16 GB | 80 GB | CPU-intensive workloads |
| r1.large | 8 | 64 GB | 160 GB | Memory-intensive (caches, analytics) |
| r1.xlarge | 16 | 128 GB | 320 GB | Large databases |
| d1.dedicated.xlarge | 16 (pinned) | 32 GB | 320 GB | Latency-sensitive, dedicated cores |

### 7.6 Image Management

- Golden images built with **Packer** on a monthly schedule.
- Base OS: Ubuntu 24.04 LTS and/or RHEL 9 (depending on application requirements).
- Images are CIS Level 1 hardened at build time.
- Cloud-init enabled for first-boot configuration (SSH keys, hostname, network, Ansible bootstrap).
- Images stored in Glance with Ceph RBD backend (copy-on-write clone for fast boot).
- Image pipeline: Packer build -> OpenSCAP scan -> Push to Glance -> Tag as "approved".

---

## 8. Identity and Access Management

### 8.1 Identity Architecture

```
Identity Architecture
=====================

                  +-------------------+
                  |    Keycloak       |  (Central IdP, OIDC/SAML)
                  | (HA, 2 nodes +   |
                  |  PostgreSQL)      |
                  +--------+----------+
                           |
          +----------------+----------------+
          |                |                |
  +-------+------+  +-----+------+  +------+-------+
  |  OpenStack   |  |  Grafana   |  |   NetBox     |
  |  Keystone    |  |            |  |              |
  | (OIDC Fed.)  |  | (OIDC)    |  |  (OIDC)     |
  +--------------+  +------------+  +--------------+
          |
  +-------+------+
  |   FreeIPA    |  (LDAP/Kerberos for Linux hosts)
  | (2 replicas) |
  +--------------+
```

### 8.2 Component Roles

| Component | Role | Details |
|-----------|------|---------|
| **Keycloak** | Central Identity Provider | OIDC and SAML2 federation. SSO for all web UIs. MFA enforcement. User/group management for non-Linux identities. |
| **FreeIPA** | Linux Host Identity | Kerberos authentication, LDAP directory, sudo rules, HBAC (host-based access control), SSH key distribution, certificate authority for host certs. |
| **OpenStack Keystone** | OpenStack IAM | Federated with Keycloak via OIDC. Maps Keycloak groups to OpenStack projects and roles. |
| **Barbican** | Secret Management | API-driven secret storage within OpenStack (TLS certs, encryption keys). |
| **HashiCorp Vault** | Secrets Management | Application-level secrets, dynamic database credentials, Ceph encryption keys, PKI intermediate CA. Deployed in HA (3-node Raft). |

### 8.3 RBAC Model

| Role | Scope | Permissions |
|------|-------|-------------|
| cloud-admin | Global | Full OpenStack admin, Ceph admin, ACI admin |
| project-admin | Per project | Create/delete VMs, volumes, networks within their project |
| project-member | Per project | Create VMs, attach volumes, view resources |
| project-viewer | Per project | Read-only access |
| security-auditor | Global (read-only) | Read-only access to all projects, logs, and audit trails |
| network-admin | Networking | ACI policy management, Neutron admin |

### 8.4 Authentication Requirements (PCI-DSS / SOC2)

- **MFA**: Enforced for all administrative access (Keycloak TOTP/WebAuthn).
- **Password policy**: Minimum 14 characters, complexity requirements, 90-day rotation for service accounts.
- **Session management**: 15-minute idle timeout for Horizon and all admin UIs.
- **Unique IDs**: Every user has a unique account -- no shared or generic accounts.
- **Access logging**: All authentication events logged to SIEM (Keycloak audit log, Keystone CADF audit, FreeIPA audit).
- **Access review**: Quarterly access reviews automated via scripts that compare Keycloak/FreeIPA group membership against HR roster.

---

## 9. Monitoring and Observability

### 9.1 Monitoring Stack

The monitoring stack is built on Prometheus, Grafana, Loki, and Alertmanager -- all FLOSS, battle-tested at scale, and well-understood by the industry.

```
Observability Architecture
==========================

+------------------+     +------------------+     +------------------+
| Compute Nodes    |     | Storage Nodes    |     | Control Plane    |
| - node_exporter  |     | - node_exporter  |     | - node_exporter  |
| - libvirt_exp.   |     | - ceph_exporter  |     | - os_exporter    |
| - opflex_exp.    |     | - smartctl_exp.  |     | - mysql_exporter |
| - promtail       |     | - promtail       |     | - rabbitmq_exp.  |
+--------+---------+     +--------+---------+     | - promtail       |
         |                         |               +--------+---------+
         |                         |                        |
         +------------+------------+------------------------+
                      |
              +-------+--------+
              |  Prometheus    |  (2 instances, HA with Thanos or
              |  (metrics)     |   shared remote-write)
              +-------+--------+
                      |
              +-------+--------+
              |    Thanos      |  (long-term storage, global query)
              | (sidecar +     |
              |  store + query)|
              +-------+--------+
                      |
              +-------+--------+
              |   Grafana      |  (dashboards, OIDC SSO via Keycloak)
              +-------+--------+
                      |
              +-------+--------+
              |  Alertmanager  |  (routing to PagerDuty, Slack, email)
              +----------------+

              +----------------+
              |     Loki       |  (log aggregation)
              | (backend: Ceph |
              |  object store) |
              +----------------+

              +----------------+
              |   Wazuh        |  (SIEM, HIDS, compliance scanning)
              +----------------+
```

### 9.2 Metrics Collection

| Target | Exporter | Key Metrics |
|--------|----------|-------------|
| Linux hosts | node_exporter | CPU, memory, disk, network, filesystem |
| Libvirt/KVM | libvirt_exporter | VM CPU, memory, disk I/O, network I/O |
| Ceph | ceph_exporter (MGR module) | OSD status, PG states, pool utilization, IOPS, latency |
| OpenStack | prometheus-openstack-exporter | Nova instances, Cinder volumes, Neutron ports, Keystone tokens |
| MariaDB | mysqld_exporter | Query rate, slow queries, replication lag, connections |
| RabbitMQ | rabbitmq_exporter | Queue depth, message rate, connection count |
| HAProxy | built-in stats | Backend health, request rate, error rate |
| Cisco ACI | aci_exporter (or SNMP) | Fabric health score, EPG health, contract drops, interface utilization |
| Hardware | IPMI exporter | Temperature, fan speed, PSU status, disk health (SMART) |

### 9.3 Alerting Strategy

Alerts are tiered to avoid fatigue:

| Severity | Route | Example | Response |
|----------|-------|---------|----------|
| P1 - Critical | PagerDuty (wake-up) | Ceph HEALTH_ERR, control plane down, CDE network breach | Immediate response, 15-min SLA |
| P2 - High | PagerDuty (business hours) + Slack | OSD down, compute node unreachable, disk >90% | Same-day response |
| P3 - Warning | Slack + email | Ceph HEALTH_WARN, CPU >80% sustained, certificate expiring | Next business day |
| P4 - Info | Slack (low-priority channel) | Completed backup, successful failover test | Informational, no action |

### 9.4 Dashboards

Pre-built Grafana dashboards for the ops team:

1. **Platform Overview**: Aggregate compute, storage, network utilization. VM count. Ceph health.
2. **Compute Deep Dive**: Per-node CPU, memory, VM density, NUMA utilization.
3. **Ceph Storage**: OSD status, pool utilization, IOPS, latency histograms, recovery progress.
4. **OpenStack Services**: API response times, error rates per service, RabbitMQ queue depths.
5. **ACI Fabric**: Fabric health score, top talkers, contract deny logs, interface errors.
6. **CDE Security**: CDE-specific traffic flows, firewall deny counts, authentication failures.
7. **Capacity Planning**: Utilization trends, projected exhaustion dates (based on linear regression).

### 9.5 Log Aggregation

- **Promtail** agents on all nodes ship logs to **Loki**.
- Log sources: syslog, OpenStack service logs, Ceph logs, ACI audit logs, Keycloak audit logs.
- Retention: 90 days hot (Loki), 1 year cold (Ceph object storage or S3-compatible).
- PCI-DSS Requirement 10: All CDE-related logs are immutable (write-once Loki retention policy), timestamped, and retained for 1 year.

### 9.6 SIEM and Security Monitoring

- **Wazuh** (FLOSS SIEM/HIDS) deployed with:
  - Agents on all hosts (file integrity monitoring, rootkit detection, log analysis).
  - Integration with Loki for centralized correlation.
  - PCI-DSS and SOC2 compliance dashboards and automated scanning.
  - MITRE ATT&CK mapping for detected threats.
- Wazuh receives:
  - ACI contract deny logs (network anomalies).
  - Keycloak/Keystone authentication events (brute force detection).
  - FreeIPA sudo/HBAC events (privilege escalation detection).
  - Host-level syscall auditing (auditd rules for CDE hosts).

---

## 10. Automation and Infrastructure as Code

### 10.1 Automation Strategy

With a 6-person team, automation is not optional -- it is the primary mechanism for operating the platform. The goal is >95% automation coverage for all day-1 and day-2 operations.

### 10.2 Toolchain

| Tool | Purpose | Scope |
|------|---------|-------|
| **Ansible** (via AWX) | Configuration management, day-2 operations | Host configuration, OpenStack deployment (Kolla-Ansible), Ceph deployment (cephadm), firmware updates, compliance remediation |
| **OpenTofu** | Infrastructure provisioning | OpenStack resources (VMs, networks, volumes), ACI policies (via Cisco ACI provider), DNS records |
| **Packer** | Image building | Golden VM images (Ubuntu, RHEL) with CIS hardening |
| **AWX** | Ansible GUI/scheduler | Job scheduling, RBAC for playbook execution, audit trail |
| **Git (Gitea or GitLab CE)** | Version control | All IaC, playbooks, configurations stored in Git |
| **CI/CD (GitLab CI or Gitea Actions)** | Pipeline automation | Lint -> Test -> Plan -> Apply for all infrastructure changes |

### 10.3 Repository Structure

```
infrastructure/
  |- ansible/
  |    |- inventory/
  |    |    |- production/
  |    |    |- staging/
  |    |- playbooks/
  |    |    |- site.yml
  |    |    |- openstack-deploy.yml
  |    |    |- ceph-deploy.yml
  |    |    |- hardening.yml
  |    |    |- monitoring.yml
  |    |    |- patching.yml
  |    |- roles/
  |    |    |- common/
  |    |    |- hardening/
  |    |    |- monitoring-agent/
  |    |    |- ceph-client/
  |    |    |- nftables/
  |    |- collections/
  |         |- requirements.yml  (openstack.cloud, cisco.aci, community.general)
  |
  |- tofu/
  |    |- modules/
  |    |    |- openstack-project/
  |    |    |- openstack-vm/
  |    |    |- aci-tenant/
  |    |    |- aci-epg/
  |    |    |- dns-record/
  |    |- environments/
  |         |- production/
  |         |    |- cde/
  |         |    |- corporate/
  |         |    |- dmz/
  |         |- staging/
  |
  |- packer/
  |    |- ubuntu-2404/
  |    |- rhel-9/
  |    |- hardening-scripts/
  |
  |- docs/
       |- architecture/
       |- runbooks/
       |- adr/  (Architecture Decision Records)
```

### 10.4 CI/CD Pipeline for Infrastructure Changes

```
Git Push -> Lint (ansible-lint, tofu validate) -> Security Scan (checkov, tfsec)
  -> Plan (tofu plan) -> Manual Approval (for production) -> Apply (tofu apply)
  -> Smoke Test (ansible playbook to validate) -> Notify (Slack)
```

- All infrastructure changes go through pull/merge requests.
- Two-person approval required for CDE changes (PCI-DSS Requirement 6).
- Full audit trail in Git and AWX.

### 10.5 Day-2 Automation Playbooks

| Playbook | Trigger | Description |
|----------|---------|-------------|
| `patching.yml` | Monthly schedule (AWX) | OS patching with rolling restart, Ceph maintenance mode |
| `cert-renewal.yml` | 30 days before expiry (AWX) | Renew TLS certificates from Vault PKI |
| `user-offboard.yml` | HR webhook / manual | Disable user in Keycloak, FreeIPA, revoke all tokens |
| `scale-compute.yml` | Manual | Add a new compute node to the cluster |
| `ceph-maintenance.yml` | Manual | Set OSD maintenance flags, drain, replace disk |
| `dr-failover.yml` | Manual | Orchestrated failover to DR site |
| `compliance-scan.yml` | Weekly (AWX) | Run OpenSCAP on all hosts, report to Wazuh |
| `backup-verify.yml` | Weekly (AWX) | Restore a random backup and validate integrity |

### 10.6 Event-Driven Ansible (EDA)

For automated remediation of common issues:

| Event | Source | Action |
|-------|--------|--------|
| Ceph OSD down | Prometheus alert -> Alertmanager webhook | Run diagnostics playbook, attempt restart, alert if failed |
| Disk SMART warning | node_exporter alert | Create Jira ticket, schedule disk replacement |
| SSH brute force detected | Wazuh alert | Add source IP to nftables blocklist |
| Certificate expiring (7 days) | Prometheus cert_exporter | Auto-renew via Vault PKI |

---

## 11. Security Architecture

### 11.1 Security Layers

```
Security Defense in Depth
=========================

Layer 1: Physical
  - Locked data center, badge access, CCTV
  - Visitor logs, escort requirements

Layer 2: Network
  - ACI microsegmentation (contracts = whitelist only)
  - Inter-VRF firewalls (CDE isolation)
  - Perimeter IDS/IPS (Suricata)
  - 802.1X on management ports (optional, via Cisco ISE)

Layer 3: Host
  - CIS Level 1 hardened OS images
  - nftables host firewall (Ansible-managed)
  - SELinux or AppArmor enforcing
  - FreeIPA HBAC (host-based access control)
  - auditd with PCI-relevant rules
  - Wazuh HIDS agent
  - Automatic security updates (unattended-upgrades)

Layer 4: Application
  - Keycloak MFA for all admin access
  - Vault for secrets management (no plaintext secrets)
  - TLS everywhere (internal and external)
  - OpenSCAP compliance scanning

Layer 5: Data
  - Ceph dm-crypt encryption at rest
  - Ceph messenger v2 encryption in transit
  - Barbican for OpenStack secret storage
  - Key rotation policies

Layer 6: Audit
  - Centralized logging to Loki (immutable)
  - SIEM correlation (Wazuh)
  - CADF audit events from Keystone
  - ACI contract logging
  - Quarterly access reviews
```

### 11.2 PCI-DSS Compliance Mapping

| PCI-DSS Requirement | Implementation |
|---------------------|----------------|
| **1. Network segmentation** | ACI VRF isolation for CDE, contracts as whitelist firewall, inter-VRF firewall service graph |
| **2. Secure configuration** | CIS-hardened images (Packer), OpenSCAP weekly scans, Ansible-enforced configuration |
| **3. Protect stored data** | Ceph dm-crypt at rest, Vault for key management, data classification in OpenStack metadata |
| **4. Encrypt transmission** | TLS 1.2+ everywhere, Ceph msgr2 encryption, ACI fabric encryption |
| **5. Anti-malware** | ClamAV on CDE hosts (via Ansible), Wazuh rootkit detection |
| **6. Secure development** | Two-person approval for CDE changes, CI/CD pipeline with security scanning |
| **7. Restrict access** | RBAC via Keystone, FreeIPA HBAC, least-privilege principle, no shared accounts |
| **8. Identify and authenticate** | Keycloak MFA, unique user IDs, 15-min session timeout, password policy enforcement |
| **9. Physical security** | Badge access, CCTV, visitor logs (data center provider responsibility if colo) |
| **10. Logging and monitoring** | Loki (immutable, 1-year retention), Wazuh SIEM, Alertmanager for real-time alerts |
| **11. Security testing** | Quarterly vulnerability scans (OpenVAS/Nessus), annual penetration test, Wazuh continuous monitoring |
| **12. Security policies** | Documented in Git, reviewed quarterly, incident response plan tested annually |

### 11.3 SOC2 Trust Service Criteria Mapping

| Criteria | Implementation |
|----------|----------------|
| **Security** | All PCI-DSS controls above, plus ACI microsegmentation, Wazuh HIDS, defense-in-depth |
| **Availability** | HA architecture (no SPOF), Ceph 3x replication, multi-AZ compute, DR plan with tested failover |
| **Processing Integrity** | Input validation at application layer, audit trails for all state changes, checksums on backups |
| **Confidentiality** | Encryption at rest and in transit, RBAC, network segmentation, data classification |
| **Privacy** | Data minimization, access controls, audit logging, retention policies |

### 11.4 TLS and Certificate Management

- **Internal PKI**: HashiCorp Vault as intermediate CA (root CA offline, air-gapped).
- **Certificate lifecycle**: Vault issues short-lived certificates (30-day TTL) to all services. Auto-renewal via Ansible/cert-manager.
- **TLS standards**: TLS 1.2 minimum, TLS 1.3 preferred. Strong cipher suites only (ECDHE, AES-GCM).
- **Service mesh consideration**: For future Kubernetes workloads, use Cilium or Istio for mTLS between services.

---

## 12. Disaster Recovery

### 12.1 DR Strategy

| Tier | Workloads | RPO | RTO | Method |
|------|-----------|-----|-----|--------|
| Tier 1 | CDE databases, payment processing | 15 min | 1 hour | Ceph RBD async mirroring to DR site + warm standby VMs |
| Tier 2 | CDE application servers | 1 hour | 4 hours | VM snapshots replicated to DR, Heat stacks for rebuild |
| Tier 3 | Corporate applications | 4 hours | 8 hours | Daily backups to DR, rebuild from IaC |
| Tier 4 | Dev/sandbox | 24 hours | 24 hours | Daily backups only |

### 12.2 DR Site Options

**Option A: Second data center (recommended)**
- Minimum viable DR site: 2 compute nodes, 3 Ceph OSD nodes, 1 control node.
- Ceph RBD mirroring (journal-based, async) for Tier 1 data.
- ACI Multi-Site or site-to-site VPN for network extension.
- Heat stacks and Ansible playbooks to rebuild workloads from IaC.

**Option B: Cloud-based DR**
- Replicate critical backups to AWS S3 (encrypted, cross-account) or Azure Blob.
- Maintain OpenTofu definitions for rebuilding in public cloud as emergency fallback.
- Useful as secondary DR target, not primary (data sovereignty considerations for finserv).

### 12.3 DR Procedures

- **Failover playbook** (`dr-failover.yml`): Automated via Ansible, promotes Ceph mirrored images, starts VMs from Heat stacks, updates DNS.
- **DR drill**: Quarterly, full failover test of Tier 1 workloads. Results documented and reviewed.
- **Runbook**: Stored in Git, versioned, reviewed quarterly. Covers step-by-step for each DR scenario.

---

## 13. Capacity Planning and Growth

### 13.1 Current Capacity vs Demand

| Resource | Capacity | Day-1 Demand (500 VMs) | Utilization |
|----------|----------|----------------------|-------------|
| vCPUs | 6,144 (at 4:1) | ~2,000 | 33% |
| RAM | ~9,830 GB (at 1.2:1) | ~4,000 GB | 41% |
| Storage | ~61 TB (3x repl.) | ~30 TB | 49% |
| Network | 25G per node | Varies | <30% typical |

### 13.2 Scaling Path

- **Compute**: Add 2-node increments per AZ. Each pair adds ~384 vCPUs and 1,024 GB RAM.
- **Storage**: Add OSD nodes in groups of 3 (one per failure domain). Each adds ~92 TB raw.
- **Network**: ACI leaf switches support 48 ports each; current design has headroom for 2x current node count before needing additional leaves.

### 13.3 Procurement Cycle

- Monitor utilization trends in Grafana capacity dashboard.
- Trigger procurement review when any resource hits 70% sustained utilization.
- Hardware lead time: 6-12 weeks typical. Maintain 1 spare compute node and 2 spare disks for emergency replacement.

---

## 14. Operational Procedures for the 6-Person Team

### 14.1 Team Role Distribution

| Role | Count | Responsibilities |
|------|-------|-----------------|
| Cloud Platform Lead | 1 | Architecture decisions, capacity planning, vendor relationships, escalation |
| OpenStack/Ceph Engineer | 2 | OpenStack operations, Ceph operations, Kolla upgrades, storage tuning |
| Network/Security Engineer | 1 | ACI fabric, firewalls, IDS/IPS, compliance scanning, security incidents |
| Automation Engineer | 1 | Ansible, OpenTofu, CI/CD pipelines, image builds, AWX administration |
| Monitoring/SRE | 1 | Prometheus, Grafana, Loki, Wazuh, alerting, on-call rotation support |

All 6 engineers participate in the on-call rotation (one week on, five weeks off). Cross-training is essential -- everyone should be able to perform basic operations on all subsystems.

### 14.2 On-Call and Escalation

- **On-call rotation**: 1 engineer on primary, 1 on secondary (backup). PagerDuty manages rotation and escalation.
- **Escalation**: P1 alerts auto-page primary. If no ack in 10 minutes, escalate to secondary. If no ack in 20 minutes, escalate to team lead.
- **Runbooks**: Every P1 and P2 alert has a linked runbook in Git with step-by-step remediation.

### 14.3 Change Management

- All changes go through Git (merge request).
- Standard changes (pre-approved types like VM creation, user onboarding): self-service or auto-approved.
- Normal changes (e.g., new ACI contract, new subnet): peer review + one approval.
- Emergency changes: single approval, post-incident review within 24 hours.
- CDE changes: two-person approval, documented in change log.

### 14.4 Maintenance Windows

- **Monthly**: OS patching, firmware updates (rolling, zero-downtime for compute via live migration).
- **Quarterly**: OpenStack minor upgrades, Ceph version updates, DR drill.
- **Annually**: OpenStack major upgrade (via Kolla-Ansible), full DR test, penetration test, architecture review.

---

## 15. Network Diagram (Logical)

```
                              INTERNET
                                 |
                         +-------+-------+
                         | Perimeter FW  |
                         | (HA pair)     |
                         +-------+-------+
                                 |
                         +-------+-------+
                         |  Border Leaf  |
                         | (2x Nexus 9k) |
                         +-------+-------+
                                 |
                    +------------+------------+
                    |                         |
             +------+------+          +------+------+
             | Spine-A     |          | Spine-B     |
             | (Nexus 9500)|          | (Nexus 9500)|
             +------+------+          +------+------+
                    |                         |
       +------------+------------+------------+------------+
       |            |            |            |            |
  +----+----+  +----+----+  +----+----+  +----+----+  +----+----+
  | Leaf-A1 |  | Leaf-A2 |  | Leaf-B1 |  | Leaf-B2 |  |Leaf-Mgmt|
  | (vPC)   |  | (vPC)   |  | (vPC)   |  | (vPC)   |  | (vPC)   |
  +----+----+  +----+----+  +----+----+  +----+----+  +----+----+
       |            |            |            |            |
  +----+----+  +----+----+  +----+----+  +----+----+  +----+----+
  |Compute  |  |Compute  |  |Compute  |  |Compute  |  |Control  |
  |Rack A1  |  |Rack A2  |  |Rack B1  |  |Rack B2  |  |Plane +  |
  |4x comp  |  |4x comp  |  |4x comp  |  |4x comp  |  |Monitor  |
  |3x Ceph  |  |         |  |3x Ceph  |  |         |  |APIC x3  |
  +---------+  +---------+  +---------+  +---------+  +---------+

      AZ-A                        AZ-B                    MGMT
```

---

## 16. Bill of Materials (Summary)

| Component | Quantity | Notes |
|-----------|----------|-------|
| Cisco Nexus 9500/9336C (spine) | 2 | Already procured |
| Cisco Nexus 9300 (leaf) | 10 | Already procured (verify count) |
| Cisco APIC | 3 | Already procured |
| Compute nodes (2U, 96C, 512GB) | 16 | 8 per AZ |
| Ceph storage nodes (2U, 8x NVMe) | 6 | 3 per AZ |
| Control plane nodes (1U, 256GB) | 3 | Spread across AZs + mgmt |
| Perimeter firewall (HA pair) | 2 | Cisco Firepower or OPNsense |
| 25GbE SFP28 transceivers | ~100 | Server to leaf connections |
| 100GbE QSFP28 transceivers | ~40 | Leaf to spine uplinks |
| PDU (dual, managed) | 20 | 2 per rack |
| UPS | Per facility | Existing or new |

### Software Licensing

| Software | License | Cost |
|----------|---------|------|
| OpenStack (Kolla-Ansible) | Apache 2.0 | Free |
| Ceph | LGPL | Free |
| Ubuntu LTS | Free / Ubuntu Pro optional | Free or ~$500/node/yr for Pro |
| Keycloak | Apache 2.0 | Free |
| FreeIPA | GPL | Free |
| Prometheus / Grafana OSS | Apache 2.0 / AGPL | Free |
| Loki | AGPL | Free |
| Wazuh | GPL | Free |
| AWX | Apache 2.0 | Free |
| OpenTofu | MPL 2.0 | Free |
| NetBox | Apache 2.0 | Free |
| PowerDNS | GPL | Free |
| HashiCorp Vault | BSL (free for self-hosted use) | Free |
| Cisco ACI | SmartNet + ACI license | Per existing Cisco contract |
| Bareos | AGPL | Free (or commercial support) |

**Key cost advantage**: The FLOSS stack eliminates annual licensing costs for hypervisor (vs. VMware), storage (vs. NetApp/Pure), monitoring (vs. Datadog/Splunk), and identity (vs. Okta/AD). The primary ongoing costs are Cisco SmartNet, hardware maintenance contracts, optional Ubuntu Pro or RHEL subscriptions, and team salaries.

---

## 17. Architecture Decision Records (ADRs)

### ADR-001: OpenStack over Proxmox or VMware

**Context**: Need IaaS platform for 500 VMs with API-driven self-service.

**Decision**: OpenStack (Kolla-Ansible deployment).

**Rationale**: OpenStack provides a full API-compatible IaaS with native Cisco ACI integration (Neutron ML2 plugin), Ceph integration (Cinder, Glance, Nova), and a mature tenant/RBAC model. Proxmox is simpler but lacks native ACI integration and enterprise multi-tenancy. VMware (post-Broadcom) is cost-prohibitive and adds license dependency.

### ADR-002: Ceph over commercial SAN

**Context**: Need block, object, and optionally file storage.

**Decision**: Ceph with all-NVMe OSDs.

**Rationale**: Ceph provides unified block (RBD), object (RGW), and file (CephFS) storage. Native integration with OpenStack. No per-TB licensing. Hardware-agnostic. The team can scale by adding OSD nodes. Commercial SAN (NetApp, Pure) would add $200k+ in licensing for equivalent capacity.

### ADR-003: Keycloak + FreeIPA over Active Directory

**Context**: Need centralized identity for cloud platform and Linux hosts.

**Decision**: Keycloak for web SSO/OIDC, FreeIPA for Linux host identity.

**Rationale**: FLOSS stack eliminates AD/Windows Server licensing. Keycloak provides OIDC/SAML federation that integrates with OpenStack Keystone, Grafana, NetBox, and AWX. FreeIPA provides Kerberos, LDAP, sudo rules, and HBAC for Linux. If the organization already has AD for Windows endpoints, Keycloak can federate with AD as an upstream IdP.

### ADR-004: Wazuh over Splunk for SIEM

**Context**: Need SIEM for PCI-DSS and SOC2 compliance.

**Decision**: Wazuh.

**Rationale**: FLOSS, purpose-built for compliance (PCI-DSS, SOC2 dashboards out of the box), includes HIDS agent (file integrity, rootkit detection), and integrates with the Prometheus/Loki stack. Splunk licensing for the log volume expected (500 VMs + infrastructure) would cost $100k+/year.

### ADR-005: Kolla-Ansible deployment method

**Context**: Need a supportable, upgradeable OpenStack deployment.

**Decision**: Kolla-Ansible.

**Rationale**: Containerized deployment simplifies upgrades (pull new images, run upgrade playbook). Well-documented. Active upstream community. Alternative approaches (TripleO/director) are being deprecated. Kayobe is a viable alternative but adds abstraction. Sunbeam (Canonical) is too new for production finserv.

---

## 18. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Ceph data loss (triple failure) | Very Low | Critical | 3x replication across failure domains, regular scrubbing, off-site backups |
| OpenStack upgrade failure | Medium | High | Kolla-Ansible upgrade tested in staging first, snapshot control plane before upgrade, rollback procedure documented |
| ACI APIC cluster failure | Low | Critical | 3-node APIC cluster tolerates 1 failure, fabric continues forwarding even if all APICs are down (last-known-good policy) |
| Key person dependency | Medium | High | Cross-training mandate, runbooks for all procedures, pair operations for complex tasks |
| Compliance audit finding | Medium | Medium | Continuous compliance scanning (OpenSCAP, Wazuh), quarterly internal audits, pre-audit checklist |
| Hardware supply chain delay | Medium | Medium | Maintain spare compute node and spare disks, 70% utilization procurement trigger |
| Security breach in CDE | Low | Critical | Defense-in-depth, microsegmentation, SIEM alerting, incident response plan, annual pen test |

---

## 19. Implementation Roadmap

| Phase | Duration | Deliverables |
|-------|----------|-------------|
| **Phase 1: Foundation** | Weeks 1-4 | Rack and stack hardware, cable, configure ACI fabric, APIC policies, OOB management network, base OS on all nodes |
| **Phase 2: Storage** | Weeks 3-6 | Deploy Ceph cluster (cephadm), create pools, benchmark performance, configure encryption |
| **Phase 3: Control Plane** | Weeks 5-8 | Deploy OpenStack via Kolla-Ansible, integrate Neutron with ACI, integrate Cinder/Glance with Ceph, deploy Keystone with Keycloak federation |
| **Phase 4: Identity and Security** | Weeks 6-9 | Deploy Keycloak, FreeIPA, Vault. Configure MFA, RBAC, TLS PKI. CIS hardening. Wazuh deployment. |
| **Phase 5: Monitoring** | Weeks 7-10 | Deploy Prometheus, Grafana, Loki, Alertmanager. Build dashboards. Configure alerting and PagerDuty. |
| **Phase 6: Automation** | Weeks 8-11 | AWX deployment, CI/CD pipelines, Packer image builds, OpenTofu modules, day-2 playbooks |
| **Phase 7: Pilot Workloads** | Weeks 10-13 | Migrate 20-30 non-CDE VMs, validate end-to-end, performance test, team training |
| **Phase 8: CDE Migration** | Weeks 13-18 | Migrate CDE workloads in waves, compliance validation, QSA pre-assessment |
| **Phase 9: Full Migration** | Weeks 16-22 | Remaining VM migration, AWS decommission planning, DR testing, production handoff |
| **Phase 10: Hardening** | Weeks 20-24 | Penetration test, compliance audit prep, runbook finalization, team certification |

---

## 20. Summary

This architecture delivers a production-grade private cloud for a financial services organization with the following characteristics:

- **Platform**: OpenStack (Kolla-Ansible) with Cisco ACI networking and Ceph storage.
- **Capacity**: 500+ VMs with ~60% headroom for growth before additional hardware.
- **Availability**: 99.99% design target with multi-AZ, no single point of failure.
- **Compliance**: PCI-DSS and SOC2 controls implemented at every layer (network, host, data, audit).
- **Security**: Defense-in-depth with ACI microsegmentation, encryption at rest and in transit, MFA, SIEM, and continuous compliance scanning.
- **Operations**: Fully automated via Ansible/OpenTofu/AWX, manageable by 6 engineers with on-call rotation.
- **Cost**: FLOSS stack minimizes licensing. Primary costs are hardware CapEx, Cisco SmartNet, and team salaries. Target 35-45% TCO reduction versus AWS over 5 years.
- **DR**: Ceph RBD mirroring for Tier 1 (15-min RPO), IaC-based rebuild for lower tiers, quarterly DR drills.

The architecture is designed to be operationally sustainable for a small team through automation, standardization, and well-documented runbooks. Every component can be scaled independently as the organization grows.
