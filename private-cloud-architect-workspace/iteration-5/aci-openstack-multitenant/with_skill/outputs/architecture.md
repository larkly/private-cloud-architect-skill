# ACI + OpenStack Multi-Tenant Private Cloud Architecture

## 1. Environment Overview

| Component | Details |
|---|---|
| Fabric | Cisco ACI spine-leaf (Nexus 9000 series) |
| APIC Cluster | 3x APIC controllers (HA cluster) |
| Cloud Platform | OpenStack (Kolla-Ansible deployment) |
| Tenants | 5 internal business units (BU1-BU5) + shared services |
| Firewall | Palo Alto VM-Series (north-south, perimeter) |
| Load Balancer | F5 BIG-IP VE (recommended) or HAProxy via Octavia |
| Automation | Ansible + OpenTofu + CI/CD pipeline |

---

## 2. ACI Object Model Design

### 2.1 Tenant Structure

The ACI tenant model maps directly to organizational boundaries. Six tenants total: one per business unit and one for shared infrastructure.

```
ACI Fabric
├── Tenant: common                    (ACI built-in, used for shared contracts)
├── Tenant: tn-shared-services        (DNS, LDAP, monitoring, Panorama, F5 mgmt)
├── Tenant: tn-bu1                    (Business Unit 1)
├── Tenant: tn-bu2                    (Business Unit 2)
├── Tenant: tn-bu3                    (Business Unit 3)
├── Tenant: tn-bu4                    (Business Unit 4)
├── Tenant: tn-bu5                    (Business Unit 5)
├── Tenant: tn-dmz                    (DMZ workloads, internet-facing services)
└── Tenant: tn-mgmt                   (Infrastructure management - APIC, Panorama, vCenter/OpenStack controllers)
```

### 2.2 VRFs (Routing Domains)

Each tenant gets its own VRF for Layer 3 isolation. This allows overlapping IP space between BUs if needed, though we recommend a unified IPAM (NetBox) to avoid operational confusion.

| Tenant | VRF | Purpose | Enforced | Preferred Group |
|---|---|---|---|---|
| tn-shared-services | vrf-shared | Shared infra services | Yes | Disabled |
| tn-bu1 | vrf-bu1 | BU1 workloads | Yes | Disabled |
| tn-bu2 | vrf-bu2 | BU2 workloads | Yes | Disabled |
| tn-bu3 | vrf-bu3 | BU3 workloads | Yes | Disabled |
| tn-bu4 | vrf-bu4 | BU4 workloads | Yes | Disabled |
| tn-bu5 | vrf-bu5 | BU5 workloads | Yes | Disabled |
| tn-dmz | vrf-dmz | Internet-facing services | Yes | Disabled |
| tn-mgmt | vrf-mgmt | Management network | Yes | Disabled |

**VRF Policy Enforcement**: Set to "enforced" on all VRFs. This ensures the ACI whitelist model is active -- traffic between EPGs within the same VRF requires an explicit Contract. This is the foundation of the zero-trust posture.

**Inter-VRF Routing (Route Leaking)**: Shared services (DNS, LDAP, monitoring) in vrf-shared must be reachable from all BU VRFs. This is achieved via inter-tenant Contract export/import (vzAny or explicit contract consumption across tenants), not VRF route leaking, to maintain policy granularity.

### 2.3 Bridge Domains and Subnets

Each VRF contains Bridge Domains sized to application tiers. ACI acts as the distributed anycast gateway for all subnets.

#### tn-shared-services

| Bridge Domain | Subnet | Gateway | Purpose |
|---|---|---|---|
| bd-dns | 10.100.1.0/24 | 10.100.1.1 | DNS servers (PowerDNS/Designate) |
| bd-ldap | 10.100.2.0/24 | 10.100.2.1 | LDAP/FreeIPA servers |
| bd-monitoring | 10.100.3.0/24 | 10.100.3.1 | Prometheus, Grafana, Loki |
| bd-ntp | 10.100.4.0/24 | 10.100.4.1 | NTP, syslog aggregation |

BD Settings: ARP flooding disabled, unicast routing enabled, L2 unknown unicast set to hardware proxy. Subnet scope set to "public" (advertised externally) and "shared" (exported to other VRFs for inter-tenant consumption).

#### tn-bu1 through tn-bu5 (template per BU, BU1 shown)

| Bridge Domain | Subnet | Gateway | Purpose |
|---|---|---|---|
| bd-bu1-web | 10.1.10.0/24 | 10.1.10.1 | Web tier VMs |
| bd-bu1-app | 10.1.20.0/24 | 10.1.20.1 | Application tier VMs |
| bd-bu1-db | 10.1.30.0/24 | 10.1.30.1 | Database tier VMs |
| bd-bu1-internal | 10.1.40.0/24 | 10.1.40.1 | Internal services, batch jobs |

Each BU uses a unique second octet: BU1=10.1.x, BU2=10.2.x, BU3=10.3.x, BU4=10.4.x, BU5=10.5.x.

#### tn-dmz

| Bridge Domain | Subnet | Gateway | Purpose |
|---|---|---|---|
| bd-dmz-frontend | 172.16.1.0/24 | 172.16.1.1 | Internet-facing VIPs (post-firewall) |
| bd-dmz-backend | 172.16.2.0/24 | 172.16.2.1 | DMZ application servers |

#### tn-mgmt

| Bridge Domain | Subnet | Gateway | Purpose |
|---|---|---|---|
| bd-mgmt-infra | 192.168.1.0/24 | 192.168.1.1 | APIC, Panorama, F5 mgmt interfaces |
| bd-mgmt-openstack | 192.168.2.0/24 | 192.168.2.1 | OpenStack control plane (Keystone, Nova API, Neutron, etc.) |
| bd-mgmt-oob | 192.168.3.0/24 | 192.168.3.1 | Out-of-band (CIMC/iLO/iDRAC) |

### 2.4 Application Profiles and EPGs

Each tenant contains Application Profiles that group related EPGs. EPGs are the fundamental policy enforcement boundary in ACI.

#### tn-shared-services Application Profile: ap-infra-services

