# OpenStack on Cisco ACI Multi-Tenant Private Cloud Architecture

## 1. Executive Summary

This document describes the architecture for deploying OpenStack on a Cisco ACI fabric (Nexus 9000 spine-leaf with a 3-node APIC cluster) to serve five internal business units (BUs). The design provides full network isolation per BU, shared services reachable from all tenants, north-south firewalling through Palo Alto VM-Series, and load balancing for web workloads via F5 BIG-IP VE integrated as ACI service graphs. Automation is handled through a combination of the Cisco ACI OpenStack plugin (GBP or unified plugin), Ansible, Terraform, and APIC REST APIs.

---

## 2. Physical Fabric Topology

### 2.1 Spine-Leaf Layout

```
                    +-----------+
                    |  Internet |
                    +-----+-----+
                          |
               +----------+----------+
               | Border Leaf Pair    |
               | (N9K-1/N9K-2) vPC  |
               +----+----------+----+
                    |          |
         +----------+----------+----------+
         |          |          |          |
    +----+----+  +--+------+  +--+------+  +--+------+
    | Spine-1 |  | Spine-2 |  | Spine-3 |  | Spine-4 |
    +---------+  +---------+  +---------+  +---------+
         |  \  /  |    |  \  /  |    |  \  /  |
    +----+----+  +--+------+  +--+------+  +--+------+
    | Leaf-1  |  | Leaf-2  |  | Leaf-3  |  | Leaf-4  |
    | Pair-A  |  | Pair-B  |  | Pair-C  |  | Pair-D  |
    +---------+  +---------+  +---------+  +---------+
        |            |            |            |
    Compute/     Compute/     Compute/     Service
    Control      Control      Control      Nodes
    Rack-1       Rack-2       Rack-3       (F5, PAN)
```

### 2.2 Hardware Bill of Materials

| Component | Role | Quantity | Notes |
|-----------|------|----------|-------|
| Nexus 9336C-FX2 | Spine | 4 | 100G uplinks |
| Nexus 93180YC-FX | Leaf (compute) | 6 (3 vPC pairs) | 48x 1/10/25G + 6x 100G |
| Nexus 93180YC-FX | Leaf (border) | 2 (1 vPC pair) | External L3 connectivity |
| Nexus 93180YC-FX | Leaf (service) | 2 (1 vPC pair) | F5, Palo Alto attachment |
| APIC-SERVER-L3 | APIC Controller | 3 | Cluster for HA |

### 2.3 APIC Cluster

- Three APICs connected to separate leaf pairs for fault isolation.
- APIC-1 on Leaf-Pair-A, APIC-2 on Leaf-Pair-B, APIC-3 on Leaf-Pair-C.
- Fabric discovery, policy repository, and DHCP relay all triply redundant.

---

## 3. ACI Object Model Design

### 3.1 Tenant Hierarchy

```
ACI Tenants
|
+-- common                         (ACI built-in, shared contracts/filters)
|
+-- tn-infra-openstack             (OpenStack control plane)
|   +-- VRF: vrf-openstack-mgmt
|   +-- BD:  bd-openstack-api
|   +-- BD:  bd-openstack-mgmt
|   +-- EPG: epg-keystone
|   +-- EPG: epg-neutron
|   +-- EPG: epg-nova
|   +-- EPG: epg-glance
|   +-- EPG: epg-cinder
|   +-- EPG: epg-rabbitmq
|   +-- EPG: epg-mariadb
|
+-- tn-shared-services             (DNS, LDAP, Monitoring)
|   +-- VRF: vrf-shared
|   +-- BD:  bd-shared-infra
|   +-- EPG: epg-dns
|   +-- EPG: epg-ldap
|   +-- EPG: epg-monitoring
|   +-- L3Out: l3out-shared-to-tenants  (leaked routes)
|
+-- tn-bu1-engineering             (Business Unit 1)
|   +-- VRF: vrf-bu1
|   +-- BD:  bd-bu1-web
|   +-- BD:  bd-bu1-app
|   +-- BD:  bd-bu1-db
|   +-- EPG: epg-bu1-web
|   +-- EPG: epg-bu1-app
|   +-- EPG: epg-bu1-db
|
+-- tn-bu2-finance                 (Business Unit 2)
|   +-- VRF: vrf-bu2
|   +-- (same BD/EPG pattern as BU1)
|
+-- tn-bu3-marketing               (Business Unit 3)
|   +-- VRF: vrf-bu3
|   +-- (same BD/EPG pattern)
|
+-- tn-bu4-operations              (Business Unit 4)
|   +-- VRF: vrf-bu4
|   +-- (same BD/EPG pattern)
|
+-- tn-bu5-research                (Business Unit 5)
|   +-- VRF: vrf-bu5
|   +-- (same BD/EPG pattern)
|
+-- tn-dmz                         (DMZ for internet-facing services)
|   +-- VRF: vrf-dmz
|   +-- BD:  bd-dmz-outside
|   +-- BD:  bd-dmz-inside
|   +-- EPG: epg-dmz-outside
|   +-- EPG: epg-dmz-inside
|   +-- Service Graph: sg-paloalto-fw
|   +-- Service Graph: sg-f5-lb
|
+-- tn-l3out-external              (External connectivity)
    +-- VRF: vrf-external
    +-- L3Out: l3out-internet
    +-- External EPG: ext-epg-internet (0.0.0.0/0)
```

### 3.2 VRF Design and Route Leaking

Each BU tenant gets its own VRF for full routing isolation. Shared services are accessed through ACI's inter-VRF contract-based route leaking (vzAny or explicit contract export).

```
VRF Relationships:

  vrf-external ----[L3Out/BGP]----> Internet (AS 65000)
       |
       | (contract: ctr-internet-to-dmz)
       v
  vrf-dmz ----------[PAN FW SG]-------+
       |                               |
       | (contract: ctr-dmz-to-bu)     |
       v                               v
  vrf-bu1, vrf-bu2, ...          vrf-shared
       |                               |
       +------[leaked contract]--------+
              ctr-shared-services
```

**Route Leaking for Shared Services:**

- `tn-shared-services/vrf-shared` exports `ctr-shared-services` (provider).
- Each `vrf-buN` imports that contract (consumer).
- ACI handles route leaking between VRFs automatically when a contract is consumed across VRF boundaries.
- Subnet `10.200.0.0/24` (shared services) is marked as "shared" on the BD subnet configuration.

### 3.3 Bridge Domain Design

