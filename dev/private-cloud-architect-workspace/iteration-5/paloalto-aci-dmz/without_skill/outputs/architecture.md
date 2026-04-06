# Private Cloud DMZ Architecture: Cisco ACI with Palo Alto PA-5400 and F5 BIG-IP i5800

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [ACI Fabric Logical Design](#2-aci-fabric-logical-design)
3. [Full Traffic Path](#3-full-traffic-path)
4. [ACI Service Graph Design and Chaining](#4-aci-service-graph-design-and-chaining)
5. [Palo Alto PA-5400 HA Design](#5-palo-alto-pa-5400-ha-design)
6. [F5 BIG-IP i5800 Design](#6-f5-big-ip-i5800-design)
7. [SSL Decryption Architecture](#7-ssl-decryption-architecture)
8. [Microsegmentation with ACI Contracts](#8-microsegmentation-with-aci-contracts)
9. [Ansible Automation](#9-ansible-automation)
10. [Monitoring and Observability](#10-monitoring-and-observability)
11. [Failure Scenarios and Resilience](#11-failure-scenarios-and-resilience)

---

## 1. Architecture Overview

### Design Principles

- All north-south traffic traverses the Palo Alto PA-5400 HA pair for stateful inspection and SSL decryption before reaching any internal resource.
- The F5 BIG-IP i5800 HA pair handles Layer 7 load balancing and application delivery for public-facing services.
- ACI Service Graphs chain Palo Alto (firewall node) and F5 (load balancer node) as managed or unmanaged L4-L7 devices in a single policy-driven service chain.
- East-west traffic between application tiers is microsegmented using ACI contracts, EPGs, and filters -- no traffic between tiers is implicitly permitted.
- All configuration is deployed and maintained through Ansible playbooks using the ACI, PAN-OS, and F5 collections.

### Physical Topology

```
                        Internet / WAN Edge
                              |
                    +---------+---------+
                    |  Border Leaf Pair |
                    |  (Leaf-01/02)     |
                    +---------+---------+
                              |
                  +-----------+-----------+
                  |                       |
          +-------+-------+     +--------+--------+
          | PA-5400 Node A|     | PA-5400 Node B  |
          | (Active)      |     | (Passive)       |
          +-------+-------+     +--------+--------+
                  |                       |
                  +-----------+-----------+
                              |
                    +---------+---------+
                    | Service Leaf Pair |
                    |  (Leaf-03/04)     |
                    +---------+---------+
                              |
                  +-----------+-----------+
                  |                       |
          +-------+-------+     +--------+--------+
          | F5 i5800      |     | F5 i5800        |
          | (Active)      |     | (Standby)       |
          +-------+-------+     +--------+--------+
                  |                       |
                  +-----------+-----------+
                              |
                    +---------+---------+
                    | Compute Leaf Pair |
                    |  (Leaf-05/06)     |
                    +---------+---------+
                              |
                  +-----------+-----------+
                  |           |           |
            +-----+---+ +----+----+ +----+----+
            |OpenStack| |OpenStack| |OpenStack|
            |Compute 1| |Compute 2| |Compute 3|
            +---------+ +---------+ +---------+
```

### Physical Connectivity

| Device | Interface | Connected To | Speed | Purpose |
|---|---|---|---|---|
| PA-5400 Node A | eth1/1-1/2 | Leaf-01/02 (outside) | 2x25G LACP | Untrusted / Internet-facing |
| PA-5400 Node A | eth1/3-1/4 | Leaf-03/04 (inside) | 2x25G LACP | Trusted / DMZ-facing |
| PA-5400 Node A | eth1/5 | PA-5400 Node B eth1/5 | 25G | HA1 (control) |
| PA-5400 Node A | eth1/6 | PA-5400 Node B eth1/6 | 25G | HA2 (data/state sync) |
| PA-5400 Node A | mgmt | OOB mgmt switch | 1G | Management |
| PA-5400 Node B | (mirrors Node A) | | | |
| F5 i5800 Active | 1.1-1.2 | Leaf-03/04 (external VLAN) | 2x25G LACP | Client-side (from PA) |
| F5 i5800 Active | 1.3-1.4 | Leaf-05/06 (internal VLAN) | 2x25G LACP | Server-side (to VMs) |
| F5 i5800 Active | HA | F5 Standby HA port | 25G | HA mirroring |
| F5 i5800 Active | mgmt | OOB mgmt switch | 1G | Management |
| F5 i5800 Standby | (mirrors Active) | | | |

---

## 2. ACI Fabric Logical Design

### Tenant Structure

```
Tenant: TN_DMZ
├── VRF: VRF_DMZ
│   ├── BD: BD_OUTSIDE          (10.1.1.0/24)    -- PA outside leg
│   ├── BD: BD_PA_TO_F5         (10.1.2.0/24)    -- PA inside / F5 external
│   ├── BD: BD_DMZ_WEB          (10.1.10.0/24)   -- Web-tier VMs
│   ├── BD: BD_DMZ_API          (10.1.11.0/24)   -- API gateway VMs
│   ├── BD: BD_DMZ_APP          (10.1.12.0/24)   -- App-tier VMs
│   └── BD: BD_DMZ_DB           (10.1.13.0/24)   -- DB-tier VMs
│
├── Application Profile: AP_DMZ_SERVICES
│   ├── EPG: EPG_OUTSIDE        (BD_OUTSIDE)
│   ├── EPG: EPG_PA_TO_F5       (BD_PA_TO_F5)
│   ├── EPG: EPG_WEB_TIER       (BD_DMZ_WEB)      -- VMM domain
│   ├── EPG: EPG_API_TIER       (BD_DMZ_API)      -- VMM domain
│   ├── EPG: EPG_APP_TIER       (BD_DMZ_APP)      -- VMM domain
│   └── EPG: EPG_DB_TIER        (BD_DMZ_DB)       -- VMM domain
│
├── L4-L7 Devices
│   ├── PA5400_HA_CLUSTER       (Unmanaged, routed mode)
│   └── F5_BIGIP_HA_CLUSTER    (Unmanaged, routed mode)
│
├── Service Graph Template: SGT_INBOUND_CHAIN
│   └── Node1: PA5400 (firewall) -> Node2: F5 (load-balancer)
│
└── Contracts
    ├── CON_INTERNET_TO_DMZ     (service graph attached)
    ├── CON_WEB_TO_API          (microsegmented)
    ├── CON_API_TO_APP          (microsegmented)
    ├── CON_APP_TO_DB           (microsegmented)
    └── CON_MGMT_ACCESS         (management)
```

### Bridge Domain Configuration

| Bridge Domain | Subnet | Unicast Routing | ARP Flooding | L2 Unknown Unicast | L3Out Association |
|---|---|---|---|---|---|
| BD_OUTSIDE | 10.1.1.1/24 | Enabled | Disabled | Hardware Proxy | L3OUT_INTERNET |
| BD_PA_TO_F5 | 10.1.2.1/24 | Enabled | Disabled | Hardware Proxy | None |
| BD_DMZ_WEB | 10.1.10.1/24 | Enabled | Disabled | Hardware Proxy | None |
| BD_DMZ_API | 10.1.11.1/24 | Enabled | Disabled | Hardware Proxy | None |
| BD_DMZ_APP | 10.1.12.1/24 | Enabled | Disabled | Hardware Proxy | None |
| BD_DMZ_DB | 10.1.13.1/24 | Enabled | Disabled | Hardware Proxy | None |

### VMM Domain Integration

The compute EPGs (EPG_WEB_TIER, EPG_API_TIER, EPG_APP_TIER, EPG_DB_TIER) are associated with a VMM domain for OpenStack integration:

- **VMM Domain Type**: OpenStack
- **Controller**: OpenStack Neutron via opflex-agent or ACI Neutron plugin (GBP or unified plugin)
- **VLAN Pool**: VLAN_POOL_DMZ_DYNAMIC (range: 2000-2199, allocation: dynamic)
- **Attachable Entity Profile**: AEP_COMPUTE links the VMM domain and physical domain to the leaf interfaces

---

## 3. Full Traffic Path

### Inbound Traffic (Internet Client to Web Server VM)

```
Step 1:  Client HTTPS request arrives at border router / WAN edge
Step 2:  Routed to ACI fabric via L3Out on Leaf-01/02
Step 3:  ACI classifies traffic into EPG_OUTSIDE (pcTag assigned)
Step 4:  Contract CON_INTERNET_TO_DMZ is matched
         -> Service Graph SGT_INBOUND_CHAIN is invoked
Step 5:  ACI PBR (Policy-Based Redirect) steers traffic to PA-5400 outside interface
         -> Arrives at PA Zone: OUTSIDE (eth1/1 vPC, VLAN 100)
Step 6:  PA-5400 performs:
         a. SSL Decryption (inbound / SSL Forward Proxy for known certs)
         b. Threat Prevention (IPS, Anti-Virus, Anti-Spyware, WildFire)
         c. URL Filtering
         d. Security policy evaluation (zone OUTSIDE -> DMZ)
         e. Re-encrypts (optional) or passes cleartext to F5
Step 7:  PA-5400 forwards permitted traffic out inside interface
         -> ACI receives on EPG_PA_TO_F5 (VLAN 200)
Step 8:  ACI PBR steers to F5 BIG-IP external VLAN
         -> F5 receives on external VLAN (10.1.2.0/24 network)
Step 9:  F5 processes via Virtual Server:
         a. Optional additional SSL termination (if PA re-encrypted)
         b. iRule / LTM policy evaluation
         c. Pool member selection (round-robin / least-connections)
         d. SNAT to F5 self-IP (to maintain return-path symmetry)
Step 10: F5 forwards to selected pool member (web server VM)
         -> Exits F5 internal VLAN
Step 11: ACI delivers to EPG_WEB_TIER on compute leaf
         -> OpenStack VM receives the request
```

### Return Traffic (Web Server VM to Internet Client)

```
Step 1:  VM responds -> hits ACI fabric on EPG_WEB_TIER
Step 2:  Because F5 used SNAT, return traffic destination is F5 self-IP
         -> ACI forwards to F5 internal VLAN
Step 3:  F5 translates back to original client IP, sends out external VLAN
Step 4:  ACI PBR ensures return traffic traverses PA-5400 inside interface
Step 5:  PA-5400 matches existing session, applies inspection
         -> Re-encrypts to client (SSL decryption operates bidirectionally on session)
Step 6:  PA-5400 sends out outside interface
Step 7:  ACI forwards to L3Out on Leaf-01/02
Step 8:  Traffic egresses to Internet / WAN edge
```

### East-West Traffic (Web Tier to API Tier)

```
Step 1:  Web VM sends request to API endpoint
Step 2:  ACI classifies source as EPG_WEB_TIER, destination as EPG_API_TIER
Step 3:  Contract CON_WEB_TO_API evaluated:
         -> Filter: permit TCP dst-port 8443 (API HTTPS)
         -> Filter: permit TCP dst-port 443
         -> All other traffic: implicit deny
Step 4:  If contract has no service graph, traffic flows directly at fabric speed
         (east-west stays within ACI, no hairpin through firewall)
Step 5:  API VM receives request
```

> **Design Decision**: East-west traffic between tiers is microsegmented via ACI contracts but does NOT traverse the Palo Alto. This avoids unnecessary latency and firewall throughput bottlenecks. If regulatory requirements mandate firewall inspection of east-west traffic, a separate service graph can be attached to each inter-tier contract to hairpin through the PA-5400 or a dedicated east-west firewall pair.

---

## 4. ACI Service Graph Design and Chaining

### L4-L7 Device Registration

#### Palo Alto PA-5400 HA Cluster

```
L4-L7 Device:       PA5400_HA_CLUSTER
Device Type:         Firewall
Service Type:        FW
Mode:                Routed (GoTo mode)
Managed:             No (unmanaged -- Palo Alto managed via Panorama/Ansible)
Context Aware:       Single Context
Concrete Devices:
  ├── CDev_PA_A:
  │     Interface: outside  -> Leaf-01 eth1/49, Leaf-02 eth1/49 (vPC)
  │     Interface: inside   -> Leaf-03 eth1/49, Leaf-04 eth1/49 (vPC)
  └── CDev_PA_B:
        Interface: outside  -> Leaf-01 eth1/50, Leaf-02 eth1/50 (vPC)
        Interface: inside   -> Leaf-03 eth1/50, Leaf-04 eth1/50 (vPC)

HA Mode:             Active/Passive
Cluster Interface:
  outside -> vPC to both PA nodes (ACI sees one logical vPC per leg)
  inside  -> vPC to both PA nodes
```

#### F5 BIG-IP i5800 HA Cluster

```
L4-L7 Device:       F5_BIGIP_HA_CLUSTER
Device Type:         ADC (Application Delivery Controller)
Service Type:        ADC
Mode:                Routed (GoTo mode)
Managed:             No (unmanaged -- F5 managed via AS3/Ansible)
Context Aware:       Single Context
Concrete Devices:
  ├── CDev_F5_ACTIVE:
  │     Interface: external -> Leaf-03 eth1/47, Leaf-04 eth1/47 (vPC)
  │     Interface: internal -> Leaf-05 eth1/49, Leaf-06 eth1/49 (vPC)
  └── CDev_F5_STANDBY:
        Interface: external -> Leaf-03 eth1/48, Leaf-04 eth1/48 (vPC)
        Interface: internal -> Leaf-05 eth1/50, Leaf-06 eth1/50 (vPC)
```

### Service Graph Template

```
Service Graph: SGT_INBOUND_CHAIN
Type: L4-L7 Service Graph Template

  Consumer EPG                                           Provider EPG
  (EPG_OUTSIDE)                                          (EPG_WEB_TIER)
       |                                                       |
       +--->[Node1: PA5400_FW]--->[Node2: F5_LB]----->--------+
            Function: GoTo       Function: GoTo
            Type: Firewall       Type: ADC
            Adjacency: L3        Adjacency: L3

Connectors:
  Consumer -> Node1.consumer (outside leg)
  Node1.provider -> Node2.consumer (PA inside -> F5 external transit)
  Node2.provider -> Provider (F5 internal -> server EPG)
```

### Policy-Based Redirect (PBR)

PBR is mandatory for unmanaged service graph insertion with routed devices. Each connector in the service graph gets a PBR policy.

#### PBR Policies

```
PBR Policy: PBR_TO_PA_OUTSIDE
  Destination:
    - IP: 10.1.1.10 (PA-5400 Active floating outside IP)
    - MAC: auto-resolved via ACI (health tracking)
  HA:
    - Resilient Hashing: Enabled
    - Threshold: Enabled (track PA health via IP-SLA/BFD)
    - Redirect Backup: 10.1.1.11 (PA Node B outside IP)

PBR Policy: PBR_PA_INSIDE_TO_F5_EXTERNAL
  Destination:
    - IP: 10.1.2.10 (F5 Active external floating self-IP)
    - MAC: auto-resolved via ACI
  HA:
    - Resilient Hashing: Enabled
    - Threshold: Enabled
    - Redirect Backup: 10.1.2.11 (F5 Standby external self-IP)

PBR Policy: PBR_F5_INTERNAL_TO_SERVERS
  (Not required if F5 uses SNAT -- traffic routes normally via ACI fabric)
```

### Service Graph Instance and Device Selection Policy

```
Device Selection Policy:
  Contract: CON_INTERNET_TO_DMZ
  Graph: SGT_INBOUND_CHAIN

  Node1 (PA5400_FW):
    Logical Device: PA5400_HA_CLUSTER
    Consumer Connector:
      BD: BD_OUTSIDE
      Redirect Policy: PBR_TO_PA_OUTSIDE
    Provider Connector:
      BD: BD_PA_TO_F5
      Redirect Policy: PBR_PA_INSIDE_TO_F5_EXTERNAL

  Node2 (F5_LB):
    Logical Device: F5_BIGIP_HA_CLUSTER
    Consumer Connector:
      BD: BD_PA_TO_F5
      (PBR handled by PA provider connector)
    Provider Connector:
      BD: BD_DMZ_WEB
      (No PBR needed -- F5 routes to server BD via ACI)
```

---

## 5. Palo Alto PA-5400 HA Design

### HA Configuration

| Parameter | Value |
|---|---|
| HA Mode | Active/Passive |
| HA1 Link | Dedicated eth1/5 (control plane, encrypted) |
| HA2 Link | Dedicated eth1/6 (session state sync) |
| HA1 Backup | Management port (backup path) |
| Election Priority | Node A: 100 (primary), Node B: 90 |
| Preemption | Disabled (manual failback after root-cause analysis) |
| HA Timer Profile | Aggressive (hello: 1000ms, heartbeat: 1000ms) |
| Session Sync | Enabled (all sessions) |
| HA Monitoring | Link monitoring on eth1/1, eth1/3 + path monitoring to ACI BD gateways |

### Zone Architecture

```
Zones:
├── OUTSIDE
│   ├── Type: Layer 3
│   ├── Interfaces: ae1 (aggregate of eth1/1-1/2, or sub-interface for VLAN 100)
│   ├── IP: 10.1.1.10/24 (floating), 10.1.1.12/24 (Node A), 10.1.1.13/24 (Node B)
│   ├── Zone Protection Profile: ZP_OUTSIDE
│   │   ├── Flood Protection: SYN (SYN cookies, 10K pps alarm, 50K pps activate, 100K pps max)
│   │   ├── Flood Protection: ICMP (1K pps), UDP (10K pps)
│   │   ├── Reconnaissance Protection: TCP/UDP port scans (block, threshold: 100 in 10s)
│   │   └── Packet-Based Attack Protection: Spoofed IP, Fragmented Traffic, Overlapping TCP Segments
│   └── Log Forwarding: LOG_PROFILE_OUTSIDE
│
├── DMZ
│   ├── Type: Layer 3
│   ├── Interfaces: ae2 (aggregate of eth1/3-1/4, or sub-interface for VLAN 200)
│   ├── IP: 10.1.2.20/24 (floating), 10.1.2.22/24 (Node A), 10.1.2.23/24 (Node B)
│   ├── Zone Protection Profile: ZP_DMZ
│   │   ├── Flood Protection: SYN (10K pps), UDP (5K pps)
│   │   └── Packet-Based Attack Protection: Spoofed IP
│   └── Log Forwarding: LOG_PROFILE_DMZ
│
└── MGMT (out-of-band)
    ├── Type: Layer 3
    ├── Interface: mgmt
    └── IP: 192.168.100.10/24 (Node A), 192.168.100.11/24 (Node B)
```

### Routing on PA-5400

```
Virtual Router: VR_DMZ
├── Static Routes:
│   ├── 0.0.0.0/0 -> 10.1.1.1 (ACI BD_OUTSIDE gateway) [metric 10]
│   ├── 10.1.10.0/24 -> 10.1.2.1 (ACI BD_PA_TO_F5 gateway) [web tier via F5]
│   ├── 10.1.11.0/24 -> 10.1.2.1 (ACI BD_PA_TO_F5 gateway) [API tier via F5]
│   └── 10.1.0.0/16 -> 10.1.2.1 (summary for all DMZ subnets)
│
├── BGP: Not used (PBR handles steering in ACI)
└── Path Monitoring:
    ├── Monitor 10.1.1.1 (outside gateway) - fail triggers HA failover
    └── Monitor 10.1.2.1 (inside gateway) - fail triggers HA failover
```

### Security Policies (ordered)

```
Rule 1: ALLOW_INBOUND_HTTPS
  Source Zone:      OUTSIDE
  Destination Zone: DMZ
  Source Address:   any
  Destination:     10.1.2.10 (F5 VIP, pre-NAT) or public VIP if NAT on PA
  Application:      ssl, web-browsing
  Service:          tcp/443, tcp/8443
  Action:           Allow
  Profile Group:    PG_STRICT_INBOUND
    ├── Antivirus:       AV_STRICT
    ├── Anti-Spyware:    AS_STRICT
    ├── Vulnerability:   VP_STRICT (IPS)
    ├── WildFire:        WF_INLINE
    ├── File Blocking:   FB_BLOCK_EXECUTABLES
    └── URL Filtering:   URL_DMZ_RESTRICTED
  SSL Decryption:   DECRYPT_INBOUND_HTTPS (see Section 7)
  Logging:          Start + End, LOG_PROFILE_SIEM

Rule 2: ALLOW_INBOUND_HTTP_REDIRECT
  Source Zone:      OUTSIDE
  Destination Zone: DMZ
  Source Address:   any
  Destination:     VIP addresses
  Application:      web-browsing
  Service:          tcp/80
  Action:           Allow (F5 will redirect to HTTPS)
  Profile Group:    PG_STANDARD
  Logging:          End, LOG_PROFILE_SIEM

Rule 3: ALLOW_INBOUND_API
  Source Zone:      OUTSIDE
  Destination Zone: DMZ
  Source Address:   any (or known API consumer CIDRs)
  Destination:     API VIP address
  Application:      ssl
  Service:          tcp/443
  Action:           Allow
  Profile Group:    PG_STRICT_INBOUND
  SSL Decryption:   DECRYPT_INBOUND_API
  Logging:          Start + End, LOG_PROFILE_SIEM

Rule 4: DENY_OUTBOUND_DMZ_TO_INTERNET
  Source Zone:      DMZ
  Destination Zone: OUTSIDE
  Source Address:   any
  Destination:     any
  Application:      any
  Service:          any
  Action:           Deny
  Logging:          Start + End, LOG_PROFILE_SIEM
  Note:             DMZ servers must not initiate outbound connections.
                    Exceptions added as needed (e.g., CRL/OCSP, update repos).

Rule 5: ALLOW_DMZ_CRL_OCSP
  Source Zone:      DMZ
  Destination Zone: OUTSIDE
  Source Address:   10.1.2.0/24
  Destination:     CRL/OCSP responder IPs (address group)
  Application:      ssl, web-browsing, ocsp
  Service:          tcp/80, tcp/443
  Action:           Allow
  Profile Group:    PG_STANDARD
  Logging:          End

Rule 6: DENY_ALL (implicit, logged)
  Source Zone:      any
  Destination Zone: any
  Action:           Deny
  Logging:          End, LOG_PROFILE_SIEM
```

### NAT Policies

```
NAT Rule 1: DNAT_INBOUND_WEB
  Type:            Destination NAT
  Source Zone:     OUTSIDE
  Dest Zone:       OUTSIDE (pre-NAT lookup)
  Original Dest:   203.0.113.10 (public VIP)
  Translated Dest: 10.1.2.10 (F5 external VIP)
  Service:         tcp/443, tcp/80

NAT Rule 2: DNAT_INBOUND_API
  Type:            Destination NAT
  Source Zone:     OUTSIDE
  Dest Zone:       OUTSIDE
  Original Dest:   203.0.113.11 (public API VIP)
  Translated Dest: 10.1.2.11 (F5 external API VIP)
  Service:         tcp/443
```

---

## 6. F5 BIG-IP i5800 Design

### HA Configuration

| Parameter | Value |
|---|---|
| HA Mode | Active/Standby |
| Config Sync | Automatic (Device Trust Group: DG_DMZ_F5) |
| Failover Type | Network failover |
| Failover Unicast | Management + HA VLAN |
| Mirroring | Connection mirroring enabled for persistence |
| Auto-failback | Disabled |

### Network Configuration

```
VLANs:
├── VLAN external (tag 200)
│   ├── Interfaces: 1.1, 1.2 (trunk)
│   ├── Self-IP (Active):  10.1.2.10/24 (floating)
│   ├── Self-IP (Node A):  10.1.2.14/24
│   ├── Self-IP (Node B):  10.1.2.15/24
│   └── Port Lockdown:     Allow Default (for health monitors)
│
├── VLAN internal (tag 210)
│   ├── Interfaces: 1.3, 1.4 (trunk)
│   ├── Self-IP (Active):  10.1.10.10/24 (floating)
│   ├── Self-IP (Node A):  10.1.10.14/24
│   ├── Self-IP (Node B):  10.1.10.15/24
│   └── Port Lockdown:     Allow Default
│
└── VLAN ha (tag 4094)
    ├── Interface: dedicated HA port
    └── Self-IPs: 169.254.1.1/24, 169.254.1.2/24

Routes:
├── Default route: 10.1.2.1 (ACI BD_PA_TO_F5 gateway via external VLAN)
├── 10.1.10.0/24: directly connected (internal VLAN)
├── 10.1.11.0/24: 10.1.10.1 (ACI gateway for API subnet)
├── 10.1.12.0/24: 10.1.10.1 (ACI gateway for app subnet)
└── 10.1.13.0/24: 10.1.10.1 (ACI gateway for DB subnet)

SNAT Pool:
├── SNAT_POOL_WEB: 10.1.10.20 - 10.1.10.25
└── SNAT_POOL_API: 10.1.10.30 - 10.1.10.35
```

### Virtual Server Configuration

#### Web Tier Virtual Server

```
Virtual Server: VS_DMZ_WEB_HTTPS
  Type:                Standard
  Destination:         10.1.2.10:443
  IP Protocol:         TCP
  Source Address:       0.0.0.0/0
  VLAN/Tunnel:         external (enabled on)

  Profiles:
  ├── TCP (client-side):    tcp-wan-optimized
  ├── TCP (server-side):    tcp-lan-optimized
  ├── HTTP:                 http_with_xff
  │   ├── Insert X-Forwarded-For: Enabled
  │   ├── Insert X-Forwarded-Proto: Enabled (via iRule)
  │   ├── Response Headers: Remove Server header
  │   └── Max Header Size: 32768
  ├── SSL (Client):         Note: if PA re-encrypts, clientssl_dmz_web profile needed
  │   │                     If PA passes cleartext, use HTTP profile only
  │   ├── Certificate:      wildcard.dmz.example.com (if doing SSL on F5)
  │   ├── Chain:            intermediate-ca-bundle
  │   ├── Ciphers:          TLS1.2+ECDHE+AESGCM:TLS1.3
  │   └── Options:          No TLSv1, No TLSv1.1, No SSLv3
  ├── OneConnect:           oneconnect_pool
  ├── HTTP Compression:     wan-optimized-compression
  └── Web Acceleration:     optimized-caching (if applicable)

  iRules:
  ├── IRULE_SECURITY_HEADERS
  │   (Adds Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options)
  ├── IRULE_MAINTENANCE_PAGE
  │   (Returns maintenance page if all pool members down)
  └── IRULE_LOGGING
      (Logs client IP, URI, response code, latency to HSL pool)

  Default Pool:          POOL_WEB_SERVERS
  SNAT:                  SNAT_POOL_WEB
  Address Translation:   Enabled
  Port Translation:      Enabled
  Connection Limit:      50000
  Rate Limiting:         1000 connections/second (per source)

Virtual Server: VS_DMZ_WEB_HTTP_REDIRECT
  Destination:           10.1.2.10:80
  iRule:                 IRULE_HTTPS_REDIRECT
    (HTTP::respond 301 Location "https://[HTTP::host][HTTP::uri]")
```

#### API Gateway Virtual Server

```
Virtual Server: VS_DMZ_API_HTTPS
  Type:                Standard
  Destination:         10.1.2.11:443
  IP Protocol:         TCP

  Profiles:
  ├── TCP (client-side):   tcp-wan-optimized
  ├── TCP (server-side):   tcp-lan-optimized
  ├── HTTP:                http_api_profile
  │   ├── Insert X-Forwarded-For: Enabled
  │   ├── Max Header Size: 65536 (APIs may have large auth headers)
  │   └── Enforcement: RFC compliance strict
  ├── OneConnect:          Disabled (APIs often use unique connections)
  └── Request Logging:     Enabled (log every API call)

  Policies:
  ├── LTM_POLICY_API_ROUTING
  │   ├── /v1/*    -> POOL_API_V1
  │   ├── /v2/*    -> POOL_API_V2
  │   └── /health  -> POOL_API_HEALTH (local response)
  └── LTM_POLICY_RATE_LIMIT
      (Rate limit by API key header or source IP)

  Default Pool:          POOL_API_SERVERS
  SNAT:                  SNAT_POOL_API
  Connection Limit:      100000
```

### Pool Configuration

```
Pool: POOL_WEB_SERVERS
  Load Balancing:    Least Connections (Member)
  Priority Group:    Enabled (active group priority 10, standby priority 5)
  Min Active Members: 1
  Monitor:           MON_HTTP_WEB (interval 5s, timeout 16s)
                     GET /health -> expect 200
  Members:
  ├── 10.1.10.101:8080 (web-vm-01, priority 10)
  ├── 10.1.10.102:8080 (web-vm-02, priority 10)
  ├── 10.1.10.103:8080 (web-vm-03, priority 10)
  └── 10.1.10.104:8080 (web-vm-04, priority 5, standby)

  Slow Ramp Time:    30 seconds (new members ramp gradually)

Pool: POOL_API_SERVERS
  Load Balancing:    Round Robin
  Monitor:           MON_HTTPS_API (interval 5s, timeout 16s)
                     GET /v1/health -> expect 200, content match "ok"
  Members:
  ├── 10.1.11.101:8443 (api-gw-01)
  ├── 10.1.11.102:8443 (api-gw-02)
  └── 10.1.11.103:8443 (api-gw-03)
```

---

## 7. SSL Decryption Architecture

### Design: Inbound SSL Inspection on Palo Alto

For inbound HTTPS inspection, the PA-5400 decrypts traffic using the server's private key (or a copy of it). This is SSL Inbound Inspection, not Forward Proxy.

```
Traffic Flow with SSL Decryption:

Client ──[TLS 1.3]──> PA-5400 ──[cleartext or re-TLS]──> F5 ──[HTTP/HTTPS]──> Web VM

The PA-5400 holds a copy of the server certificate and private key
for each VIP it decrypts. It terminates the client TLS session,
inspects the cleartext, then either:
  Option A: Forwards cleartext HTTP to F5 (simpler, higher performance)
  Option B: Re-encrypts to F5 (defense-in-depth on transit segment)
```

### Recommended Option: Option A (Cleartext to F5)

Since the PA-to-F5 transit network (BD_PA_TO_F5) is a dedicated, isolated bridge domain within ACI with no other endpoints, cleartext forwarding is acceptable and avoids double decryption overhead.

### SSL Decryption Configuration on PA-5400

```
SSL Decryption Profile: SSLPROF_INBOUND_WEB
  Mode:                    SSL Inbound Inspection
  Certificate:             wildcard.dmz.example.com (PEM + private key)
  Min TLS Version:         TLSv1.2
  Max TLS Version:         TLSv1.3
  Supported Ciphers:       ECDHE-RSA-AES256-GCM-SHA384,
                           ECDHE-RSA-AES128-GCM-SHA256,
                           TLS_AES_256_GCM_SHA384 (TLS 1.3),
                           TLS_AES_128_GCM_SHA256 (TLS 1.3)
  Block Unsupported Versions: Yes
  Block Unsupported Ciphers:  Yes

SSL Decryption Profile: SSLPROF_INBOUND_API
  Mode:                    SSL Inbound Inspection
  Certificate:             api.dmz.example.com (PEM + private key)
  (Same TLS settings as above)

Decryption Policy Rule: DECRYPT_INBOUND_HTTPS
  Source Zone:            OUTSIDE
  Destination Zone:       DMZ
  Destination Address:    VIP addresses (203.0.113.10, 203.0.113.11)
  Service:                tcp/443
  Action:                 Decrypt
  Type:                   SSL Inbound Inspection
  Decryption Profile:     SSLPROF_INBOUND_WEB
  Logging:                Enabled

Decryption Policy Rule: DECRYPT_INBOUND_API
  Source Zone:            OUTSIDE
  Destination Zone:       DMZ
  Destination Address:    203.0.113.11
  Service:                tcp/443
  Action:                 Decrypt
  Type:                   SSL Inbound Inspection
  Decryption Profile:     SSLPROF_INBOUND_API

Decryption Policy Rule: NO_DECRYPT_OTHER
  Source Zone:            any
  Destination Zone:       any
  Action:                 No Decrypt
```

### SSL Certificate Management

```
Certificate Store on PA-5400:
├── wildcard.dmz.example.com
│   ├── Type: Server certificate (with private key)
│   ├── Usage: SSL Inbound Inspection for web VIPs
│   ├── Renewal: Automated via Ansible + Let's Encrypt or internal CA
│   └── Key Storage: PA-5400 HSM module (if available) or encrypted keystore
│
├── api.dmz.example.com
│   ├── Type: Server certificate (with private key)
│   └── Usage: SSL Inbound Inspection for API VIPs
│
└── Intermediate CA Bundle
    └── For certificate chain validation
```

### Hardware SSL Offload Considerations

The PA-5400 series includes dedicated SSL/TLS processing hardware. Capacity planning:

| Metric | PA-5400 Capacity | Expected Load | Headroom |
|---|---|---|---|
| SSL Decryption Throughput | ~20 Gbps | ~5 Gbps | 75% |
| New SSL Sessions/sec | ~120,000 | ~30,000 | 75% |
| Concurrent SSL Sessions | ~4,000,000 | ~500,000 | 87% |

---

## 8. Microsegmentation with ACI Contracts

### Contract Definitions

#### North-South Contract (with Service Graph)

```
Contract: CON_INTERNET_TO_DMZ
  Scope:     Tenant
  QoS Class: Unspecified

  Subject: SUB_HTTPS_INBOUND
    Service Graph: SGT_INBOUND_CHAIN
    Reverse Filter Ports: Yes
    Filters:
    ├── FLT_HTTPS:    permit tcp dst-port 443
    ├── FLT_HTTP:     permit tcp dst-port 80
    └── FLT_API:      permit tcp dst-port 8443

  Consumer: EPG_OUTSIDE
  Provider: EPG_WEB_TIER
```

#### East-West Microsegmentation Contracts

```
Contract: CON_WEB_TO_API
  Scope:     VRF
  Subject: SUB_API_ACCESS
    Filters:
    ├── FLT_API_HTTPS:    permit tcp dst-port 8443
    ├── FLT_API_REST:     permit tcp dst-port 443
    └── FLT_DNS:          permit udp dst-port 53

  Consumer: EPG_WEB_TIER
  Provider: EPG_API_TIER

Contract: CON_API_TO_APP
  Scope:     VRF
  Subject: SUB_APP_ACCESS
    Filters:
    ├── FLT_APP_HTTPS:    permit tcp dst-port 8443
    ├── FLT_APP_GRPC:     permit tcp dst-port 50051
    └── FLT_DNS:          permit udp dst-port 53

  Consumer: EPG_API_TIER
  Provider: EPG_APP_TIER

Contract: CON_APP_TO_DB
  Scope:     VRF
  Subject: SUB_DB_ACCESS
    Filters:
    ├── FLT_MYSQL:        permit tcp dst-port 3306
    ├── FLT_POSTGRES:     permit tcp dst-port 5432
    └── FLT_REDIS:        permit tcp dst-port 6379

  Consumer: EPG_APP_TIER
  Provider: EPG_DB_TIER
```

#### Intra-EPG Isolation

```
EPG_WEB_TIER:
  Intra-EPG Isolation: Unenforced (web servers may need to communicate)

EPG_API_TIER:
  Intra-EPG Isolation: Unenforced (API nodes may cluster)

EPG_APP_TIER:
  Intra-EPG Isolation: Unenforced

EPG_DB_TIER:
  Intra-EPG Isolation: Enforced (DB nodes communicate only via replication
                                  contract or specific intra-EPG contract)
```

#### Denied Flows (Explicit)

By ACI design, any traffic without a matching contract is implicitly denied. The following paths have NO contract and are therefore blocked:

- EPG_WEB_TIER -> EPG_APP_TIER (web cannot bypass API tier)
- EPG_WEB_TIER -> EPG_DB_TIER (web cannot access database directly)
- EPG_API_TIER -> EPG_DB_TIER (API cannot access database directly)
- EPG_DB_TIER -> any other EPG (database cannot initiate connections)
- EPG_OUTSIDE -> EPG_API_TIER, EPG_APP_TIER, EPG_DB_TIER (internet cannot reach internal tiers)

### VRF Enforcement

```
VRF: VRF_DMZ
  Policy Control Enforcement: Enforced
  Policy Control Direction:   Ingress
  Preferred Group:            Disabled (all traffic must match explicit contracts)
```

---

## 9. Ansible Automation

### Repository Structure

```
ansible-dmz-infra/
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── aci.yml
│   │       ├── paloalto.yml
│   │       ├── f5.yml
│   │       └── all.yml            # Shared vars (vault-encrypted secrets)
│   └── staging/
│       └── ...
│
├── playbooks/
│   ├── site.yml                   # Master playbook (includes all)
│   ├── 01_aci_fabric.yml          # Tenant, VRF, BD, EPG, contracts
│   ├── 02_aci_l4l7_devices.yml   # L4-L7 device registration
│   ├── 03_aci_service_graph.yml  # Service graph + PBR
│   ├── 04_paloalto_base.yml      # PA zones, interfaces, routing
│   ├── 05_paloalto_ha.yml        # PA HA configuration
│   ├── 06_paloalto_policies.yml  # PA security + NAT + decryption
│   ├── 07_f5_base.yml            # F5 VLANs, self-IPs, routing
│   ├── 08_f5_ha.yml              # F5 device trust + failover
│   ├── 09_f5_app_services.yml    # F5 virtual servers, pools, profiles
│   └── 10_validation.yml         # End-to-end validation
│
├── roles/
│   ├── aci_tenant/
│   ├── aci_networking/
│   ├── aci_contracts/
│   ├── aci_service_graph/
│   ├── paloalto_base/
│   ├── paloalto_security/
│   ├── paloalto_ssl_decrypt/
│   ├── f5_networking/
│   ├── f5_ltm/
│   └── validation/
│
├── collections/
│   └── requirements.yml
│       # - cisco.aci (>= 2.8.0)
│       # - paloaltonetworks.panos (>= 2.17.0)
│       # - f5networks.f5_modules (>= 1.25.0)
│
├── filter_plugins/
│   └── aci_filters.py            # Custom Jinja2 filters for ACI
│
├── templates/
│   ├── f5_as3_declaration.j2     # AS3 JSON declaration template
│   └── pa_config_snippet.j2
│
├── vault/
│   └── secrets.yml               # ansible-vault encrypted
│
└── ansible.cfg
```

### Inventory (hosts.yml)

```yaml
all:
  children:
    aci:
      hosts:
        apic1:
          ansible_host: 192.168.100.1
          ansible_user: admin
          ansible_httpapi_use_ssl: true
          ansible_httpapi_validate_certs: false
          ansible_network_os: cisco.aci.aci
          ansible_connection: ansible.netcommon.httpapi
        apic2:
          ansible_host: 192.168.100.2
        apic3:
          ansible_host: 192.168.100.3

    paloalto:
      hosts:
        pa5400-a:
          ansible_host: 192.168.100.10
          panos_provider:
            ip_address: 192.168.100.10
            username: "{{ vault_pa_username }}"
            password: "{{ vault_pa_password }}"
        pa5400-b:
          ansible_host: 192.168.100.11
          panos_provider:
            ip_address: 192.168.100.11
            username: "{{ vault_pa_username }}"
            password: "{{ vault_pa_password }}"

    f5:
      hosts:
        f5-active:
          ansible_host: 192.168.100.20
          f5_server: 192.168.100.20
          f5_user: "{{ vault_f5_username }}"
          f5_password: "{{ vault_f5_password }}"
          f5_validate_certs: false
        f5-standby:
          ansible_host: 192.168.100.21
```

### Key Playbook: ACI Service Graph (03_aci_service_graph.yml)

```yaml
---
- name: Deploy ACI L4-L7 Service Graph for DMZ
  hosts: apic1
  gather_facts: false
  connection: ansible.netcommon.httpapi
  collections:
    - cisco.aci

  vars:
    tenant: TN_DMZ
    service_graph: SGT_INBOUND_CHAIN

  tasks:
    - name: Create L4-L7 Logical Device - Palo Alto
      cisco.aci.aci_rest:
        path: /api/mo/uni/tn-{{ tenant }}.json
        method: post
        content:
          vnsLDevVip:
            attributes:
              name: PA5400_HA_CLUSTER
              devtype: PHYSICAL
              svcType: FW
              contextAware: single-Context
              managed: "no"
              funcType: GoTo
            children:
              - vnsCDev:
                  attributes:
                    name: CDev_PA_A
                  children:
                    - vnsCIf:
                        attributes:
                          name: outside
                    - vnsCIf:
                        attributes:
                          name: inside
              - vnsCDev:
                  attributes:
                    name: CDev_PA_B
                  children:
                    - vnsCIf:
                        attributes:
                          name: outside
                    - vnsCIf:
                        attributes:
                          name: inside
              - vnsLIf:
                  attributes:
                    name: outside
                    encap: vlan-100
              - vnsLIf:
                  attributes:
                    name: inside
                    encap: vlan-200

    - name: Create L4-L7 Logical Device - F5 BIG-IP
      cisco.aci.aci_rest:
        path: /api/mo/uni/tn-{{ tenant }}.json
        method: post
        content:
          vnsLDevVip:
            attributes:
              name: F5_BIGIP_HA_CLUSTER
              devtype: PHYSICAL
              svcType: ADC
              contextAware: single-Context
              managed: "no"
              funcType: GoTo
            children:
              - vnsCDev:
                  attributes:
                    name: CDev_F5_ACTIVE
                  children:
                    - vnsCIf:
                        attributes:
                          name: external
                    - vnsCIf:
                        attributes:
                          name: internal
              - vnsCDev:
                  attributes:
                    name: CDev_F5_STANDBY
                  children:
                    - vnsCIf:
                        attributes:
                          name: external
                    - vnsCIf:
                        attributes:
                          name: internal
              - vnsLIf:
                  attributes:
                    name: external
                    encap: vlan-200
              - vnsLIf:
                  attributes:
                    name: internal
                    encap: vlan-210

    - name: Create Service Graph Template
      cisco.aci.aci_rest:
        path: /api/mo/uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}.json
        method: post
        content:
          vnsAbsGraph:
            attributes:
              name: "{{ service_graph }}"
              uiTemplateType: UNMANAGED
            children:
              - vnsAbsTermNodeCon:
                  attributes:
                    name: T1
                  children:
                    - vnsAbsTermConn:
                        attributes:
                          name: C1
              - vnsAbsTermNodeProv:
                  attributes:
                    name: T2
                  children:
                    - vnsAbsTermConn:
                        attributes:
                          name: C2
              - vnsAbsNode:
                  attributes:
                    name: PA5400_FW
                    funcType: GoTo
                    funcTemplateType: FW_ROUTED
                    managed: "no"
                  children:
                    - vnsAbsFuncConn:
                        attributes:
                          name: consumer
                          attNotify: "no"
                    - vnsAbsFuncConn:
                        attributes:
                          name: provider
                          attNotify: "no"
              - vnsAbsNode:
                  attributes:
                    name: F5_LB
                    funcType: GoTo
                    funcTemplateType: ADC_ONE_ARM
                    managed: "no"
                  children:
                    - vnsAbsFuncConn:
                        attributes:
                          name: consumer
                          attNotify: "no"
                    - vnsAbsFuncConn:
                        attributes:
                          name: provider
                          attNotify: "no"
              - vnsAbsConnection:
                  attributes:
                    name: C1
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsTermNodeCon-T1/AbsTConn"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsNode-PA5400_FW/AbsFConn-consumer"
              - vnsAbsConnection:
                  attributes:
                    name: C2
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsNode-PA5400_FW/AbsFConn-provider"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsNode-F5_LB/AbsFConn-consumer"
              - vnsAbsConnection:
                  attributes:
                    name: C3
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsNode-F5_LB/AbsFConn-provider"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ tenant }}/AbsGraph-{{ service_graph }}/AbsTermNodeProv-T2/AbsTConn"

    - name: Create PBR Policy for Palo Alto Outside
      cisco.aci.aci_rest:
        path: /api/mo/uni/tn-{{ tenant }}.json
        method: post
        content:
          vnsSvcRedirectPol:
            attributes:
              name: PBR_TO_PA_OUTSIDE
              thresholdEnable: "yes"
              programLocalPodOnly: "no"
              resilientHashEnabled: "yes"
            children:
              - vnsRedirectDest:
                  attributes:
                    ip: "10.1.1.10"
                    mac: "00:50:56:XX:XX:XX"
              - vnsRedirectDest:
                  attributes:
                    ip: "10.1.1.11"
                    mac: "00:50:56:XX:XX:XY"

    - name: Create PBR Policy for PA Inside to F5 External
      cisco.aci.aci_rest:
        path: /api/mo/uni/tn-{{ tenant }}.json
        method: post
        content:
          vnsSvcRedirectPol:
            attributes:
              name: PBR_PA_INSIDE_TO_F5_EXTERNAL
              thresholdEnable: "yes"
              resilientHashEnabled: "yes"
            children:
              - vnsRedirectDest:
                  attributes:
                    ip: "10.1.2.10"
                    mac: "00:50:56:YY:YY:YY"
```

### Key Playbook: Palo Alto Security Policies (06_paloalto_policies.yml)

```yaml
---
- name: Configure Palo Alto PA-5400 Security and Decryption Policies
  hosts: pa5400-a
  gather_facts: false
  collections:
    - paloaltonetworks.panos

  vars:
    provider: "{{ panos_provider }}"

  tasks:
    - name: Configure SSL Decryption Certificate
      paloaltonetworks.panos.panos_import:
        provider: "{{ provider }}"
        category: certificate
        certificate_name: wildcard_dmz
        format: pem
        filename: "{{ playbook_dir }}/../vault/wildcard.dmz.example.com.pem"
        passphrase: "{{ vault_cert_passphrase }}"

    - name: Create Decryption Profile
      paloaltonetworks.panos.panos_decryption_rule:
        provider: "{{ provider }}"
        name: DECRYPT_INBOUND_HTTPS
        source_zones: ['OUTSIDE']
        destination_zones: ['DMZ']
        destination_addresses: ['203.0.113.10', '203.0.113.11']
        services: ['service-https']
        action: decrypt
        ssl_decrypt_type: ssl-inbound-inspection
        ssl_certificate: wildcard_dmz
        decryption_profile: SSLPROF_INBOUND_WEB

    - name: Create Anti-Virus Security Profile
      paloaltonetworks.panos.panos_security_profile_group:
        provider: "{{ provider }}"
        name: PG_STRICT_INBOUND
        virus: AV_STRICT
        spyware: AS_STRICT
        vulnerability: VP_STRICT
        wildfire_analysis: WF_INLINE
        file_blocking: FB_BLOCK_EXECUTABLES
        url_filtering: URL_DMZ_RESTRICTED

    - name: Create Security Policy - Allow Inbound HTTPS
      paloaltonetworks.panos.panos_security_rule:
        provider: "{{ provider }}"
        rule_name: ALLOW_INBOUND_HTTPS
        source_zone: ['OUTSIDE']
        destination_zone: ['DMZ']
        source_ip: ['any']
        destination_ip: ['10.1.2.10', '10.1.2.11']
        application: ['ssl', 'web-browsing']
        service: ['service-https', 'tcp-8443']
        action: allow
        group_profile: PG_STRICT_INBOUND
        log_start: true
        log_end: true
        log_setting: LOG_PROFILE_SIEM
        location: top

    - name: Create Security Policy - Deny DMZ to Internet
      paloaltonetworks.panos.panos_security_rule:
        provider: "{{ provider }}"
        rule_name: DENY_OUTBOUND_DMZ_TO_INTERNET
        source_zone: ['DMZ']
        destination_zone: ['OUTSIDE']
        source_ip: ['any']
        destination_ip: ['any']
        application: ['any']
        service: ['any']
        action: deny
        log_start: true
        log_end: true
        log_setting: LOG_PROFILE_SIEM

    - name: Create NAT Rule - DNAT Inbound Web
      paloaltonetworks.panos.panos_nat_rule:
        provider: "{{ provider }}"
        rule_name: DNAT_INBOUND_WEB
        source_zone: ['OUTSIDE']
        destination_zone: ['OUTSIDE']
        source_ip: ['any']
        destination_ip: ['203.0.113.10']
        service: service-https
        dnat_address: '10.1.2.10'
        dnat_port: '443'

    - name: Commit configuration
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ provider }}"
```

### Key Playbook: F5 Application Services via AS3 (09_f5_app_services.yml)

```yaml
---
- name: Deploy F5 BIG-IP Application Services
  hosts: f5-active
  gather_facts: false
  collections:
    - f5networks.f5_modules

  tasks:
    - name: Deploy AS3 Declaration for DMZ Web Services
      f5networks.f5_modules.bigip_as3_deploy:
        content: "{{ lookup('template', '../templates/f5_as3_declaration.j2') }}"
      delegate_to: localhost

# templates/f5_as3_declaration.j2 generates:
# {
#   "class": "AS3",
#   "action": "deploy",
#   "declaration": {
#     "class": "ADC",
#     "schemaVersion": "3.40.0",
#     "DMZ_Tenant": {
#       "class": "Tenant",
#       "DMZ_Web_App": {
#         "class": "Application",
#         "template": "https",
#         "serviceMain": {
#           "class": "Service_HTTP",
#           "virtualAddresses": ["10.1.2.10"],
#           "virtualPort": 443,
#           "pool": "pool_web_servers",
#           "profileHTTP": { "use": "http_xff_profile" },
#           "snat": { "use": "snat_pool_web" },
#           "iRules": [{ "use": "irule_security_headers" }]
#         },
#         "pool_web_servers": {
#           "class": "Pool",
#           "loadBalancingMode": "least-connections-member",
#           "monitors": [{ "use": "mon_http_web" }],
#           "members": [{
#             "servicePort": 8080,
#             "serverAddresses": [
#               "10.1.10.101", "10.1.10.102",
#               "10.1.10.103", "10.1.10.104"
#             ]
#           }]
#         },
#         ...
#       }
#     }
#   }
# }
```

### Validation Playbook (10_validation.yml)

```yaml
---
- name: Validate End-to-End DMZ Deployment
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Verify ACI Service Graph is deployed
      cisco.aci.aci_rest:
        host: "{{ apic_host }}"
        username: "{{ apic_user }}"
        password: "{{ apic_pass }}"
        path: /api/class/vnsGraphInst.json?query-target-filter=eq(vnsGraphInst.graphName,"SGT_INBOUND_CHAIN")
        method: get
      register: sg_result
      failed_when: sg_result.imdata | length == 0

    - name: Verify PA-5400 HA status
      paloaltonetworks.panos.panos_op:
        provider: "{{ pa_provider }}"
        cmd: show high-availability state
      register: ha_state
      failed_when: "'active' not in ha_state.stdout"

    - name: Verify F5 pool member health
      f5networks.f5_modules.bigip_pool:
        name: POOL_WEB_SERVERS
        state: present
      register: pool_state

    - name: Test HTTPS connectivity through chain
      ansible.builtin.uri:
        url: "https://{{ public_vip }}"
        validate_certs: false
        status_code: [200, 301, 302]
        timeout: 10
      register: https_test
      retries: 3
      delay: 5

    - name: Verify PBR redirect health on ACI
      cisco.aci.aci_rest:
        host: "{{ apic_host }}"
        username: "{{ apic_user }}"
        password: "{{ apic_pass }}"
        path: /api/class/vnsSvcRedirectPol.json
        method: get
      register: pbr_health

    - name: Generate validation report
      ansible.builtin.template:
        src: validation_report.j2
        dest: /tmp/dmz_validation_{{ ansible_date_time.date }}.html
```

---

## 10. Monitoring and Observability

### Monitoring Stack Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Monitoring Plane                      │
│                                                         │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌───────┐ │
│  │Prometheus│  │Elasticsearch│ │ Grafana  │  │PagerDuty│ │
│  │ + Thanos │  │  + Kibana  │  │Dashboard │  │Alerting│ │
│  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └───┬───┘ │
│       │              │              │             │      │
│  ┌────┴─────┐  ┌─────┴─────┐                     │      │
│  │Telegraf/ │  │ Filebeat/ │                     │      │
│  │SNMP Exp. │  │ Syslog-ng │                     │      │
│  └──────────┘  └───────────┘                     │      │
└─────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    ┌────┴────┐   ┌─────┴────┐   ┌────┴────┐
    │ACI APIC │   │PA-5400   │   │F5 BIG-IP│
    │(SNMP,   │   │(Syslog,  │   │(SNMP,   │
    │ REST API│   │ SNMP,    │   │ iHealth, │
    │ Health  │   │ Cortex   │   │ REST,   │
    │ Scores) │   │ Data Lake│   │ AVR)    │
    └─────────┘   └──────────┘   └─────────┘
```

### ACI Fabric Monitoring

| Metric | Collection Method | Tool | Alert Threshold |
|---|---|---|---|
| Service Graph health | ACI REST API `/api/class/vnsGraphInst` | Prometheus ACI exporter | State != formed |
| PBR redirect health | ACI REST API `/api/class/vnsSvcRedirectHealthGroup` | Prometheus ACI exporter | Any destination down |
| EPG health score | ACI REST API `/api/class/fvAEPg` | Prometheus ACI exporter | Health < 90 |
| Contract hit counts | ACI REST API `/api/class/actrlRule` | Prometheus ACI exporter | Sudden drop (anomaly) |
| Fabric interface utilization | SNMP IF-MIB | Telegraf SNMP input | > 80% sustained 5min |
| Leaf CPU/memory | SNMP (HOST-RESOURCES-MIB) | Telegraf SNMP input | CPU > 80%, Mem > 85% |
| Fault summary | ACI REST API `/api/class/faultSummary` | Custom script | Any critical fault |
| TCAM usage | ACI REST API `/api/class/eqptcapacityPolUsage5min` | Prometheus | > 80% policy TCAM |

### Palo Alto PA-5400 Monitoring

| Metric | Collection Method | Alert Threshold |
|---|---|---|
| HA state | SNMP + PAN-OS API `show ha state` | State change (active->passive) |
| Session count | SNMP panSessionUtilization | > 80% of max |
| Session rate | PAN-OS API `show session info` | > 80% of rated CPS |
| SSL Decryption throughput | SNMP custom OID | > 70% of hardware capacity |
| SSL Decryption failures | Syslog (type=decryption, action=deny) | > 100/min |
| Threat detections | Syslog (type=threat) | Any critical severity |
| CPU (mgmt/data plane) | SNMP panCPU | Data plane > 80% |
| Memory | SNMP panSessionMemory | > 80% |
| Interface status | SNMP IF-MIB | Link down |
| Policy deny count | Syslog (action=deny) | Anomalous spike |
| WildFire submissions | PAN-OS API | Verdict=malicious |
| Certificate expiry | PAN-OS API `show ssl-decrypt certificate` | < 30 days |
| GlobalProtect (if used) | SNMP | N/A for DMZ |

#### PA-5400 Syslog Configuration

```
Syslog Server Profile: SYSLOG_TO_SIEM
  Server:     siem.internal.example.com:514
  Transport:  TCP (TLS)
  Format:     BSD
  Facility:   LOG_LOCAL5

Log Forwarding Profile: LOG_PROFILE_SIEM
  Traffic:    syslog SYSLOG_TO_SIEM (all)
  Threat:     syslog SYSLOG_TO_SIEM (informational+)
  WildFire:   syslog SYSLOG_TO_SIEM (all)
  URL:        syslog SYSLOG_TO_SIEM (all)
  Decryption: syslog SYSLOG_TO_SIEM (all)
  Auth:       syslog SYSLOG_TO_SIEM (all)
```

### F5 BIG-IP i5800 Monitoring

| Metric | Collection Method | Alert Threshold |
|---|---|---|
| Virtual server availability | SNMP ltmVsStatusAvailState | Not green |
| Pool member health | SNMP ltmPoolMemberMonitorStatus | Any member down |
| Active connections | SNMP sysStatClientCurConns | > 80% of connection limit |
| Throughput (in/out) | SNMP IF-MIB | > 80% of interface capacity |
| CPU utilization | SNMP sysMultiHostCpuUsageRatio | > 80% |
| Memory utilization | SNMP sysMultiHostMemoryUsed | > 85% |
| TMM CPU | SNMP sysGlobalTmmStatCpu | > 80% |
| SSL TPS | SNMP sysClientsslStatTotNativeConns | > 70% of rated |
| HTTP request rate | AVR or iRule HSL logging | Anomalous spike/drop |
| HTTP error rate (5xx) | AVR or iRule HSL logging | > 5% of total |
| Response time | iRule HSL (server-side latency) | P99 > 2s |
| HA failover events | SNMP sysCmFailoverStatus | Any state change |
| Certificate expiry | F5 REST API | < 30 days |
| Disk usage | SNMP HOST-RESOURCES-MIB | > 85% |

### Grafana Dashboard Layout

```
Dashboard: DMZ Service Chain Overview
├── Row 1: Health Summary
│   ├── Panel: Service Graph Status (ACI)         [status indicator]
│   ├── Panel: PA-5400 HA State                    [status indicator]
│   ├── Panel: F5 HA State                         [status indicator]
│   └── Panel: Overall Chain Health                [traffic light]
│
├── Row 2: Traffic Flow
│   ├── Panel: Inbound Requests/sec               [time series]
│   ├── Panel: PA Throughput (encrypted/decrypted) [time series]
│   ├── Panel: F5 Active Connections               [time series]
│   └── Panel: Backend Response Time (P50/P95/P99) [time series]
│
├── Row 3: Security
│   ├── Panel: Threats Detected (by type)          [bar chart]
│   ├── Panel: SSL Decryption Stats                [time series]
│   ├── Panel: Denied Flows (by policy rule)       [table]
│   └── Panel: Top Blocked Source IPs              [table]
│
├── Row 4: ACI Microsegmentation
│   ├── Panel: Contract Hit Counts (per contract)  [time series]
│   ├── Panel: Denied Flows by EPG Pair            [heatmap]
│   ├── Panel: EPG Health Scores                   [gauge]
│   └── Panel: PBR Redirect Health                 [status indicator]
│
└── Row 5: Capacity
    ├── Panel: PA Session Utilization %            [gauge]
    ├── Panel: F5 CPU / TMM Utilization %          [gauge]
    ├── Panel: ACI Leaf TCAM Usage %               [gauge]
    └── Panel: SSL Decryption Capacity %           [gauge]
```

### Alerting Rules

```yaml
# Prometheus alerting rules (alerts.yml)
groups:
  - name: dmz_service_chain
    rules:
      - alert: ServiceGraphNotFormed
        expr: aci_service_graph_state{name="SGT_INBOUND_CHAIN"} != 1
        for: 2m
        labels:
          severity: critical
          team: network
        annotations:
          summary: "ACI Service Graph SGT_INBOUND_CHAIN is not in formed state"

      - alert: PaloAltoHAFailover
        expr: changes(panos_ha_state[5m]) > 0
        labels:
          severity: critical
          team: security
        annotations:
          summary: "PA-5400 HA failover detected"

      - alert: F5PoolDegraded
        expr: f5_pool_active_members{pool="POOL_WEB_SERVERS"} < 2
        for: 1m
        labels:
          severity: warning
          team: app
        annotations:
          summary: "F5 pool POOL_WEB_SERVERS has fewer than 2 active members"

      - alert: SSLDecryptionCapacityHigh
        expr: panos_ssl_decrypt_utilization > 70
        for: 5m
        labels:
          severity: warning
          team: security
        annotations:
          summary: "PA-5400 SSL decryption utilization above 70%"

      - alert: PBRRedirectDown
        expr: aci_pbr_redirect_health{policy="PBR_TO_PA_OUTSIDE"} == 0
        for: 30s
        labels:
          severity: critical
          team: network
        annotations:
          summary: "ACI PBR redirect to PA-5400 outside is down"

      - alert: HighHTTPErrorRate
        expr: rate(f5_http_responses_5xx[5m]) / rate(f5_http_responses_total[5m]) > 0.05
        for: 3m
        labels:
          severity: warning
          team: app
        annotations:
          summary: "HTTP 5xx error rate exceeds 5%"

      - alert: ContractDenySpike
        expr: rate(aci_contract_deny_count[5m]) > 1000
        for: 2m
        labels:
          severity: warning
          team: security
        annotations:
          summary: "Unusual spike in ACI contract deny count"
```

---

## 11. Failure Scenarios and Resilience

### Failure Matrix

| Failure | Detection | Automatic Recovery | Impact |
|---|---|---|---|
| PA-5400 Node A failure | HA heartbeat loss (1s) | Node B becomes active (failover < 3s) | Existing sessions synced, minimal packet loss |
| F5 Active failure | F5 HA heartbeat loss | Standby takes over floating IPs (< 5s) | Mirrored connections persist, new connections unaffected |
| Single leaf switch failure | ACI vPC peer keepalive | Traffic shifts to surviving leaf in vPC pair | Sub-second with vPC fast convergence |
| PA outside interface down | PA link monitoring | PA HA failover to peer | < 3s failover |
| PBR destination unreachable | ACI health tracking (IP-SLA) | PBR fails to backup destination | Automatic with threshold-based health tracking |
| ACI spine failure | Fabric IS-IS reconvergence | Traffic reroutes via remaining spines | < 1s for dual-spine |
| SSL certificate expiry | Monitoring alert at 30 days | Manual renewal via Ansible | No impact if renewed before expiry |
| OpenStack compute node failure | F5 health monitor (5s interval) | Pool member marked down after 3 failed checks (16s) | Connections drained to healthy members |
| Complete PA-5400 pair failure | ACI PBR health tracking | PBR bypass (if configured) or total outage | Service outage unless bypass enabled |
| APIC cluster node failure | APIC cluster health | Remaining 2 APICs form quorum | No data-plane impact |

### PBR Health Tracking Detail

```
ACI PBR Health Group: HG_PA_OUTSIDE
  SLA Monitoring Policy:
    Frequency:          5 seconds
    Retry Count:        3
    Retry Delay:        2 seconds
    Threshold Down:     3 consecutive failures (15s to mark down)
    Threshold Up:       1 success (5s to mark up)
    Protocol:           TCP (port 443 for L7 health) or ICMP

  Redirect Destinations:
    Primary:   10.1.1.10 (PA Active outside float)
    Secondary: 10.1.1.11 (PA Standby outside)

  Behavior on All-Down:
    Option A: Drop traffic (secure default -- recommended)
    Option B: Bypass service node (permits uninspected traffic -- NOT recommended for DMZ)
```

---

## Appendix A: IP Address Allocation Summary

| Subnet | CIDR | Gateway (ACI BD) | Purpose |
|---|---|---|---|
| 10.1.1.0/24 | BD_OUTSIDE | 10.1.1.1 | PA outside leg / Internet edge |
| 10.1.2.0/24 | BD_PA_TO_F5 | 10.1.2.1 | PA inside to F5 external transit |
| 10.1.10.0/24 | BD_DMZ_WEB | 10.1.10.1 | Web-tier VMs |
| 10.1.11.0/24 | BD_DMZ_API | 10.1.11.1 | API gateway VMs |
| 10.1.12.0/24 | BD_DMZ_APP | 10.1.12.1 | Application-tier VMs |
| 10.1.13.0/24 | BD_DMZ_DB | 10.1.13.1 | Database-tier VMs |
| 192.168.100.0/24 | OOB Mgmt | 192.168.100.1 | Out-of-band management |
| 203.0.113.0/28 | Public VIPs | N/A | Internet-facing VIPs (NAT on PA) |

## Appendix B: VLAN Allocation

| VLAN ID | Name | Usage |
|---|---|---|
| 100 | VLAN_PA_OUTSIDE | PA-5400 outside leg to ACI |
| 200 | VLAN_PA_INSIDE_F5_EXT | PA inside / F5 external transit |
| 210 | VLAN_F5_INTERNAL | F5 internal to compute fabric |
| 2000-2199 | Dynamic pool | OpenStack VM EPG encapsulation |
| 4094 | F5_HA | F5 HA VLAN (isolated) |

## Appendix C: Ansible Execution Order

```bash
# Full deployment (sequential -- each step depends on the previous)
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml

# site.yml includes:
#   1. 01_aci_fabric.yml         -- ACI tenant, VRF, BDs, EPGs
#   2. 02_aci_l4l7_devices.yml  -- Register PA and F5 as L4-L7 devices
#   3. 03_aci_service_graph.yml -- Service graph + PBR policies
#   4. 04_paloalto_base.yml     -- PA interfaces, zones, routing
#   5. 05_paloalto_ha.yml       -- PA HA configuration
#   6. 06_paloalto_policies.yml -- PA security, NAT, SSL decryption
#   7. 07_f5_base.yml           -- F5 VLANs, self-IPs, routing
#   8. 08_f5_ha.yml             -- F5 HA / device trust
#   9. 09_f5_app_services.yml   -- F5 virtual servers, pools (AS3)
#  10. 10_validation.yml        -- End-to-end verification

# Day-2 operations (targeted)
ansible-playbook -i inventories/production/hosts.yml playbooks/09_f5_app_services.yml  # Add pool members
ansible-playbook -i inventories/production/hosts.yml playbooks/06_paloalto_policies.yml # Update policies
ansible-playbook -i inventories/production/hosts.yml playbooks/10_validation.yml        # Re-validate
```