| EPG | Bridge Domain | VMM Domain | Description |
|---|---|---|---|
| epg-dns | bd-dns | openstack-vmm | DNS servers |
| epg-ldap | bd-ldap | openstack-vmm | LDAP/FreeIPA |
| epg-monitoring | bd-monitoring | openstack-vmm | Prometheus, Grafana, Loki, Alertmanager |
| epg-ntp | bd-ntp | openstack-vmm | NTP, syslog |

#### tn-bu1 Application Profile: ap-bu1-workloads (template for all BUs)

| EPG | Bridge Domain | VMM Domain | Description |
|---|---|---|---|
| epg-bu1-web | bd-bu1-web | openstack-vmm | Web frontends |
| epg-bu1-app | bd-bu1-app | openstack-vmm | Application logic |
| epg-bu1-db | bd-bu1-db | openstack-vmm | Databases |
| epg-bu1-internal | bd-bu1-internal | openstack-vmm | Internal services |

#### tn-dmz Application Profile: ap-dmz

| EPG | Bridge Domain | VMM Domain | Description |
|---|---|---|---|
| epg-dmz-frontend | bd-dmz-frontend | openstack-vmm | Post-FW internet-facing |
| epg-dmz-backend | bd-dmz-backend | openstack-vmm | DMZ app servers |

### 2.5 Contracts (Inter-EPG Policy)

Contracts define permitted traffic flows between EPGs. ACI uses a whitelist model: no contract means no communication.

#### Filters (referenced by Contracts)

| Filter Name | Entries | Description |
|---|---|---|
| flt-dns | TCP/UDP 53 | DNS queries |
| flt-ldap | TCP 389, 636 | LDAP and LDAPS |
| flt-monitoring | TCP 9090, 9093, 9100, 3000, 3100 | Prometheus, Alertmanager, node_exporter, Grafana, Loki |
| flt-ntp | UDP 123 | NTP |
| flt-syslog | UDP 514, TCP 514 | Syslog |
| flt-http | TCP 80, 443 | HTTP/HTTPS |
| flt-app-tcp | TCP 8080, 8443 | Application ports |
| flt-db-mysql | TCP 3306 | MySQL/MariaDB |
| flt-db-pgsql | TCP 5432 | PostgreSQL |
| flt-ssh | TCP 22 | SSH management |
| flt-icmp | ICMP | Ping/traceroute |
| flt-any | IP (any) | Permit all (used sparingly) |

#### Intra-Tenant Contracts (BU1 example, replicated per BU)

| Contract | Provider EPG | Consumer EPG | Filter | Direction |
|---|---|---|---|---|
| ctr-bu1-web-to-app | epg-bu1-app | epg-bu1-web | flt-app-tcp | Uni-directional |
| ctr-bu1-app-to-db | epg-bu1-db | epg-bu1-app | flt-db-pgsql | Uni-directional |
| ctr-bu1-internal | epg-bu1-internal | epg-bu1-internal | flt-any | Bi-directional (intra-EPG) |

#### Inter-Tenant Contracts (Shared Services)

These contracts are defined in tn-shared-services and exported for consumption by BU tenants.

| Contract | Provider EPG | Consumer Tenants | Filter | Scope |
|---|---|---|---|---|
| ctr-shared-dns | epg-dns | All BU tenants (vzAny or explicit) | flt-dns | Global |
| ctr-shared-ldap | epg-ldap | All BU tenants | flt-ldap | Global |
| ctr-shared-monitoring | epg-monitoring | All BU tenants | flt-monitoring | Global |
| ctr-shared-ntp | epg-ntp | All BU tenants | flt-ntp, flt-syslog | Global |

**Implementation**: In each BU tenant, the shared service contracts are imported as consumed contract interfaces. The contract scope is set to "global" to permit cross-VRF communication. ACI handles the inter-VRF policy stitching automatically -- no manual route leaking configuration needed.

#### DMZ Contracts

| Contract | Provider EPG | Consumer EPG | Filter | Service Graph |
|---|---|---|---|---|
| ctr-internet-to-dmz | epg-dmz-frontend | l3out-internet (External EPG) | flt-http | sg-palo-alto (FW) |
| ctr-dmz-to-web | epg-bu*-web | epg-dmz-frontend | flt-http | sg-f5-lb (LB) |

### 2.6 L3Outs (External Connectivity)

| L3Out | Tenant | VRF | Protocol | Peer | Purpose |
|---|---|---|---|---|---|
| l3out-internet | tn-dmz | vrf-dmz | BGP | Border routers | Internet connectivity |
| l3out-wan | tn-mgmt | vrf-mgmt | OSPF | WAN routers | Corporate WAN |
| l3out-shared-transit | tn-shared-services | vrf-shared | Static | Core routers | Shared services reachability from campus |

Each L3Out has an associated External EPG (l3extInstP) that classifies external traffic and enables contract-based policy enforcement at the fabric border.

---

## 3. OpenStack-ACI Integration

### 3.1 Integration Architecture

The ACI Neutron ML2 mechanism driver is deployed in "managed" mode. APIC manages the physical infrastructure (spine-leaf fabric, L3Outs), while OpenStack manages tenant-facing constructs (networks, routers, security groups) which are automatically mapped to ACI objects.

```
OpenStack Control Plane
├── Keystone (Identity)
├── Nova (Compute)
├── Neutron + ACI ML2 Plugin
│   ├── APIC API (REST) ──→ APIC Cluster (3x)
│   └── OpFlex Agent ──→ Compute Nodes (OVS + OpFlex)
├── Cinder (Block Storage) ──→ Ceph
├── Glance (Images) ──→ Ceph
├── Octavia (LBaaS)
├── Heat (Orchestration)
└── Horizon (Dashboard)
```

### 3.2 Object Mapping: OpenStack to ACI

| OpenStack Concept | ACI Object | Notes |
|---|---|---|
| OpenStack Project (Tenant) | ACI Tenant | 1:1 mapping; BU1 OpenStack project = tn-bu1 ACI tenant |
| Neutron Network | EPG | Each Neutron network creates/maps to an ACI EPG |
| Neutron Subnet | Bridge Domain Subnet | Subnet is associated with the BD that backs the EPG |
| Neutron Router | Contract + VRF routing | Router enables inter-network (inter-EPG) communication via contracts |
| Neutron Security Group | Contract (intra-EPG filtering) | Security group rules map to ACI filters and contracts |
| Neutron Floating IP | L3Out SNAT/DNAT | Floating IP provisioned via ACI L3Out external EPG |
| Neutron Port | ACI Endpoint | Port attachment registers as an endpoint via OpFlex |
| Octavia Loadbalancer | F5 VE or Amphora | Depending on the Octavia provider driver |