| Tenant | Bridge Domain | Subnet | Gateway | Properties |
|--------|--------------|--------|---------|------------|
| tn-infra-openstack | bd-openstack-api | 10.100.1.0/24 | 10.100.1.1 | Unicast routing, ARP flooding off |
| tn-infra-openstack | bd-openstack-mgmt | 10.100.2.0/24 | 10.100.2.1 | Unicast routing |
| tn-shared-services | bd-shared-infra | 10.200.0.0/24 | 10.200.0.1 | Shared subnet flag, unicast routing |
| tn-bu1-engineering | bd-bu1-web | 10.1.10.0/24 | 10.1.10.1 | Unicast routing, L2 unknown flood off |
| tn-bu1-engineering | bd-bu1-app | 10.1.20.0/24 | 10.1.20.1 | Unicast routing |
| tn-bu1-engineering | bd-bu1-db | 10.1.30.0/24 | 10.1.30.1 | Unicast routing |
| tn-bu2-finance | bd-bu2-web | 10.2.10.0/24 | 10.2.10.1 | Same pattern |
| tn-bu2-finance | bd-bu2-app | 10.2.20.0/24 | 10.2.20.1 | |
| tn-bu2-finance | bd-bu2-db | 10.2.30.0/24 | 10.2.30.1 | |
| tn-bu3-marketing | bd-bu3-web | 10.3.10.0/24 | 10.3.10.1 | Same pattern |
| tn-bu3-marketing | bd-bu3-app | 10.3.20.0/24 | 10.3.20.1 | |
| tn-bu3-marketing | bd-bu3-db | 10.3.30.0/24 | 10.3.30.1 | |
| tn-bu4-operations | bd-bu4-web | 10.4.10.0/24 | 10.4.10.1 | Same pattern |
| tn-bu4-operations | bd-bu4-app | 10.4.20.0/24 | 10.4.20.1 | |
| tn-bu4-operations | bd-bu4-db | 10.4.30.0/24 | 10.4.30.1 | |
| tn-bu5-research | bd-bu5-web | 10.5.10.0/24 | 10.5.10.1 | Same pattern |
| tn-bu5-research | bd-bu5-app | 10.5.20.0/24 | 10.5.20.1 | |
| tn-bu5-research | bd-bu5-db | 10.5.30.0/24 | 10.5.30.1 | |
| tn-dmz | bd-dmz-outside | 198.51.100.0/28 | 198.51.100.1 | Public-facing |
| tn-dmz | bd-dmz-inside | 10.250.0.0/24 | 10.250.0.1 | Internal DMZ |

**BD Configuration Standards:**

- Unicast routing: enabled on all BDs.
- ARP flooding: disabled (use ARP gleaning with hardware proxy for scale).
- L2 unknown unicast: hardware proxy mode (no flooding).
- GARP-based EP move detection: enabled.
- Endpoint dataplane learning: enabled.
- Subnet scope on BU web BDs: "advertised externally" for those requiring internet access via the DMZ.

### 3.4 EPG and Contract Design

#### 3.4.1 Filters (defined in tenant `common`)

```
flt-allow-web:        TCP dst 80, 443
flt-allow-app:        TCP dst 8080, 8443
flt-allow-db:         TCP dst 3306 (MySQL), 5432 (PostgreSQL)
flt-allow-dns:        TCP/UDP dst 53
flt-allow-ldap:       TCP dst 389, 636
flt-allow-monitoring: TCP dst 9090 (Prometheus), 9100 (node_exporter),
                      UDP dst 514 (syslog), TCP dst 443 (Grafana)
flt-allow-ssh:        TCP dst 22
flt-allow-icmp:       IP protocol ICMP
flt-allow-https:      TCP dst 443
flt-allow-any:        IP protocol unspecified (used sparingly)
```

#### 3.4.2 Contracts

**Intra-BU Contracts (per tenant, same pattern):**

```
ctr-bu1-web-to-app:
  Provider: epg-bu1-app
  Consumer: epg-bu1-web
  Filters:  flt-allow-app
  Scope:    Tenant

ctr-bu1-app-to-db:
  Provider: epg-bu1-db
  Consumer: epg-bu1-app
  Filters:  flt-allow-db
  Scope:    Tenant
```

**Shared Services Contract (tenant `common`):**

```
ctr-shared-services:
  Provider: epg-dns, epg-ldap, epg-monitoring (in tn-shared-services)
  Consumer: vzAny in each vrf-buN
  Filters:  flt-allow-dns, flt-allow-ldap, flt-allow-monitoring
  Scope:    Global

  Implementation detail:
  - The contract is defined in tn-shared-services.
  - Each BU tenant's VRF has a "consumed contract interface"
    pointing to this contract.
  - ACI automatically leaks /32 or /24 routes for the shared
    services subnets into each BU VRF.
```

**Internet-to-DMZ Contract (through Palo Alto service graph):**

```
ctr-internet-to-dmz:
  Provider: epg-dmz-outside
  Consumer: ext-epg-internet
  Filters:  flt-allow-https
  Scope:    Global
  Service Graph: sg-paloalto-fw  (redirect via PBR)
```

**DMZ-to-BU Web Tier Contract (through F5 service graph):**

```
ctr-dmz-to-web:
  Provider: epg-buN-web (each BU that needs external access)
  Consumer: epg-dmz-inside
  Filters:  flt-allow-web
  Scope:    Global
  Service Graph: sg-f5-lb  (redirect via PBR)
```

#### 3.4.3 vzAny Usage

For each BU VRF, a `vzAny` preferred group is optionally enabled to allow intra-VRF communication where explicit contracts are too burdensome, though the recommended approach is explicit contracts for auditability. `vzAny` is used specifically for the shared-services consumption to avoid configuring the same contract consumption on every EPG individually.

### 3.5 Full ACI Object Model Diagram

```
tenant: common
  +-- filter: flt-allow-web         (TCP 80,443)
  +-- filter: flt-allow-app         (TCP 8080,8443)
  +-- filter: flt-allow-db          (TCP 3306,5432)
  +-- filter: flt-allow-dns         (TCP/UDP 53)
  +-- filter: flt-allow-ldap        (TCP 389,636)
  +-- filter: flt-allow-monitoring  (TCP 9090,9100, UDP 514, TCP 443)
  +-- filter: flt-allow-ssh         (TCP 22)
  +-- filter: flt-allow-icmp        (ICMP)
  +-- filter: flt-allow-https       (TCP 443)

tenant: tn-shared-services
  +-- VRF: vrf-shared
  +-- BD: bd-shared-infra --> vrf-shared
  |     subnet: 10.200.0.0/24 (scope: shared, advertised)
  +-- AP: ap-shared
  |   +-- EPG: epg-dns          --> bd-shared-infra
  |   +-- EPG: epg-ldap         --> bd-shared-infra
  |   +-- EPG: epg-monitoring   --> bd-shared-infra
  +-- Contract: ctr-shared-services (exported)
        subject: subj-dns  --> flt-allow-dns
        subject: subj-ldap --> flt-allow-ldap
        subject: subj-mon  --> flt-allow-monitoring

tenant: tn-bu1-engineering           (template for all BU tenants)
  +-- VRF: vrf-bu1
  |     consumed contract interface: ctr-shared-services
  +-- BD: bd-bu1-web --> vrf-bu1
  |     subnet: 10.1.10.0/24
  +-- BD: bd-bu1-app --> vrf-bu1
  |     subnet: 10.1.20.0/24
  +-- BD: bd-bu1-db  --> vrf-bu1
  |     subnet: 10.1.30.0/24
  +-- AP: ap-bu1
  |   +-- EPG: epg-bu1-web  --> bd-bu1-web  (provides: ctr-dmz-to-web)
  |   +-- EPG: epg-bu1-app  --> bd-bu1-app
  |   +-- EPG: epg-bu1-db   --> bd-bu1-db
  +-- Contract: ctr-bu1-web-to-app
  +-- Contract: ctr-bu1-app-to-db

tenant: tn-dmz
  +-- VRF: vrf-dmz
  +-- BD: bd-dmz-outside --> vrf-dmz
  +-- BD: bd-dmz-inside  --> vrf-dmz
  +-- AP: ap-dmz
  |   +-- EPG: epg-dmz-outside --> bd-dmz-outside
  |   +-- EPG: epg-dmz-inside  --> bd-dmz-inside
  +-- L3Out: l3out-internet --> vrf-dmz
  |   +-- ExtEPG: ext-epg-internet (0.0.0.0/0)
  +-- Service Graph Template: sg-paloalto-fw
  +-- Service Graph Template: sg-f5-lb
  +-- Device Selection Policy (for each SG)
  +-- Contract: ctr-internet-to-dmz (SG: sg-paloalto-fw)
  +-- Contract: ctr-dmz-to-web      (SG: sg-f5-lb)
```