### 3.3 VMM Domain Integration

A VMM (Virtual Machine Manager) domain is configured in APIC to integrate with OpenStack:

- **Domain Type**: OpenStack
- **Controller**: Points to the Neutron API endpoint
- **Credential**: Keystone service account with admin privileges on all BU projects
- **VLAN Pool**: Dynamic pool (e.g., 1000-1999) for EPG encapsulation on the fabric
- **Attachable Entity Profile (AEP)**: Links the VMM domain to the physical interface policies on leaf switches where compute nodes connect
- **OpFlex**: The OpFlex agent runs on each Nova compute node alongside OVS. It receives policy from APIC and programs local OVS flow rules. This means packet forwarding decisions happen locally on the compute node while policy is centrally managed by APIC.

### 3.4 Compute Node Connectivity

Each OpenStack compute node connects to ACI leaf switches via:

```
Compute Node (Nova + OpFlex + OVS)
├── Bond0 (2x 25G) ──→ Leaf-pair (vPC) ──→ Infra VLAN (OpFlex discovery, DHCP)
├── Bond1 (2x 25G) ──→ Leaf-pair (vPC) ──→ Tenant data (dynamic VLAN from pool)
└── CIMC/iLO ──→ OOB Mgmt switch ──→ bd-mgmt-oob
```

Interface policies on the leaf switches:
- **Interface Policy Group**: vPC policy group with LACP active
- **Access Port Selector**: Maps physical ports to the policy group
- **AEP**: Binds the VMM domain (OpenStack) to the interface policy group
- **Infra VLAN**: A single VLAN used for OpFlex communication and DHCP between compute nodes and APIC

### 3.5 Neutron Network Creation Flow

When a BU admin creates a network via Horizon or CLI:

1. User calls `openstack network create bu1-web-network --project bu1`
2. Neutron ML2 ACI driver intercepts the request
3. Driver calls APIC REST API to create EPG `epg-bu1-web-network` under the corresponding ACI Tenant `tn-bu1`, Application Profile `ap-bu1-workloads`
4. APIC allocates a VLAN from the dynamic pool and associates it with the EPG
5. When a VM is launched on this network, OpFlex on the compute node registers the endpoint (MAC, IP) with APIC
6. APIC pushes policy (which contracts apply to this EPG) to the OpFlex agent
7. OVS on the compute node enforces the policy locally

### 3.6 OpenStack Project-to-ACI Tenant Mapping

| OpenStack Project | ACI Tenant | Keystone Domain | LDAP Group |
|---|---|---|---|
| bu1-project | tn-bu1 | BU1 | cn=bu1-cloud-users |
| bu2-project | tn-bu2 | BU2 | cn=bu2-cloud-users |
| bu3-project | tn-bu3 | BU3 | cn=bu3-cloud-users |
| bu4-project | tn-bu4 | BU4 | cn=bu4-cloud-users |
| bu5-project | tn-bu5 | BU5 | cn=bu5-cloud-users |
| shared-services | tn-shared-services | Infra | cn=infra-admins |
| dmz-project | tn-dmz | Infra | cn=infra-admins |

Keystone is configured with LDAP backend (FreeIPA) so user authentication is centralized. Each BU domain maps to an LDAP organizational unit.

---

## 4. Service Graph Design: Palo Alto VM-Series (North-South Firewall)

### 4.1 Deployment Model

The Palo Alto VM-Series is inserted into the north-south traffic path between the internet and the DMZ using an ACI Service Graph. This is a **routed mode** insertion with **Policy-Based Redirect (PBR)**.

```
Internet
  │
  ▼
L3Out (l3out-internet, BGP to border routers)
  │
  ▼
External EPG (extepg-internet) ── Consumer of ctr-internet-to-dmz
  │
  │  ┌─────────────────────────────┐
  │  │   ACI Service Graph:        │
  │  │   sg-palo-alto-perimeter    │
  │  │                             │
  │  │   Function: GoTo            │
  │  │   Type: Firewall (routed)   │
  │  │   Connector: Consumer ←──── │ ── PBR to PA untrust interface
  │  │   Connector: Provider ────→ │ ── PBR from PA trust interface
  │  └─────────────────────────────┘
  │
  ▼
Provider EPG (epg-dmz-frontend) ── Provider of ctr-internet-to-dmz
```

### 4.2 ACI Service Graph Objects

**L4-L7 Device (Logical Device):**

| Attribute | Value |
|---|---|
| Name | ldev-paloalto-vseries |
| Device Type | Virtual |
| Service Type | Firewall |
| Context Aware | Single Context (single vsys) |
| Managed | Yes (managed mode -- ACI pushes config via device package) |
| Device Interface - Consumer | eth1/1 (untrust zone, connects to bd-pa-untrust) |
| Device Interface - Provider | eth1/2 (trust zone, connects to bd-pa-trust) |

**Concrete Device (maps to the actual VM-Series instances):**

| Attribute | Value |
|---|---|
| VM-Series 1 | pa-vseries-01 (active) |
| VM-Series 2 | pa-vseries-02 (standby) |
| HA Mode | Active/Passive with session sync |
| Management IP | 192.168.1.10/24 (bd-mgmt-infra) |
| Panorama | 192.168.1.5 (centralized management) |

**Service Graph Template:**

```
sg-palo-alto-perimeter:
  Node: N1-firewall
    Function: GoTo
    Device: ldev-paloalto-vseries
    Routing: Redirect (PBR)
```

**Policy-Based Redirect (PBR) Policy:**

| PBR Policy | Destination IP | Destination MAC | Health Group |
|---|---|---|---|
| pbr-pa-consumer | 10.200.1.10 (PA untrust) | auto-learned | hg-paloalto (ICMP probe) |
| pbr-pa-provider | 10.200.2.10 (PA trust) | auto-learned | hg-paloalto (ICMP probe) |

The health group monitors the Palo Alto VM-Series liveness. If the active firewall fails, ACI detects the health probe failure and redirects to the standby (or bypasses if configured for fail-open -- though fail-closed is recommended for security).