---

## 4. OpenStack-ACI Integration

### 4.1 Integration Model: Cisco ACI Neutron Plugin (Unified Plugin)

The Cisco ACI plugin for OpenStack Neutron replaces the reference ML2/OVS plugin. It maps Neutron abstractions to ACI objects.

**OpenStack to ACI Mapping:**

| OpenStack Object | ACI Object | Notes |
|-----------------|------------|-------|
| Neutron Network | EPG | One EPG per Neutron network |
| Neutron Subnet | BD Subnet | Subnet under the corresponding BD |
| Neutron Router | VRF (or contract) | Inter-BD routing |
| Security Group | Contract + Filters | Security group rules become filter entries |
| Floating IP | Routed via L3Out | SNAT/DNAT on border leaf or external device |
| OpenStack Project (Tenant) | ACI Tenant | 1:1 mapping when using separate tenant mode |

### 4.2 Plugin Deployment Modes

We use **Separate Tenant Mode** (also called "multi-tenant mode"):

- Each OpenStack project maps to a dedicated ACI tenant.
- The five BU projects (`bu1-engineering`, `bu2-finance`, etc.) map to `tn-bu1-engineering`, `tn-bu2-engineering`, etc.
- The plugin auto-creates EPGs, BDs, and contracts in the corresponding ACI tenant when Neutron networks and security groups are created.

### 4.3 OpFlex vs VLAN Mode

**Recommendation: OpFlex mode**

- OpFlex is the native ACI SDN protocol.
- Compute nodes run the `opflex-agent` (also known as `agent-ovs`) which programs Open vSwitch on each hypervisor based on policies downloaded from the APIC leaf switch.
- No VLAN pool management needed on the fabric side; VXLAN is used fabric-internally.
- Endpoint registration is dynamic: when a VM boots, the opflex-agent registers it in the correct EPG on the local leaf.

**OpFlex Deployment Components on Compute Nodes:**

```
+---------------------------------------------------+
|  OpenStack Compute Node                           |
|                                                   |
|  +------------+  +-------------+  +------------+ |
|  | nova-      |  | neutron-    |  | opflex-    | |
|  | compute    |  | opflex-     |  | agent      | |
|  |            |  | agent       |  | (agent-ovs)| |
|  +------------+  +-------------+  +------+-----+ |
|                                          |        |
|  +---------------------------------------+------+ |
|  |          Open vSwitch (OVS)                  | |
|  |  br-int          br-fabric                   | |
|  +------+-----------------------------------+---+ |
|         |                                   |     |
+---------+-----------------------------------+-----+
          |                                   |
     VM tap interfaces               Fabric uplink
                                    (to leaf switch)
```

### 4.4 Neutron Configuration

```ini
# /etc/neutron/neutron.conf (relevant sections)
[DEFAULT]
core_plugin = ml2
service_plugins = cisco_apic_l3,metering

# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = opflex
tenant_network_types = opflex
mechanism_drivers = cisco_apic_ml2

[ml2_cisco_apic]
apic_hosts = 10.100.1.10,10.100.1.11,10.100.1.12
apic_username = admin
apic_password = <vault-encrypted>
apic_name_mapping = use_name
apic_provision_infra = True
apic_provision_hostlinks = True
```

### 4.5 OpenStack Keystone Project to ACI Tenant Mapping

```
Keystone Project          ACI Tenant               VRF
-----------------------------------------------------------------
bu1-engineering           tn-bu1-engineering        vrf-bu1
bu2-finance               tn-bu2-finance            vrf-bu2
bu3-marketing             tn-bu3-marketing          vrf-bu3
bu4-operations            tn-bu4-operations         vrf-bu4
bu5-research              tn-bu5-research           vrf-bu5
admin (control plane)     tn-infra-openstack        vrf-openstack-mgmt
shared-services           tn-shared-services        vrf-shared
dmz                       tn-dmz                    vrf-dmz
```

The pre-existing ACI tenants are linked to OpenStack projects via the plugin's tenant mapping configuration. The plugin manages EPG/BD lifecycle within each tenant.

---

## 5. Service Graph Design

### 5.1 Palo Alto VM-Series (North-South Firewall)

#### 5.1.1 Deployment Model

- Two Palo Alto VM-Series instances deployed as virtual appliances on dedicated service nodes.
- Active/Passive HA pair.
- Connected to the service leaf pair via two interfaces each: outside (untrust) and inside (trust).
- Managed via Panorama for centralized policy and logging.

#### 5.1.2 ACI Service Graph for Palo Alto

```
ACI Service Graph: sg-paloalto-fw
  Type:       L4-L7 Service Graph Template
  Graph Mode: Routed (go-to mode)
  PBR:        Yes (Policy-Based Redirect)
  Device:     Palo Alto VM-Series HA pair

  L4-L7 Device Cluster: dc-paloalto
    Device Type:    Virtual (VM)
    Service Type:   Firewall
    Context Aware:  Single Context (one PAN instance per graph)
    Devices:
      - pan-fw-1 (active):  outside=eth1/1, inside=eth1/2
      - pan-fw-2 (passive): outside=eth1/1, inside=eth1/2
    Concrete Interfaces:
      - consumer-side (outside): VLAN on service leaf (encap auto)
      - provider-side (inside):  VLAN on service leaf (encap auto)

  Function Node: N1-firewall
    Function Type: GoTo
    Redirect:      Enabled
    Connectors:
      - consumer: bound to bd-dmz-outside
      - provider: bound to bd-dmz-inside

  Device Selection Policy:
    Contract:       ctr-internet-to-dmz
    Graph Template: sg-paloalto-fw
    Cluster:        dc-paloalto
```

#### 5.1.3 PBR (Policy-Based Redirect) for Palo Alto

```
PBR Policy: pbr-paloalto-outside
  IP:  198.51.100.10 (PAN outside interface VIP)
  MAC: auto-resolved
  Health Group: hg-paloalto
    Tracking: ICMP or TCP 443 to PAN mgmt

PBR Policy: pbr-paloalto-inside
  IP:  10.250.0.10 (PAN inside interface VIP)
  MAC: auto-resolved
  Health Group: hg-paloalto
```

ACI redirects traffic matching the contract `ctr-internet-to-dmz` to the Palo Alto firewall using PBR. The PBR health group monitors firewall availability and will bypass or drop if the firewall is down (configurable: redirect-only or permit-if-down).

### 5.2 F5 BIG-IP VE (Load Balancer)

#### 5.2.1 Load Balancer Selection: F5 BIG-IP VE

**F5 BIG-IP VE vs. HAProxy (Octavia) Comparison:**