### 4.3 Palo Alto VM-Series Configuration

**Zones:**
- untrust: Interface eth1/1 (facing L3Out/internet)
- trust: Interface eth1/2 (facing DMZ EPG)
- management: mgmt interface

**Security Policy (north-south):**

| Rule | Source Zone | Dest Zone | Source | Destination | Application | Action | Security Profiles |
|---|---|---|---|---|---|---|---|
| Allow-Inbound-HTTPS | untrust | trust | any | DMZ VIPs | ssl, web-browsing | Allow | AV, Anti-Spyware, Vuln Protection, URL Filtering, WildFire |
| Allow-Inbound-HTTP-Redirect | untrust | trust | any | DMZ VIPs | web-browsing (80) | Allow | (redirect to HTTPS via LB or app) |
| Block-All-Inbound | untrust | trust | any | any | any | Deny | -- |
| Allow-DMZ-Outbound-Updates | trust | untrust | DMZ servers | update repos | ssl, apt, yum | Allow | AV, WildFire |
| Block-All-Outbound | trust | untrust | any | any | any | Deny | -- |

**SSL Decryption**: Forward proxy decryption on inbound HTTPS traffic to inspect for threats before traffic reaches the load balancer. The PA decrypts, inspects, re-encrypts, and forwards.

**User-ID**: Not applicable for north-south internet traffic. Used for internal BU-to-BU flows if an inter-zone firewall is added later.

**Dynamic Address Groups**: Configured to pull endpoint data from APIC. As VMs are created/destroyed in OpenStack, the Palo Alto policy automatically reflects the current state:

```
DAG: dag-dmz-servers
  Criteria: 'aci' and 'epg-dmz-frontend'
  Source: APIC REST API integration (registered IP tags)
```

### 4.4 Traffic Flow: Internet to DMZ (North-South through Palo Alto)

```
1. Client request (HTTPS) arrives at border router
2. Border router forwards via BGP to ACI L3Out (l3out-internet)
3. ACI classifies traffic against External EPG (extepg-internet)
4. Contract ctr-internet-to-dmz is matched
5. Service Graph sg-palo-alto-perimeter is applied
6. ACI PBR redirects traffic to Palo Alto untrust interface (10.200.1.10)
7. Palo Alto performs:
   a. SSL decryption (if configured)
   b. App-ID identification (ssl, web-browsing)
   c. Security policy lookup (Allow-Inbound-HTTPS rule matches)
   d. Content-ID inspection (AV, IPS, URL filtering, WildFire)
   e. If clean, forward out trust interface (10.200.2.10)
8. Traffic returns to ACI fabric
9. ACI delivers to epg-dmz-frontend (provider side of service graph)
10. Traffic reaches the DMZ web servers (or next service graph node - load balancer)
```

---

## 5. Service Graph Design: Load Balancer (F5 BIG-IP VE vs HAProxy/Octavia)

### 5.1 Load Balancer Selection Recommendation

**Recommended: F5 BIG-IP VE** for the following reasons in this specific architecture:

| Criterion | F5 BIG-IP VE | HAProxy (Octavia Amphora) |
|---|---|---|
| ACI Service Graph | Native device package -- ACI can steer traffic through F5 automatically and manage VIP/pool lifecycle | No ACI device package -- requires standalone deployment with manual or Octavia-driven configuration |
| Multi-tenant isolation | Route domains and partitions per BU tenant | Separate amphora VMs per loadbalancer (higher resource overhead) |
| SSL offloading | Hardware-accelerated (on VE, software but highly optimized) | Software-based (adequate for moderate loads) |
| iRules / advanced L7 | Full iRules and traffic policies for complex routing | ACLs and basic L7, Lua scripting possible but less mature |
| WAF | ASM/AWAF module available (can complement Palo Alto) | No native WAF |
| OpenStack integration | Octavia F5 provider driver or AS3-based self-service | Native Octavia amphora driver (default, well-tested) |
| Cost | Significant licensing cost | Free (HAProxy is FLOSS) |

**If budget is constrained**: HAProxy via Octavia is a fully viable option. Deploy it as the default Octavia amphora provider. You lose ACI Service Graph integration but gain simplicity and zero licensing cost. In this case, traffic steering to HAProxy is handled by Neutron routing rather than ACI PBR.

**Hybrid approach**: Use F5 BIG-IP VE for DMZ/internet-facing workloads (where ACI Service Graph integration and advanced WAF add value) and HAProxy/Octavia for internal east-west load balancing within BU tenants (where cost efficiency matters more).

### 5.2 F5 BIG-IP VE: ACI Service Graph Integration

**L4-L7 Device (Logical Device):**

| Attribute | Value |
|---|---|
| Name | ldev-f5-bigip |
| Device Type | Virtual |
| Service Type | ADC (Load Balancer) |
| Context Aware | Multi-Context (route domains per tenant) |
| Managed | Yes |
| Device Interface - Consumer | 1.1 (client-facing VLAN, connects to DMZ frontend BD) |
| Device Interface - Provider | 1.2 (server-facing VLAN, connects to BU web-tier BD) |

**Concrete Devices:**

| Attribute | Value |
|---|---|
| BIG-IP 1 | f5-bigip-01 (active) |
| BIG-IP 2 | f5-bigip-02 (standby) |
| HA Mode | Active/Standby with config sync + network failover |
| Management IP | 192.168.1.20/24, 192.168.1.21/24 |
| Floating Self-IP (client) | 172.16.1.100 (VIP subnet) |
| Floating Self-IP (server) | dynamic per BU web-tier BD |

**Service Graph Template (chained with Palo Alto):**

For internet-facing traffic, the Palo Alto and F5 are chained in a single service graph:

```
sg-perimeter-fw-lb:
  Node: N1-firewall
    Function: GoTo
    Device: ldev-paloalto-vseries
    Routing: Redirect (PBR)
  Node: N2-loadbalancer
    Function: GoTo
    Device: ldev-f5-bigip
    Routing: Redirect (PBR)
```

This two-node service graph ensures traffic flows: Internet -> Palo Alto (inspect) -> F5 (load balance) -> Backend web servers.

**F5 BIG-IP Configuration (per-BU VIP example for BU1):**

```
Virtual Server: vs-bu1-web
  Destination: 172.16.1.10:443 (VIP on DMZ frontend)
  Pool: pool-bu1-web
    Members: 10.1.10.11:443, 10.1.10.12:443, 10.1.10.13:443
    Monitor: https-health-check (GET /healthz, expect 200)
    Load Balancing Method: Least Connections
  SSL Profile (Client): clientssl-bu1 (terminate TLS from internet)
  SSL Profile (Server): serverssl-bu1 (re-encrypt to backends)
  Persistence: Cookie-based (JSESSIONID insert)
  iRule: redirect-http-to-https (redirect port 80 -> 443)
  SNAT: Automap (SNAT to F5 self-IP for return traffic symmetry)
```

### 5.3 HAProxy via Octavia (Alternative/Internal)

For internal load balancing within BU tenants:

```
openstack loadbalancer create --name bu1-internal-lb \
  --vip-subnet-id bu1-app-subnet \
  --project bu1-project

openstack loadbalancer listener create --name bu1-api-listener \
  --protocol HTTPS --protocol-port 443 \
  --loadbalancer bu1-internal-lb

openstack loadbalancer pool create --name bu1-api-pool \
  --protocol HTTPS --lb-algorithm LEAST_CONNECTIONS \
  --listener bu1-api-listener

openstack loadbalancer member create --name bu1-api-member-1 \
  --address 10.1.20.11 --protocol-port 8443 \
  bu1-api-pool
```

Octavia deploys an amphora VM (running HAProxy) in the tenant's network. The amphora is automatically registered as an ACI endpoint via OpFlex.

### 5.4 Traffic Flow: Internet to BU1 Web Application (Full Path)

```
1. Client HTTPS request arrives at border router
2. Border router → ACI L3Out (l3out-internet) via BGP
3. ACI matches destination VIP 172.16.1.10 → External EPG → Contract ctr-internet-to-dmz
4. Service Graph sg-perimeter-fw-lb is invoked:

   ┌─ Node 1: Palo Alto VM-Series ─┐
   │  a. PBR redirects to PA untrust (10.200.1.10)
   │  b. SSL decrypt → App-ID → Security policy → Content-ID
   │  c. Pass: forward out trust (10.200.2.10)
   └────────────────────────────────┘
        │
        ▼
   ┌─ Node 2: F5 BIG-IP VE ────────┐
   │  a. PBR redirects to F5 client VLAN (172.16.1.100)
   │  b. VIP 172.16.1.10:443 matched
   │  c. SSL terminate (client-side) → inspect → re-encrypt (server-side)
   │  d. Pool selection: pool-bu1-web
   │  e. Member selection: 10.1.10.12:443 (least connections)
   │  f. SNAT to F5 self-IP (ensures return traffic comes back through F5)
   └────────────────────────────────┘
        │
        ▼
5. Traffic arrives at BU1 web server VM (10.1.10.12) in epg-bu1-web
6. Web server processes request, responds
7. Return traffic follows reverse path:
   Web server → F5 (because of SNAT) → Palo Alto (PBR) → L3Out → Border router → Client
```

---

## 6. Shared Services Access Pattern

### 6.1 How BU Tenants Reach DNS, LDAP, Monitoring

ACI inter-tenant contracts with global scope enable cross-VRF communication without route leaking:

```
tn-bu1 / vrf-bu1                    tn-shared-services / vrf-shared
┌─────────────────┐                 ┌─────────────────────┐
│ epg-bu1-web     │                 │ epg-dns             │
│ epg-bu1-app     │── Consumes ────→│   Provider of       │
│ epg-bu1-db      │  ctr-shared-dns │   ctr-shared-dns    │
│ epg-bu1-internal│                 │                     │
└─────────────────┘                 └─────────────────────┘
```

**Implementation steps:**

1. In tn-shared-services, create contract `ctr-shared-dns` with filter `flt-dns` (TCP/UDP 53)
2. EPG `epg-dns` provides this contract
3. Export the contract as a "Contract Interface" from tn-shared-services
4. In each BU tenant (tn-bu1 through tn-bu5), import the contract interface
5. Configure EPGs (or use vzAny on the BU VRF) to consume the imported contract
6. ACI automatically creates the cross-VRF policy entries in hardware (TCAM)

**Using vzAny for simplicity:** Rather than explicitly attaching the shared contracts to every EPG in every BU tenant, use the vzAny construct on each BU VRF. This makes every EPG in vrf-bu1 automatically consume the shared service contracts:

```
vrf-bu1:
  vzAny:
    Consumed Contract Interfaces:
      - ctr-shared-dns
      - ctr-shared-ldap
      - ctr-shared-monitoring
      - ctr-shared-ntp
```

### 6.2 Monitoring Architecture

All BU tenants export metrics and logs to the shared monitoring stack:

```
BU Tenant VMs
  ├── node_exporter (port 9100) ──→ Prometheus (10.100.3.10)
  ├── promtail ──→ Loki (10.100.3.20)
  └── SNMP/syslog ──→ syslog aggregator (10.100.4.10)

ACI Fabric
  └── Streaming telemetry (gNMI) ──→ Telegraf ──→ Prometheus

Palo Alto VM-Series
  └── Syslog ──→ Panorama ──→ Loki (or Splunk if licensed)

F5 BIG-IP
  └── Telemetry Streaming (TS) ──→ Prometheus pushgateway

Dashboards: Grafana (10.100.3.30)
Alerting: Alertmanager (10.100.3.40) → PagerDuty/Slack/email
```

---

## 7. Complete Network Topology Diagram