| Criterion | F5 BIG-IP VE | HAProxy (Octavia) |
|-----------|-------------|-------------------|
| ACI Service Graph Integration | Native (F5 ACI ServiceCenter) | Not supported as L4-L7 device |
| Advanced L7 Features | Full iRules, WAF, SSL offload | Basic L7, limited WAF |
| Multi-tenant Isolation | Route domains | Amphora per tenant (resource heavy) |
| Throughput | Licensed, predictable | Amphora VM limited |
| HA | Active/Standby device cluster | Amphora HA (active-standby) |
| ACI PBR Support | Yes, native | No |
| Operational Complexity | Higher (F5 expertise needed) | Lower (OpenStack native) |

**Decision: F5 BIG-IP VE** for production web-tier load balancing, integrated via ACI Service Graph with PBR. This provides full service insertion visibility in APIC and deterministic traffic steering. Octavia with HAProxy remains available as a self-service option for developers doing internal non-production load balancing within their BU tenant.

#### 5.2.2 ACI Service Graph for F5

```
ACI Service Graph: sg-f5-lb
  Type:       L4-L7 Service Graph Template
  Graph Mode: Routed (go-through for one-arm) or Two-Arm
  PBR:        Yes
  Device:     F5 BIG-IP VE HA pair

  L4-L7 Device Cluster: dc-f5-bigip
    Device Type:    Virtual (VM)
    Service Type:   ADC (Load Balancer)
    Context Aware:  Multi-Context (route domains per tenant)
    Devices:
      - f5-bigip-1 (active):  external=1.1, internal=1.2
      - f5-bigip-2 (standby): external=1.1, internal=1.2
    Concrete Interfaces:
      - consumer-side (external): service leaf VLAN
      - provider-side (internal): service leaf VLAN

  Function Node: N1-loadbalancer
    Function Type: GoTo
    Redirect:      Enabled
    Connectors:
      - consumer: bound to bd-dmz-inside
      - provider: bound to bd-buN-web (per-BU device selection)

  Device Selection Policy (per BU):
    Contract:       ctr-dmz-to-web
    Graph Template: sg-f5-lb
    Cluster:        dc-f5-bigip
    Custom:         Route domain per BU tenant on F5
```

#### 5.2.3 F5 BIG-IP VE Configuration Highlights

```
# F5 Route Domains (one per BU for isolation)
Route Domain 1: BU1-Engineering  (VLAN: internal-bu1)
Route Domain 2: BU2-Finance      (VLAN: internal-bu2)
Route Domain 3: BU3-Marketing    (VLAN: internal-bu3)
Route Domain 4: BU4-Operations   (VLAN: internal-bu4)
Route Domain 5: BU5-Research     (VLAN: internal-bu5)
Route Domain 0: DMZ-External     (VLAN: external-dmz)

# Virtual Server (per BU)
ltm virtual /BU1/vs-bu1-web-https {
    destination 10.250.0.101:443%0
    ip-protocol tcp
    pool /BU1/pool-bu1-web
    profiles {
        /Common/tcp { }
        /Common/http { }
        /Common/clientssl { context clientside }
        /Common/serverssl { context serverside }
    }
    source-address-translation { type automap }
    translate-address enabled
}

ltm pool /BU1/pool-bu1-web {
    members {
        10.1.10.10%1:443 { address 10.1.10.10%1 }
        10.1.10.11%1:443 { address 10.1.10.11%1 }
        10.1.10.12%1:443 { address 10.1.10.12%1 }
    }
    monitor /Common/https
}
```

### 5.3 Chained Service Graph (Firewall + Load Balancer)

For traffic that must traverse both the Palo Alto firewall and the F5 load balancer, ACI supports service graph chaining.

```
ACI Service Graph: sg-fw-then-lb
  Nodes:
    N1: Firewall (Palo Alto) - GoTo, PBR
    N2: Load Balancer (F5)   - GoTo, PBR

  Traffic Flow:
    ext-epg-internet --> [N1: PAN FW] --> [N2: F5 LB] --> epg-buN-web

  Contract: ctr-internet-to-bu-web
    Subject: subj-https
    Filter:  flt-allow-https
    Service Graph: sg-fw-then-lb
    Consumer: ext-epg-internet
    Provider: epg-buN-web
```

```
+----------------+     +-----------+     +---------+     +-------------+
|   Internet     |     | Palo Alto |     | F5      |     | BU Web      |
|  (ext-epg)     +---->+ VM-Series +---->+ BIG-IP  +---->+ Servers     |
|  Consumer      |     | N1 (FW)   |     | N2 (LB) |     | Provider    |
+----------------+     +-----------+     +---------+     +-------------+

  PBR redirect          PBR redirect      PBR redirect
  to PAN outside        to F5 external    to real server
  198.51.100.10         10.250.0.100      10.X.10.0/24
```

---

## 6. Traffic Flow Analysis

### 6.1 North-South: Internet to BU1 Web Application

```
Step  Location             Action
----  -------------------  ------------------------------------------------
1     Internet             Client sends HTTPS to public VIP 203.0.113.50
2     Border Leaf          L3Out receives, matches ext-epg-internet
3     ACI Fabric           Contract ctr-internet-to-bu-web matched
                           PBR redirects to Palo Alto (node N1)
4     Service Leaf         Packet delivered to PAN outside interface
                           VLAN encap to 198.51.100.10
5     Palo Alto FW         Inspects packet: zone untrust -> trust
                           Security policy: allow HTTPS
                           NAT: destination NAT to F5 VIP 10.250.0.101
                           Forwards to inside interface
6     Service Leaf         ACI receives from PAN inside interface
                           PBR redirects to F5 (node N2)
7     F5 BIG-IP VE         Receives on external VIP 10.250.0.101:443
                           SSL offload / re-encrypt
                           Selects pool member: 10.1.10.10:443
                           SNATs source to F5 internal self-IP
8     ACI Fabric           Packet routed from vrf-dmz into vrf-bu1
                           (inter-VRF via contract)
9     Compute Leaf         Delivered to epg-bu1-web, endpoint 10.1.10.10
10    Compute Node         opflex-agent delivers to VM tap interface
```

### 6.2 East-West: BU1 App to Shared DNS

```
Step  Location             Action
----  -------------------  ------------------------------------------------
1     Compute Node         BU1 app VM sends DNS query to 10.200.0.10
2     OVS (opflex-agent)   Packet classified to epg-bu1-app
3     Compute Leaf         ACI fabric lookup: dest 10.200.0.10 is in
                           vrf-shared (route leaked via ctr-shared-services)
4     ACI Fabric           Inter-VRF routing: vrf-bu1 -> vrf-shared
                           Contract ctr-shared-services permits UDP 53
5     Destination Leaf     Delivered to epg-dns endpoint 10.200.0.10
6     Compute Node         DNS server responds, reverse path same
```

### 6.3 East-West: BU1 Web to BU1 App (Intra-Tenant)

```
Step  Location             Action
----  -------------------  ------------------------------------------------
1     Compute Node         BU1 web VM sends request to 10.1.20.15:8080
2     OVS (opflex-agent)   Packet classified to epg-bu1-web
3     Compute Leaf         Destination in same VRF (vrf-bu1)
                           Contract ctr-bu1-web-to-app evaluated:
                           epg-bu1-web (consumer) -> epg-bu1-app (provider)
                           Filter flt-allow-app permits TCP 8080
4     ACI Fabric           Routed within vrf-bu1 (VXLAN fabric internal)
5     Destination Leaf     Delivered to epg-bu1-app endpoint 10.1.20.15
```