```
                            INTERNET
                               │
                        ┌──────┴──────┐
                        │ Border Rtrs │ (BGP peering)
                        └──────┬──────┘
                               │
                    ┌──────────┴──────────┐
                    │   ACI L3Out         │
                    │   (l3out-internet)  │
                    │   External EPG      │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┴────── ACI Service Graph ──────────────────┐
          │                                                                │
          │  ┌──────────────────────────────────────────────────────────┐  │
          │  │  Node 1: Palo Alto VM-Series (HA pair)                  │  │
          │  │  ┌─────────────┐        ┌─────────────┐                 │  │
          │  │  │ pa-vseries-01│  HA   │ pa-vseries-02│                │  │
          │  │  │  (active)   │◄─────►│  (standby)   │                │  │
          │  │  └─────────────┘        └─────────────┘                 │  │
          │  │  Untrust (10.200.1.0/24) ←→ Trust (10.200.2.0/24)      │  │
          │  └──────────────────────────────────────────────────────────┘  │
          │                        │                                       │
          │  ┌─────────────────────┴────────────────────────────────────┐  │
          │  │  Node 2: F5 BIG-IP VE (HA pair)                         │  │
          │  │  ┌─────────────┐        ┌─────────────┐                 │  │
          │  │  │ f5-bigip-01 │  HA   │ f5-bigip-02 │                 │  │
          │  │  │  (active)   │◄─────►│  (standby)   │                │  │
          │  │  └─────────────┘        └─────────────┘                 │  │
          │  │  Client VLAN (172.16.1.0/24) ←→ Server VLANs (per BU)  │  │
          │  └──────────────────────────────────────────────────────────┘  │
          │                                                                │
          └────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────┴───┐   ┌───────┴────┐   ┌───────┴────┐
     │ tn-bu1     │   │ tn-bu2     │   │ tn-bu3..5  │
     │ vrf-bu1    │   │ vrf-bu2    │   │ vrf-bu3..5 │
     │            │   │            │   │            │
     │ Web EPG    │   │ Web EPG    │   │ Web EPG    │
     │ App EPG    │   │ App EPG    │   │ App EPG    │
     │ DB EPG     │   │ DB EPG     │   │ DB EPG     │
     └──────┬─────┘   └──────┬─────┘   └──────┬─────┘
            │                │                │
            └───── vzAny consumes ────────────┘
                         │
              ┌──────────┴──────────┐
              │ tn-shared-services  │
              │ vrf-shared          │
              │                     │
              │ DNS EPG             │
              │ LDAP EPG            │
              │ Monitoring EPG      │
              │ NTP EPG             │
              └─────────────────────┘
```

---

## 8. Automation Approach

### 8.1 Tool Selection

| Layer | Tool | Why |
|---|---|---|
| ACI Fabric | OpenTofu + cisco/aci provider | Declarative, state-tracked, FLOSS. Manages tenants, VRFs, BDs, EPGs, contracts, service graphs, L3Outs |
| OpenStack | OpenTofu + openstack provider | Manages projects, networks, subnets, routers, security groups, flavors, images |
| Palo Alto | Ansible + paloaltonetworks.panos | Playbooks for security policies, zones, interfaces, NAT rules, security profiles. Ansible suits the imperative workflow of firewall rule management |
| F5 BIG-IP | Ansible + f5networks.f5_modules + AS3 | AS3 declarations for virtual servers, pools, monitors. Ansible wraps AS3 for CI/CD integration |
| Day-2 Operations | Ansible + AWX | Runbooks for common operations: add BU tenant, onboard new workload, certificate rotation, firewall rule changes |
| IPAM/DCIM | NetBox | Source of truth for IP allocations, device inventory, rack layouts. Dynamic inventory source for Ansible |
| Secrets | HashiCorp Vault | API keys, certificates, APIC credentials, Panorama credentials, F5 admin passwords |
| CI/CD | GitLab CI (or Gitea + Woodpecker for FLOSS) | Pipeline triggers on merge to main; plan/apply stages with approval gates |
| Monitoring | Prometheus + Grafana + Loki | Infrastructure and application observability |

### 8.2 Repository Structure

```
infra-as-code/
├── tofu/
│   ├── aci/
│   │   ├── main.tf                  # Provider config
│   │   ├── tenants.tf               # All tenant definitions
│   │   ├── vrfs.tf                  # VRF per tenant
│   │   ├── bridge-domains.tf        # BDs and subnets
│   │   ├── epgs.tf                  # EPGs per tenant
│   │   ├── contracts.tf             # Contracts and filters
│   │   ├── l3outs.tf                # External connectivity
│   │   ├── service-graphs.tf        # PA + F5 service graphs
│   │   ├── vmm-domains.tf           # OpenStack VMM domain
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── openstack/
│   │   ├── main.tf
│   │   ├── projects.tf              # BU projects + role assignments
│   │   ├── networks.tf              # Neutron networks per BU
│   │   ├── security-groups.tf       # Default SGs per BU
│   │   ├── flavors.tf               # Compute flavors
│   │   ├── images.tf                # Glance images
│   │   └── octavia.tf               # LB definitions (if using Octavia)
│   └── modules/
│       ├── aci-bu-tenant/           # Reusable module: creates full BU tenant stack
│       │   ├── main.tf              # Tenant, VRF, BDs, EPGs, contracts
│       │   ├── variables.tf         # BU name, CIDR blocks, EPG list
│       │   └── outputs.tf
│       └── openstack-bu-project/    # Reusable module: creates full BU project
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
├── ansible/
│   ├── inventory/
│   │   ├── netbox.yml               # Dynamic inventory from NetBox
│   │   └── group_vars/
│   │       ├── paloalto.yml
│   │       ├── f5.yml
│   │       └── openstack.yml
│   ├── playbooks/
│   │   ├── paloalto-baseline.yml    # Zones, interfaces, base policies
│   │   ├── paloalto-policies.yml    # Security and NAT rules
│   │   ├── f5-baseline.yml          # VLANs, self-IPs, HA config
│   │   ├── f5-as3-deploy.yml        # AS3 declarations for VIPs
│   │   ├── onboard-new-bu.yml       # Full BU onboarding (ACI+OS+PA+F5)
│   │   └── day2-cert-rotation.yml   # Certificate renewal across all devices
│   └── roles/
│       ├── paloalto-common/
│       ├── f5-common/
│       └── openstack-common/
├── as3-declarations/
│   ├── bu1-web-vip.json             # F5 AS3 declaration for BU1
│   ├── bu2-web-vip.json
│   └── shared-template.json
├── ci/
│   ├── .gitlab-ci.yml               # Pipeline definition
│   └── scripts/
│       ├── tofu-plan.sh
│       ├── tofu-apply.sh
│       └── ansible-check.sh
└── docs/
    └── runbooks/
        ├── onboard-new-bu.md
        ├── firewall-rule-change.md
        └── incident-response.md
```

### 8.3 OpenTofu Module: ACI BU Tenant (Example)

```hcl
# modules/aci-bu-tenant/main.tf

variable "bu_name" {
  type = string
}

variable "bu_id" {
  type    = number
  description = "Unique BU identifier (1-5), used for IP addressing"
}

variable "tiers" {
  type    = list(string)
  default = ["web", "app", "db", "internal"]
}

resource "aci_tenant" "bu" {
  name        = "tn-${var.bu_name}"
  description = "Tenant for ${var.bu_name}"
}

resource "aci_vrf" "bu" {
  tenant_dn          = aci_tenant.bu.id
  name               = "vrf-${var.bu_name}"
  pc_enf_pref        = "enforced"
  pc_enf_dir         = "ingress"
}

resource "aci_bridge_domain" "tiers" {
  for_each           = toset(var.tiers)
  tenant_dn          = aci_tenant.bu.id
  name               = "bd-${var.bu_name}-${each.key}"
  relation_fv_rs_ctx = aci_vrf.bu.id
  arp_flood          = "no"
  unk_mac_ucast_act  = "proxy"
}

resource "aci_subnet" "tiers" {
  for_each  = toset(var.tiers)
  parent_dn = aci_bridge_domain.tiers[each.key].id
  ip        = "10.${var.bu_id}.${index(var.tiers, each.key) * 10 + 10}.1/24"
  scope     = ["private"]
}

resource "aci_application_profile" "bu" {
  tenant_dn = aci_tenant.bu.id
  name      = "ap-${var.bu_name}-workloads"
}

resource "aci_application_epg" "tiers" {
  for_each               = toset(var.tiers)
  application_profile_dn = aci_application_profile.bu.id
  name                   = "epg-${var.bu_name}-${each.key}"
  relation_fv_rs_bd      = aci_bridge_domain.tiers[each.key].id
  relation_fv_rs_dom_att = [var.vmm_domain_dn]
}

# Contract: web -> app
resource "aci_contract" "web_to_app" {
  tenant_dn = aci_tenant.bu.id
  name      = "ctr-${var.bu_name}-web-to-app"
  scope     = "tenant"
}

resource "aci_contract_subject" "web_to_app" {
  contract_dn                  = aci_contract.web_to_app.id
  name                         = "subj-app-traffic"
  relation_vz_rs_subj_filt_att = [var.flt_app_tcp_dn]
}
```

### 8.4 Ansible Playbook: Palo Alto Security Policy (Example)

```yaml
# playbooks/paloalto-policies.yml
---
- name: Configure Palo Alto VM-Series security policies
  hosts: paloalto
  connection: local
  gather_facts: false

  vars_files:
    - "{{ inventory_dir }}/group_vars/paloalto.yml"

  tasks:
    - name: Create security zones
      paloaltonetworks.panos.panos_zone:
        provider: "{{ panos_provider }}"
        name: "{{ item.name }}"
        mode: layer3
        interface: "{{ item.interface }}"
      loop:
        - { name: untrust, interface: ethernet1/1 }
        - { name: trust, interface: ethernet1/2 }

    - name: Create address objects for DMZ VIPs
      paloaltonetworks.panos.panos_address_object:
        provider: "{{ panos_provider }}"
        name: "vip-{{ item.bu }}-web"
        value: "{{ item.vip }}"
        description: "{{ item.bu }} web VIP"
      loop:
        - { bu: bu1, vip: "172.16.1.10" }
        - { bu: bu2, vip: "172.16.1.20" }
        - { bu: bu3, vip: "172.16.1.30" }
        - { bu: bu4, vip: "172.16.1.40" }
        - { bu: bu5, vip: "172.16.1.50" }

    - name: Create security policy - Allow inbound HTTPS
      paloaltonetworks.panos.panos_security_rule:
        provider: "{{ panos_provider }}"
        rule_name: Allow-Inbound-HTTPS
        source_zone: [untrust]
        destination_zone: [trust]
        source_ip: [any]
        destination_ip: [vip-bu1-web, vip-bu2-web, vip-bu3-web, vip-bu4-web, vip-bu5-web]
        application: [ssl, web-browsing]
        action: allow
        antivirus: Strict
        vulnerability: Strict
        spyware: Strict
        url_filtering: Default
        wildfire_analysis: Default
        log_end: true

    - name: Create security policy - Deny all inbound
      paloaltonetworks.panos.panos_security_rule:
        provider: "{{ panos_provider }}"
        rule_name: Block-All-Inbound
        source_zone: [untrust]
        destination_zone: [trust]
        source_ip: [any]
        destination_ip: [any]
        application: [any]
        action: deny
        log_end: true

    - name: Commit configuration
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"
```

### 8.5 CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - approve
  - apply
  - verify

variables:
  TOFU_VERSION: "1.7.0"

tofu-validate-aci:
  stage: validate
  script:
    - cd tofu/aci && tofu init && tofu validate
    - tofu fmt -check

tofu-plan-aci:
  stage: plan
  script:
    - cd tofu/aci && tofu init && tofu plan -out=aci.plan
  artifacts:
    paths: [tofu/aci/aci.plan]

tofu-apply-aci:
  stage: apply
  script:
    - cd tofu/aci && tofu apply aci.plan
  when: manual  # Manual approval gate
  only:
    - main

ansible-check-paloalto:
  stage: validate
  script:
    - ansible-playbook playbooks/paloalto-policies.yml --check --diff

ansible-apply-paloalto:
  stage: apply
  script:
    - ansible-playbook playbooks/paloalto-policies.yml
  when: manual
  only:
    - main

verify-connectivity:
  stage: verify
  script:
    - python3 ci/scripts/verify-aci-health.py   # Check APIC health scores
    - python3 ci/scripts/verify-pa-policy.py     # Validate PA policy via API
    - python3 ci/scripts/verify-f5-vips.py       # Check F5 VIP health