### 6.4 Denied: BU1 to BU2 (Inter-Tenant Isolation)

```
Step  Location             Action
----  -------------------  ------------------------------------------------
1     Compute Node         BU1 VM attempts to reach 10.2.20.10 (BU2 app)
2     Compute Leaf         Destination 10.2.20.10 in vrf-bu2
                           No contract exists between vrf-bu1 and vrf-bu2
                           No route leaked for 10.2.0.0/16 into vrf-bu1
3     Result               Packet dropped (implicit deny)
                           ACI logs: contract miss / route not found
```

---

## 7. High Availability Design

### 7.1 Fabric HA

| Component | HA Mechanism | RPO/RTO |
|-----------|-------------|---------|
| APIC Cluster | 3-node cluster, quorum-based | RPO=0, RTO<60s |
| Spine Switches | N+1 redundancy (4 spines, can lose 1) | RPO=0, RTO<1s (ECMP reconvergence) |
| Leaf Pairs | vPC pairs, dual-homed hosts | RPO=0, RTO<3s |
| Border Leafs | vPC pair, dual L3Out peering | RPO=0, RTO<5s (BGP reconvergence) |

### 7.2 Service Appliance HA

| Appliance | HA Mechanism | Failover |
|-----------|-------------|----------|
| Palo Alto VM-Series | Active/Passive, heartbeat link | <10s, ACI PBR health group detects and redirects to standby |
| F5 BIG-IP VE | Active/Standby, config-sync | <10s, floating self-IPs migrate, ACI PBR follows |

### 7.3 OpenStack HA

| Component | HA Mechanism |
|-----------|-------------|
| Keystone, Nova API, Neutron API, Glance, Horizon | 3-node active/active behind HAProxy (internal) |
| RabbitMQ | 3-node cluster, mirrored queues |
| MariaDB/Galera | 3-node Galera cluster |
| Neutron agents (DHCP, L3) | Agent HA, multiple DHCP agents per network |

---

## 8. Automation Approach

### 8.1 Automation Stack

```
+-----------------------------------------------------------+
|                   GitLab CI/CD Pipeline                    |
+-----------------------------------------------------------+
         |              |              |              |
    +----+----+   +-----+-----+  +----+----+  +-----+-----+
    | Terraform|   | Ansible   |  | Python  |  | F5 AS3   |
    | (ACI    |   | (OpenStack|  | (APIC   |  | + PAN    |
    |  provider)|  |  deploy)  |  |  SDK)   |  |  Ansible |
    +----+----+   +-----+-----+  +----+----+  +-----+-----+
         |              |              |              |
    +----+--------------+--------------+--------------+----+
    |                    APIC REST API                      |
    |                    Keystone API                       |
    |                    Panorama API                       |
    |                    F5 iControl REST                   |
    +------------------------------------------------------+
```

### 8.2 Terraform: ACI Tenant Infrastructure

Terraform with the `CiscoDevNet/aci` provider manages the declarative ACI object model.

```hcl
# terraform/modules/bu-tenant/main.tf

variable "bu_name" {}
variable "bu_id" {}
variable "web_subnet" {}
variable "app_subnet" {}
variable "db_subnet" {}

resource "aci_tenant" "bu" {
  name        = "tn-${var.bu_name}"
  description = "Tenant for ${var.bu_name}"
}

resource "aci_vrf" "bu" {
  tenant_dn              = aci_tenant.bu.id
  name                   = "vrf-bu${var.bu_id}"
  pc_enf_pref            = "enforced"
  pc_enf_dir             = "ingress"
}

resource "aci_bridge_domain" "web" {
  tenant_dn          = aci_tenant.bu.id
  name               = "bd-bu${var.bu_id}-web"
  relation_fv_rs_ctx = aci_vrf.bu.id
  arp_flood          = "no"
  unicast_route      = "yes"
}

resource "aci_subnet" "web" {
  parent_dn = aci_bridge_domain.web.id
  ip        = var.web_subnet
  scope     = ["private", "shared"]
}

resource "aci_bridge_domain" "app" {
  tenant_dn          = aci_tenant.bu.id
  name               = "bd-bu${var.bu_id}-app"
  relation_fv_rs_ctx = aci_vrf.bu.id
  arp_flood          = "no"
  unicast_route      = "yes"
}

resource "aci_subnet" "app" {
  parent_dn = aci_bridge_domain.app.id
  ip        = var.app_subnet
  scope     = ["private"]
}

resource "aci_bridge_domain" "db" {
  tenant_dn          = aci_tenant.bu.id
  name               = "bd-bu${var.bu_id}-db"
  relation_fv_rs_ctx = aci_vrf.bu.id
  arp_flood          = "no"
  unicast_route      = "yes"
}

resource "aci_subnet" "db" {
  parent_dn = aci_bridge_domain.db.id
  ip        = var.db_subnet
  scope     = ["private"]
}

resource "aci_application_profile" "bu" {
  tenant_dn = aci_tenant.bu.id
  name      = "ap-bu${var.bu_id}"
}

resource "aci_application_epg" "web" {
  application_profile_dn = aci_application_profile.bu.id
  name                   = "epg-bu${var.bu_id}-web"
  relation_fv_rs_bd      = aci_bridge_domain.web.id
}

resource "aci_application_epg" "app" {
  application_profile_dn = aci_application_profile.bu.id
  name                   = "epg-bu${var.bu_id}-app"
  relation_fv_rs_bd      = aci_bridge_domain.app.id
}

resource "aci_application_epg" "db" {
  application_profile_dn = aci_application_profile.bu.id
  name                   = "epg-bu${var.bu_id}-db"
  relation_fv_rs_bd      = aci_bridge_domain.db.id
}

# Contracts
resource "aci_contract" "web_to_app" {
  tenant_dn = aci_tenant.bu.id
  name      = "ctr-bu${var.bu_id}-web-to-app"
  scope     = "tenant"
}

resource "aci_contract_subject" "web_to_app" {
  contract_dn                  = aci_contract.web_to_app.id
  name                         = "subj-app-access"
  relation_vz_rs_subj_filt_att = [data.aci_filter.allow_app.id]
}

resource "aci_contract" "app_to_db" {
  tenant_dn = aci_tenant.bu.id
  name      = "ctr-bu${var.bu_id}-app-to-db"
  scope     = "tenant"
}

resource "aci_contract_subject" "app_to_db" {
  contract_dn                  = aci_contract.app_to_db.id
  name                         = "subj-db-access"
  relation_vz_rs_subj_filt_att = [data.aci_filter.allow_db.id]
}

# EPG contract bindings
resource "aci_epg_to_contract" "web_consumes_web_to_app" {
  application_epg_dn = aci_application_epg.web.id
  contract_dn        = aci_contract.web_to_app.id
  contract_type      = "consumer"
}

resource "aci_epg_to_contract" "app_provides_web_to_app" {
  application_epg_dn = aci_application_epg.app.id
  contract_dn        = aci_contract.web_to_app.id
  contract_type      = "provider"
}

resource "aci_epg_to_contract" "app_consumes_app_to_db" {
  application_epg_dn = aci_application_epg.app.id
  contract_dn        = aci_contract.app_to_db.id
  contract_type      = "consumer"
}

resource "aci_epg_to_contract" "db_provides_app_to_db" {
  application_epg_dn = aci_application_epg.db.id
  contract_dn        = aci_contract.app_to_db.id
  contract_type      = "provider"
}
```