```

### 8.6 New BU Onboarding Workflow

Adding BU6 to the platform:

1. **Add to OpenTofu variables** (`terraform.tfvars`):
   ```hcl
   bus = {
     bu1 = { id = 1 }, bu2 = { id = 2 }, bu3 = { id = 3 },
     bu4 = { id = 4 }, bu5 = { id = 5 }, bu6 = { id = 6 }
   }
   ```

2. **Commit and push** -- pipeline runs `tofu plan` showing new ACI tenant, VRF, BDs, EPGs, contracts

3. **Approve apply** -- ACI objects created, OpenStack project created, shared service contracts imported

4. **Run Ansible onboarding playbook**:
   ```bash
   ansible-playbook playbooks/onboard-new-bu.yml -e bu_name=bu6
   ```
   This configures:
   - Palo Alto address objects and security rules for BU6 VIPs
   - F5 AS3 declaration for BU6 virtual server
   - OpenStack quotas, default security groups, keypairs

5. **Verify** -- automated tests confirm connectivity to shared services, firewall policy allows expected traffic, load balancer health checks pass

---

## 9. Security Considerations

### 9.1 Network Isolation Guarantees

- **ACI whitelist model**: No contract = no communication. BU tenants cannot reach each other unless an explicit contract exists. There are no contracts between BU tenants in this design -- they are fully isolated.
- **VRF isolation**: Each BU has a dedicated VRF. Even at the routing table level, BU1 has no routes to BU2's subnets.
- **Shared services are provider-only**: BU tenants consume shared service contracts but cannot provide services to other tenants. The contract direction enforces this.
- **Palo Alto north-south inspection**: All internet-bound and internet-origin traffic passes through the Palo Alto VM-Series with App-ID, Content-ID, and SSL decryption.
- **OpenStack security groups**: Provide an additional layer of intra-EPG filtering (even within the same EPG, security groups restrict VM-to-VM communication).

### 9.2 Management Plane Security

- APIC cluster accessible only from tn-mgmt / vrf-mgmt (192.168.1.0/24)
- Panorama accessible only from vrf-mgmt
- F5 management interfaces in vrf-mgmt
- OpenStack API endpoints in vrf-mgmt (192.168.2.0/24), exposed to BU users via L3Out or reverse proxy with authentication
- RBAC: Keystone domain-per-BU ensures BU admins can only manage their own project
- APIC RBAC: Read-only access for BU admins to view their tenant; write access restricted to infra team
- All management access requires MFA via FreeIPA + TOTP

### 9.3 Compliance and Auditing

- ACI audit log: Every APIC API call is logged with timestamp, user, and change
- Palo Alto traffic logs: Forward to Panorama and Loki for retention
- F5 request logs: Telemetry Streaming to Prometheus/Loki
- OpenStack audit middleware: All API calls logged to centralized syslog
- Drift detection: Scheduled `tofu plan` runs detect unauthorized changes to ACI or OpenStack

---

## 10. High Availability Summary

| Component | HA Mechanism | RPO | RTO |
|---|---|---|---|
| ACI APIC Cluster | 3-node cluster (tolerates 1 failure) | 0 | 0 (fabric continues on policy cache) |
| ACI Fabric | Spine-leaf redundancy, vPC to compute | 0 | ~seconds (reconvergence) |
| Palo Alto VM-Series | Active/Passive HA pair, session sync | 0 | ~10s (failover) |
| F5 BIG-IP VE | Active/Standby, config sync + network failover | 0 | ~10s (failover) |
| OpenStack Control Plane | 3-node HA (HAProxy + Keepalived fronting API services) | 0 | ~30s (VIP failover) |
| Octavia Amphora | Spare amphora pool, health-based failover | 0 | ~60s (new amphora boot) |
| Ceph Storage | 3x replication across failure domains | 0 | 0 (automatic rebalance) |

---

## 11. IP Address Allocation Summary

| Network | CIDR | Purpose |
|---|---|---|
| 10.1.10-40.0/24 | BU1 workloads (web, app, db, internal) | |
| 10.2.10-40.0/24 | BU2 workloads | |
| 10.3.10-40.0/24 | BU3 workloads | |
| 10.4.10-40.0/24 | BU4 workloads | |
| 10.5.10-40.0/24 | BU5 workloads | |
| 10.100.1-4.0/24 | Shared services (DNS, LDAP, monitoring, NTP) | |
| 10.200.1.0/24 | Palo Alto untrust transit | |
| 10.200.2.0/24 | Palo Alto trust transit | |
| 172.16.1.0/24 | DMZ frontend (VIPs) | |
| 172.16.2.0/24 | DMZ backend | |
| 192.168.1.0/24 | Management infrastructure | |
| 192.168.2.0/24 | OpenStack control plane | |
| 192.168.3.0/24 | Out-of-band management (CIMC/iLO) | |

All IP allocations are tracked in NetBox as the authoritative IPAM source.

---

## 12. Key Design Decisions and Rationale

| Decision | Choice | Rationale |
|---|---|---|
| ACI integration mode | Managed mode (ML2 driver) | OpenStack manages tenant self-service; APIC manages fabric. Best balance of automation and control. |
| Tenant-per-BU in ACI | Separate ACI tenants | Strongest isolation boundary. Prevents accidental policy leakage between BUs. |
| VRF-per-BU | Separate VRFs | Layer 3 isolation, allows overlapping IP space, independent routing tables. |
| Shared services pattern | Inter-tenant contract export/import + vzAny | Clean, scalable -- adding a new BU only requires vzAny consumption on the new VRF. |
| Firewall insertion | ACI Service Graph with PBR | Traffic steering is handled by the fabric, not by routing tricks. PBR ensures all north-south traffic hits the firewall. |
| LB for internet-facing | F5 BIG-IP VE via Service Graph | ACI-native integration, advanced L7 features, chained with Palo Alto in a single service graph. |
| LB for internal | HAProxy via Octavia | Cost-effective, tenant self-service via OpenStack API, no licensing overhead. |
| IaC tool for ACI/OpenStack | OpenTofu | FLOSS, declarative, state-tracked. Preferred over Terraform due to licensing (BSL). |
| IaC tool for PA/F5 | Ansible | Better fit for imperative device configuration. Strong module ecosystem for both platforms. |
| Monitoring | Prometheus + Grafana + Loki | FLOSS stack, covers metrics, logs, and alerting. Avoids vendor lock-in. |
| Identity | FreeIPA + Keystone LDAP backend | FLOSS, centralized auth for all BUs, TOTP MFA support. |
| IPAM | NetBox | FLOSS, authoritative source of truth for IP and device inventory. |