```hcl
# terraform/environments/prod/main.tf

module "bu1" {
  source     = "../../modules/bu-tenant"
  bu_name    = "bu1-engineering"
  bu_id      = "1"
  web_subnet = "10.1.10.1/24"
  app_subnet = "10.1.20.1/24"
  db_subnet  = "10.1.30.1/24"
}

module "bu2" {
  source     = "../../modules/bu-tenant"
  bu_name    = "bu2-finance"
  bu_id      = "2"
  web_subnet = "10.2.10.1/24"
  app_subnet = "10.2.20.1/24"
  db_subnet  = "10.2.30.1/24"
}

module "bu3" {
  source     = "../../modules/bu-tenant"
  bu_name    = "bu3-marketing"
  bu_id      = "3"
  web_subnet = "10.3.10.1/24"
  app_subnet = "10.3.20.1/24"
  db_subnet  = "10.3.30.1/24"
}

module "bu4" {
  source     = "../../modules/bu-tenant"
  bu_name    = "bu4-operations"
  bu_id      = "4"
  web_subnet = "10.4.10.1/24"
  app_subnet = "10.4.20.1/24"
  db_subnet  = "10.4.30.1/24"
}

module "bu5" {
  source     = "../../modules/bu-tenant"
  bu_name    = "bu5-research"
  bu_id      = "5"
  web_subnet = "10.5.10.1/24"
  app_subnet = "10.5.20.1/24"
  db_subnet  = "10.5.30.1/24"
}
```

### 8.3 Terraform: Service Graph and PBR

```hcl
# terraform/modules/service-graph/paloalto.tf

resource "aci_l4_l7_service_graph_template" "fw" {
  tenant_dn = data.aci_tenant.dmz.id
  name      = "sg-paloalto-fw"
}

resource "aci_function_node" "fw_node" {
  l4_l7_service_graph_template_dn = aci_l4_l7_service_graph_template.fw.id
  name                             = "N1-firewall"
  func_type                        = "GoTo"
  managed                          = "no"
}

resource "aci_l4_l7_device" "paloalto" {
  tenant_dn       = data.aci_tenant.dmz.id
  name            = "dc-paloalto"
  devtype         = "VIRTUAL"
  svc_type        = "FW"
  context_aware   = "single-Context"
}

resource "aci_service_redirect_policy" "pan_outside" {
  tenant_dn   = data.aci_tenant.dmz.id
  name        = "pbr-paloalto-outside"
  dest_type   = "L3"
  hashing_algorithm = "sip-dip-prototype"
}

resource "aci_destination_of_redirected_traffic" "pan_outside_dest" {
  service_redirect_policy_dn = aci_service_redirect_policy.pan_outside.id
  ip                         = "198.51.100.10"
  mac                        = "00:50:56:XX:XX:01"
}

resource "aci_service_redirect_policy" "pan_inside" {
  tenant_dn   = data.aci_tenant.dmz.id
  name        = "pbr-paloalto-inside"
  dest_type   = "L3"
  hashing_algorithm = "sip-dip-prototype"
}

resource "aci_destination_of_redirected_traffic" "pan_inside_dest" {
  service_redirect_policy_dn = aci_service_redirect_policy.pan_inside.id
  ip                         = "10.250.0.10"
  mac                        = "00:50:56:XX:XX:02"
}
```

### 8.4 Ansible: OpenStack Deployment

```yaml
# ansible/playbooks/deploy-openstack.yml
---
- name: Deploy OpenStack Control Plane
  hosts: openstack_controllers
  become: yes
  roles:
    - role: openstack-keystone
    - role: openstack-glance
    - role: openstack-nova-controller
    - role: openstack-neutron-server
      vars:
        neutron_plugin: cisco_apic_ml2
        apic_hosts: "{{ apic_cluster_ips }}"
    - role: openstack-cinder
    - role: openstack-horizon
    - role: openstack-heat

- name: Deploy OpenStack Compute Nodes
  hosts: openstack_compute
  become: yes
  roles:
    - role: openstack-nova-compute
    - role: cisco-opflex-agent
      vars:
        opflex_peer_ip: "{{ leaf_switch_ip }}"
        opflex_encap_type: vxlan
    - role: ovs-configuration
      vars:
        bridges:
          - name: br-int
            type: integration
          - name: br-fabric
            type: fabric
            uplink: "{{ fabric_interface }}"

- name: Configure OpenStack Projects and Quotas
  hosts: openstack_controllers[0]
  tasks:
    - name: Create BU projects
      openstack.cloud.project:
        name: "{{ item.name }}"
        domain: default
        description: "{{ item.description }}"
      loop:
        - { name: bu1-engineering, description: "Engineering BU" }
        - { name: bu2-finance,     description: "Finance BU" }
        - { name: bu3-marketing,   description: "Marketing BU" }
        - { name: bu4-operations,  description: "Operations BU" }
        - { name: bu5-research,    description: "Research BU" }

    - name: Set per-BU quotas
      openstack.cloud.quota:
        name: "{{ item }}"
        instances: 50
        cores: 200
        ram: 524288
        networks: 10
        subnets: 20
        routers: 5
        floating_ips: 20
        security_groups: 50
      loop:
        - bu1-engineering
        - bu2-finance
        - bu3-marketing
        - bu4-operations
        - bu5-research
```

### 8.5 Ansible: Palo Alto Configuration

```yaml
# ansible/playbooks/configure-paloalto.yml
---
- name: Configure Palo Alto VM-Series
  hosts: panorama
  connection: local
  collections:
    - paloaltonetworks.panos

  tasks:
    - name: Configure zones
      panos_zone:
        provider: "{{ provider }}"
        template: "openstack-fw"
        zone: "{{ item.name }}"
        mode: layer3
      loop:
        - { name: untrust }
        - { name: trust }
        - { name: dmz }

    - name: Configure interfaces
      panos_interface:
        provider: "{{ provider }}"
        template: "openstack-fw"
        if_name: "ethernet1/1"
        zone_name: untrust
        ip: ["198.51.100.10/28"]
        comment: "Outside - ACI bd-dmz-outside"

    - name: Configure inside interface
      panos_interface:
        provider: "{{ provider }}"
        template: "openstack-fw"
        if_name: "ethernet1/2"
        zone_name: trust
        ip: ["10.250.0.10/24"]
        comment: "Inside - ACI bd-dmz-inside"

    - name: Create security rules
      panos_security_rule:
        provider: "{{ provider }}"
        device_group: "openstack-dg"
        rule_name: "allow-https-inbound"
        source_zone: ["untrust"]
        destination_zone: ["trust"]
        application: ["ssl", "web-browsing"]
        action: "allow"
        log_end: true
        log_setting: "default-logging"

    - name: Create NAT rules
      panos_nat_rule:
        provider: "{{ provider }}"
        device_group: "openstack-dg"
        rule_name: "dnat-web-to-f5"
        source_zone: ["untrust"]
        destination_zone: ["untrust"]
        destination_ip: ["203.0.113.50"]
        dnat_address: "10.250.0.101"
        snat_type: "dynamic-ip-and-port"
        snat_interface: "ethernet1/2"
```

### 8.6 F5 AS3 Declaration (Declarative Automation)

```json
{
  "class": "AS3",
  "action": "deploy",
  "persist": true,
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.40.0",
    "id": "openstack-aci-lb",
    "BU1_Engineering": {
      "class": "Tenant",
      "Web_App": {
        "class": "Application",
        "template": "https",
        "serviceMain": {
          "class": "Service_HTTPS",
          "virtualAddresses": ["10.250.0.101"],
          "virtualPort": 443,
          "pool": "pool_bu1_web",
          "clientTLS": "tlsClient_bu1",
          "serverTLS": "tlsServer_bu1",
          "snat": "auto",
          "persistenceMethods": ["cookie"],
          "profileHTTP": {
            "use": "http_profile_bu1"
          }
        },
        "pool_bu1_web": {
          "class": "Pool",
          "monitors": ["https"],
          "members": [
            {
              "servicePort": 443,
              "addressDiscovery": "static",
              "serverAddresses": [
                "10.1.10.10",
                "10.1.10.11",
                "10.1.10.12"
              ]
            }
          ]
        },
        "tlsClient_bu1": {
          "class": "TLS_Client",
          "sendSNI": "bu1.internal.example.com"
        },
        "tlsServer_bu1": {
          "class": "TLS_Server",
          "certificates": [
            {
              "certificate": "webcert_bu1"
            }
          ]
        },
        "webcert_bu1": {
          "class": "Certificate",
          "certificate": {"bigip": "/Common/bu1.internal.example.com.crt"},
          "privateKey": {"bigip": "/Common/bu1.internal.example.com.key"}
        },
        "http_profile_bu1": {
          "class": "HTTP_Profile",
          "xForwardedFor": true
        }
      }
    }
  }
}
```

### 8.7 CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply-aci
  - deploy-openstack
  - configure-services
  - test

validate-terraform:
  stage: validate
  script:
    - cd terraform/environments/prod
    - terraform init
    - terraform validate
    - terraform fmt -check

validate-ansible:
  stage: validate
  script:
    - ansible-lint ansible/playbooks/*.yml

plan-aci:
  stage: plan
  script:
    - cd terraform/environments/prod
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - terraform/environments/prod/tfplan

apply-aci:
  stage: apply-aci
  when: manual
  script:
    - cd terraform/environments/prod
    - terraform apply tfplan
  only:
    - main

deploy-openstack:
  stage: deploy-openstack
  when: manual
  script:
    - ansible-playbook -i inventory/prod ansible/playbooks/deploy-openstack.yml
  only:
    - main

configure-paloalto:
  stage: configure-services
  script:
    - ansible-playbook -i inventory/prod ansible/playbooks/configure-paloalto.yml

configure-f5:
  stage: configure-services
  script:
    - >
      curl -sk -u admin:${F5_PASSWORD}
      -H "Content-Type: application/json"
      -d @f5/as3-declaration.json
      https://${F5_MGMT_IP}/mgmt/shared/appsvcs/declare

smoke-test:
  stage: test
  script:
    - python3 tests/smoke_test_connectivity.py
    - python3 tests/smoke_test_isolation.py
    - python3 tests/smoke_test_shared_services.py
```

### 8.8 Day-2 Operations Automation

```python
#!/usr/bin/env python3
"""
day2_onboard_bu.py - Automates onboarding a new BU tenant.
Orchestrates: ACI tenant creation, OpenStack project creation,
shared-services contract attachment, F5 route domain, PAN zone.
"""

import cobra.mit.access
import cobra.mit.session
import cobra.model.fv as fv
import cobra.model.vz as vz
from keystoneauth1 import session as ks_session
from keystoneauth1.identity import v3 as ks_identity
from openstack import connection
import subprocess
import json
import sys

def onboard_bu(bu_name: str, bu_id: int, subnets: dict):
    """
    Full BU onboarding workflow.

    Args:
        bu_name:  e.g., "bu6-analytics"
        bu_id:    e.g., 6
        subnets:  {"web": "10.6.10.0/24", "app": "10.6.20.0/24", "db": "10.6.30.0/24"}
    """

    # Step 1: Terraform apply for ACI objects
    tf_vars = {
        "bu_name": bu_name,
        "bu_id": str(bu_id),
        "web_subnet": subnets["web"].replace(".0/", ".1/"),
        "app_subnet": subnets["app"].replace(".0/", ".1/"),
        "db_subnet": subnets["db"].replace(".0/", ".1/"),
    }

    tf_var_args = " ".join([f'-var="{k}={v}"' for k, v in tf_vars.items()])
    subprocess.run(
        f"terraform apply -auto-approve {tf_var_args}",
        shell=True, cwd="terraform/modules/bu-tenant", check=True
    )

    # Step 2: Create OpenStack project
    auth = ks_identity.Password(
        auth_url="https://keystone.internal:5000/v3",
        username="admin",
        password="<vault>",
        project_name="admin",
        user_domain_name="Default",
        project_domain_name="Default",
    )
    sess = ks_session.Session(auth=auth)
    conn = connection.Connection(session=sess)

    project = conn.identity.create_project(
        name=bu_name,
        domain_id="default",
        description=f"Project for {bu_name}",
    )

    # Step 3: Set quotas
    conn.set_compute_quotas(project.id, instances=50, cores=200, ram=524288)

    # Step 4: Create networks in the new project
    for tier in ["web", "app", "db"]:
        network = conn.network.create_network(
            name=f"{bu_name}-{tier}-net",
            project_id=project.id,
        )
        conn.network.create_subnet(
            name=f"{bu_name}-{tier}-subnet",
            network_id=network.id,
            cidr=subnets[tier],
            ip_version=4,
            project_id=project.id,
        )

    # Step 5: Configure F5 route domain via AS3
    # Step 6: Add PAN security zone if needed

    print(f"BU {bu_name} onboarded successfully.")


if __name__ == "__main__":
    onboard_bu(
        bu_name=sys.argv[1],
        bu_id=int(sys.argv[2]),
        subnets=json.loads(sys.argv[3]),
    )
```

---

## 9. Monitoring and Observability

### 9.1 ACI Fabric Monitoring

| Tool | Purpose | Integration |
|------|---------|-------------|
| APIC GUI/API | Fabric health score, fault dashboard | Native |
| Cisco NAE/Nexus Dashboard Insights | Assurance, compliance, change analysis | APIC integration |
| Prometheus + SNMP Exporter | Leaf/spine interface counters, CPU, memory | SNMP polling from tn-shared-services |
| Grafana | Dashboards for fabric utilization | Prometheus data source |
| Syslog to Splunk/ELK | ACI faults, audit logs, contract drops | APIC syslog export |

### 9.2 Contract Drop Monitoring

ACI contract drops are critical signals. Configure:

```
APIC -> Fabric Policies -> Monitoring -> Common Policy -> Syslog
  - Source: Contract Drop Logs
  - Destination: 10.200.0.50:514 (Splunk/ELK in shared services)
  - Severity: warning
  - Include: src-epg, dst-epg, protocol, src-ip, dst-ip
```

### 9.3 Service Appliance Monitoring

- **Palo Alto**: Panorama for centralized log collection; forward to Splunk via syslog. Monitor threat logs, traffic logs, and system logs.
- **F5 BIG-IP**: Telemetry Streaming to Prometheus/Grafana. Monitor pool member health, virtual server connections, SSL TPS, and throughput.

---

## 10. Security Considerations

### 10.1 Micro-Segmentation Strategy

- **Tier 1 (coarse)**: VRF isolation between BUs. No inter-BU traffic without explicit contract.
- **Tier 2 (medium)**: EPG contracts within each BU enforce tiered access (web->app->db, never web->db).
- **Tier 3 (fine)**: OpenStack security groups (mapped to ACI contracts via the plugin) provide per-VM-group rules.
- **Tier 4 (north-south)**: Palo Alto VM-Series provides L7 inspection, IPS/IDS, URL filtering, and threat prevention for all internet-bound traffic.

### 10.2 Contract Enforcement Matrix

| Source | Destination | Allowed | Enforced By |
|--------|-------------|---------|-------------|
| Internet | DMZ Web VIPs | HTTPS only | PAN FW + ACI contract |
| DMZ Inside | BU Web Tier | HTTP/HTTPS | F5 LB + ACI contract |
| BU Web | BU App | TCP 8080/8443 | ACI contract |
| BU App | BU DB | TCP 3306/5432 | ACI contract |
| Any BU | Shared DNS | UDP/TCP 53 | ACI inter-VRF contract |
| Any BU | Shared LDAP | TCP 389/636 | ACI inter-VRF contract |
| Any BU | Shared Monitoring | TCP 9090/9100 | ACI inter-VRF contract |
| BU1 | BU2 | DENIED | VRF isolation (no contract) |
| BU Web | BU DB (direct) | DENIED | No contract (must go through App) |

### 10.3 Encryption

- All OpenStack API endpoints use TLS 1.2+.
- F5 handles SSL offload with certificates managed via Vault or ACME.
- Palo Alto SSL decryption policy for outbound inspection (optional, privacy-sensitive).
- ACI fabric: MACsec on inter-switch links (optional, for compliance).

---

## 11. IP Address Allocation Summary

| Range | Assignment |
|-------|-----------|
| 10.100.1.0/24 | OpenStack API network |
| 10.100.2.0/24 | OpenStack management network |
| 10.200.0.0/24 | Shared services (DNS, LDAP, monitoring) |
| 10.1.0.0/16 | BU1 Engineering (10.1.10/20/30.0/24) |
| 10.2.0.0/16 | BU2 Finance |
| 10.3.0.0/16 | BU3 Marketing |
| 10.4.0.0/16 | BU4 Operations |
| 10.5.0.0/16 | BU5 Research |
| 10.250.0.0/24 | DMZ inside |
| 198.51.100.0/28 | DMZ outside (public) |
| 203.0.113.0/28 | Public VIPs (NAT) |
| 10.0.0.0/24 | ACI infra TEP pool (fabric internal) |
| 10.99.0.0/24 | APIC out-of-band management |

---

## 12. Deployment Sequence

```
Phase 1: Foundation (Week 1-2)
  1. Rack and cable Nexus 9K spine-leaf fabric
  2. Initialize APIC cluster (3 nodes)
  3. Fabric discovery and firmware upgrade
  4. Configure fabric access policies (interface, switch, VLAN pools)
  5. Configure ACI infra tenant (TEP, VXLAN pools)

Phase 2: ACI Policy (Week 2-3)
  6. Terraform apply: common filters and shared contracts
  7. Terraform apply: tn-shared-services (VRF, BDs, EPGs)
  8. Terraform apply: tn-dmz (VRF, BDs, L3Out)
  9. Terraform apply: BU tenants (x5)
  10. Terraform apply: Service graph templates and PBR policies

Phase 3: OpenStack (Week 3-5)
  11. Deploy OpenStack control plane (Ansible)
  12. Install Cisco ACI Neutron plugin
  13. Configure OpFlex on compute nodes
  14. Validate Neutron network creation maps to ACI EPGs
  15. Create BU projects and quotas

Phase 4: Service Appliances (Week 5-6)
  16. Deploy Palo Alto VM-Series HA pair
  17. Configure PAN via Panorama (Ansible)
  18. Attach PAN to ACI service graph, validate PBR
  19. Deploy F5 BIG-IP VE HA pair
  20. Configure F5 via AS3
  21. Attach F5 to ACI service graph, validate PBR

Phase 5: Validation (Week 6-7)
  22. North-south traffic flow test (Internet -> PAN -> F5 -> BU web)
  23. East-west shared services test (BU -> DNS/LDAP)
  24. Isolation test (BU1 cannot reach BU2)
  25. Failover tests (PAN HA, F5 HA, leaf failure, spine failure)
  26. Performance baseline

Phase 6: Handover (Week 7-8)
  27. Documentation and runbook finalization
  28. Operations team training
  29. Production cutover
```

---

## 13. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| APIC cluster split-brain | Fabric policy inconsistency | 3 APICs on separate leaf pairs; OOB management network |
| Service graph PBR misconfiguration | Traffic black-holed | Health groups with ICMP tracking; "permit" action on failure as fallback |
| OpFlex agent crash on compute | VMs lose network | OpFlex agent systemd watchdog; pre-existing flows remain in OVS |
| Terraform state drift | ACI config mismatch | Scheduled `terraform plan` drift detection in CI; APIC change notifications |
| F5 route domain misconfiguration | Cross-tenant traffic leak | Strict automation-only config; F5 audit logging; periodic compliance scans |
| PAN firmware vulnerability | North-south bypass | Panorama auto-update; IPS signatures on aggressive schedule |

---

## 14. Appendix: Key ACI Configuration Parameters

### 14.1 Fabric Access Policies

```
VLAN Pool:  vlanpool-openstack-dynamic (dynamic allocation, 2000-3999)
Domain:     vmm-openstack (VMM domain, OpenStack type)
AEP:        aep-openstack-compute (attachable entity profile)
Int Policy Group: ipg-compute-vpc (vPC, LACP active, CDP on, LLDP on)
Leaf Profile: lp-compute-pair-A (leafs 101-102)
              lp-compute-pair-B (leafs 103-104)
              lp-compute-pair-C (leafs 105-106)
              lp-service-pair   (leafs 107-108)
              lp-border-pair    (leafs 109-110)
```

### 14.2 L3Out Configuration (Border Leafs)

```
L3Out: l3out-internet
  Tenant:     tn-dmz
  VRF:        vrf-dmz
  L3 Domain:  l3dom-external
  Node Profile: np-border-leafs (leafs 109-110)
    Interface Profile:
      - Routed Sub-Interface on port-channel (vPC)
      - VLAN: 100 (external handoff)
    BGP Peer:
      - Peer IP: 203.0.113.1 (ISP router)
      - Remote AS: 65100
      - Local AS: 65000
      - Timers: keepalive 10, hold 30
      - Route Maps: rm-inbound (accept default + specifics)
                    rm-outbound (advertise public VIPs only)
  External EPG: ext-epg-internet
    Subnet: 0.0.0.0/0 (external-subnets, shared-security, shared-rtctrl)
```
