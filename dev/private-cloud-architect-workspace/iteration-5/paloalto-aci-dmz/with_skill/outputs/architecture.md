# DMZ Architecture: Cisco ACI Fabric with Palo Alto PA-5400 and F5 BIG-IP i5800

## 1. Architecture Overview

This design places a Palo Alto PA-5400 HA pair and an F5 BIG-IP i5800 HA pair into a Cisco ACI fabric using Service Graph chaining. The DMZ hosts public-facing OpenStack VMs (web servers and API gateways). Inbound HTTPS traffic is SSL-decrypted by the Palo Alto for threat inspection before being forwarded to the F5 for load balancing. Internal east-west traffic between application tiers is microsegmented using ACI Contracts.

### Design Principles

- Zero-trust: all inter-EPG traffic denied by default (ACI whitelist model); every flow requires an explicit Contract
- Defense in depth: perimeter NGFW (Palo Alto) plus ACI Contract microsegmentation plus application-level controls
- HA everywhere: Palo Alto active/passive HA, F5 active/standby with config sync, APIC 3-node cluster
- Infrastructure as Code: all configuration managed via Ansible (cisco.aci, paloaltonetworks.panos, f5networks.f5_modules collections)
- SSL visibility: inbound HTTPS decrypted at Palo Alto for threat inspection, re-encrypted to F5, then offloaded at F5 for backend distribution

---

## 2. Full Traffic Path (North-South Inbound)

```
Client (Internet)
       |
       | HTTPS (TCP/443)
       v
  [ISP / Border Router]
       |
       | BGP peering
       v
  [ACI L3Out - External Routed Network]
       |
       | ACI Policy-Based Redirect (PBR)
       v
  [Palo Alto PA-5400 HA Pair]          <-- Service Graph Node 1
       | Zone: Untrust -> DMZ
       | SSL Decryption (inbound inspection)
       | App-ID, Threat Prevention (IPS, AV, Anti-Spyware, WildFire)
       | URL Filtering
       | Re-encrypt traffic (TLS to F5)
       |
       | ACI PBR steers to next node
       v
  [F5 BIG-IP i5800 HA Pair]            <-- Service Graph Node 2
       | SSL offload (terminates TLS from Palo Alto)
       | Virtual Server: vs_dmz_https (0.0.0.0/0:443)
       | iRule: X-Forwarded-For insertion, Host-based routing
       | Pool selection based on Host header:
       |   - pool_web_servers (web frontend VMs)
       |   - pool_api_gateways (API gateway VMs)
       | Health monitors: HTTP GET /health, interval 5s, timeout 16s
       |
       | HTTP/HTTPS to backend
       v
  [ACI Web-EPG / API-EPG]
       | OpenStack VMs (web servers, API gateways)
       |
       | ACI Contract (microsegmented)
       v
  [ACI App-EPG]
       | Application tier VMs
       |
       | ACI Contract (microsegmented)
       v
  [ACI DB-EPG]
       | Database tier VMs
```

### Return Traffic Path

Return traffic follows the same Service Graph chain in reverse. ACI PBR ensures symmetric routing through the Palo Alto and F5 in both directions, which is critical for stateful firewall inspection and connection tracking.

---

## 3. ACI Tenant and Object Model

### Tenant Structure

```
Tenant: TN_DMZ_Production
  |
  +-- VRF: VRF_DMZ
  |     |
  |     +-- Bridge Domain: BD_External
  |     |     Subnet: 203.0.113.0/24 (public, advertised via L3Out)
  |     |
  |     +-- Bridge Domain: BD_DMZ_Web
  |     |     Subnet: 10.10.1.0/24 (gateway on ACI fabric)
  |     |
  |     +-- Bridge Domain: BD_DMZ_API
  |     |     Subnet: 10.10.2.0/24
  |     |
  |     +-- Bridge Domain: BD_App
  |     |     Subnet: 10.10.10.0/24
  |     |
  |     +-- Bridge Domain: BD_DB
  |           Subnet: 10.10.20.0/24
  |
  +-- Application Profile: AP_DMZ
  |     |
  |     +-- EPG: EPG_External (bound to L3Out, External EPG)
  |     +-- EPG: EPG_Web (VMM domain: OpenStack, BD: BD_DMZ_Web)
  |     +-- EPG: EPG_API (VMM domain: OpenStack, BD: BD_DMZ_API)
  |     +-- EPG: EPG_App (VMM domain: OpenStack, BD: BD_App)
  |     +-- EPG: EPG_DB  (VMM domain: OpenStack, BD: BD_DB)
  |
  +-- L3Out: L3OUT_Internet
  |     Protocol: BGP
  |     External EPG: EPG_External (0.0.0.0/0)
  |     Advertised subnets: 203.0.113.0/24
  |
  +-- Service Graph Template: SGT_PaloAlto_F5
  |     Node 1: Palo Alto PA-5400 (GoTo mode, routed, PBR)
  |     Node 2: F5 BIG-IP i5800 (GoTo mode, routed)
  |
  +-- Device Selection Policies
  +-- L4-L7 Device Packages (Palo Alto, F5)
```

### L3Out Configuration

| Parameter | Value |
|---|---|
| Name | L3OUT_Internet |
| VRF | VRF_DMZ |
| Protocol | BGP (AS 65001 local, ISP AS varies) |
| Node Profile | Leaf-101, Leaf-102 (border leaves) |
| Interface Profile | eth1/48 on each border leaf (routed, point-to-point) |
| External EPG | EPG_External, subnets: 0.0.0.0/0 |
| Advertised subnets | 203.0.113.0/24 (public VIP range) |
| Route Control | Export route-map for public prefixes only |

---

## 4. ACI Service Graph Design

### L4-L7 Device Registration

#### Palo Alto PA-5400 HA Pair

| Parameter | Value |
|---|---|
| Device Type | Physical, Routed mode |
| Device Package | Palo Alto PANOS device package (imported to APIC) |
| Cluster Interface | Active: PA-5400-A, Standby: PA-5400-B |
| Management IP | 10.250.1.10 (A), 10.250.1.11 (B) |
| Consumer Connector (outside) | VLAN 100, interface Ethernet1/1, IP 10.10.100.1/24 |
| Provider Connector (inside) | VLAN 101, interface Ethernet1/2, IP 10.10.101.1/24 |
| HA3 (packet forwarding) | Dedicated link between HA peers |
| Device Selection | Primary: PA-5400-A, Secondary: PA-5400-B |

#### F5 BIG-IP i5800 HA Pair

| Parameter | Value |
|---|---|
| Device Type | Physical, Routed mode |
| Device Package | F5 BIG-IP device package (imported to APIC) |
| Cluster Interface | Active: F5-i5800-A, Standby: F5-i5800-B |
| Management IP | 10.250.1.20 (A), 10.250.1.21 (B) |
| Consumer Connector (outside/client-side) | VLAN 200, interface 1.1, Self IP 10.10.200.1/24 |
| Provider Connector (inside/server-side) | VLAN 201, interface 1.2, Self IP 10.10.201.1/24 |
| Floating Self IP | 10.10.200.3 (outside), 10.10.201.3 (inside) |
| Device Selection | Primary: F5-i5800-A, Secondary: F5-i5800-B |

### Service Graph Template: SGT_PaloAlto_F5

```
                    Service Graph Chain
  [Consumer]                                      [Provider]
  EPG_External ──> [Node1: PaloAlto] ──> [Node2: F5_BIG-IP] ──> EPG_Web / EPG_API
                     (Routed, PBR)        (Routed)
```

- **Graph Type**: Two-node chain
- **Node 1 (Palo Alto)**: Function = GoTo, Routing = Routed, PBR enabled
- **Node 2 (F5)**: Function = GoTo, Routing = Routed
- **Connector VLANs**: ACI allocates VLANs from a static VLAN pool for each connector (consumer-side and provider-side of each node)

### Policy-Based Redirect (PBR)

PBR policies on ACI redirect traffic matching the Contract between EPG_External and EPG_Web/EPG_API through the Service Graph nodes. The PBR policy specifies:

| PBR Policy | Target | Health Check |
|---|---|---|
| PBR_PaloAlto_Consumer | PA-5400 outside interface (10.10.100.1) | ICMP or TCP SYN to PA mgmt |
| PBR_PaloAlto_Provider | PA-5400 inside interface (10.10.101.1) | ICMP or TCP SYN to PA mgmt |
| PBR_F5_Consumer | F5 outside floating self IP (10.10.200.3) | TCP probe on port 443 |
| PBR_F5_Provider | F5 inside floating self IP (10.10.201.3) | TCP probe on port 80 |

PBR health checks (IP SLA or redirect health group) ensure that if the Palo Alto or F5 becomes unreachable, ACI can take corrective action (mark the redirect target as down, potentially trigger failover).

### Contract and Subject Binding

```
Contract: CON_Internet_to_DMZ
  Subject: SUB_HTTPS
    Filter: FLT_HTTPS (TCP dst 443)
    Service Graph: SGT_PaloAlto_F5
    Direction: Consumer-to-Provider
  Subject: SUB_HTTP_Redirect
    Filter: FLT_HTTP (TCP dst 80)
    Service Graph: SGT_PaloAlto_F5
    Direction: Consumer-to-Provider

Contract Consumers: EPG_External (via L3Out)
Contract Providers: EPG_Web, EPG_API
```

---

## 5. Inter-Tier Microsegmentation (ACI Contracts)

All east-west traffic between application tiers is controlled by ACI Contracts. No Service Graph insertion is needed for east-west flows -- ACI enforces these as policy CAM entries on the leaf switches.

### Contract Definitions

| Contract | Consumer EPG | Provider EPG | Filters | Purpose |
|---|---|---|---|---|
| CON_Web_to_App | EPG_Web | EPG_App | TCP/8080, TCP/8443 | Web frontends call application tier APIs |
| CON_API_to_App | EPG_API | EPG_App | TCP/8080, TCP/8443 | API gateways call application tier |
| CON_App_to_DB | EPG_App | EPG_DB | TCP/5432 (PostgreSQL), TCP/3306 (MySQL) | App tier queries databases |
| CON_Web_to_DB | -- | -- | (No contract = denied) | Web tier has no direct DB access |
| CON_API_to_DB | -- | -- | (No contract = denied) | API tier has no direct DB access |
| CON_Mgmt_Access | EPG_Mgmt | EPG_Web, EPG_API, EPG_App, EPG_DB | TCP/22 (SSH), TCP/443 (HTTPS mgmt) | Management access for operations |

### Filter Definitions

| Filter Name | Entries |
|---|---|
| FLT_HTTPS | EtherType IP, Protocol TCP, Dst Port 443 |
| FLT_HTTP | EtherType IP, Protocol TCP, Dst Port 80 |
| FLT_App_Ports | EtherType IP, Protocol TCP, Dst Port 8080; Dst Port 8443 |
| FLT_PostgreSQL | EtherType IP, Protocol TCP, Dst Port 5432 |
| FLT_MySQL | EtherType IP, Protocol TCP, Dst Port 3306 |
| FLT_SSH | EtherType IP, Protocol TCP, Dst Port 22 |
| FLT_ICMP | EtherType IP, Protocol ICMP |

### Microsegmentation Design Notes

- ACI operates as a whitelist model: no Contract between two EPGs means zero communication (implicit deny)
- EPG_Web and EPG_API cannot reach EPG_DB directly, enforcing a strict three-tier separation
- Preferred Group is NOT enabled on these EPGs to maintain explicit contract enforcement
- vzAny is not used; each inter-EPG relationship is defined with a specific contract
- Intra-EPG isolation can be enabled on EPG_DB to prevent lateral movement between database VMs if required

---

## 6. Palo Alto PA-5400 Configuration

### Zones

| Zone Name | Type | Interfaces | Purpose |
|---|---|---|---|
| Untrust | Layer 3 | ethernet1/1 (VLAN 100) | Outside / internet-facing (ACI consumer connector) |
| DMZ | Layer 3 | ethernet1/2 (VLAN 101) | Inside / toward F5 and DMZ servers (ACI provider connector) |
| Management | Management | MGT | Out-of-band management, Panorama connectivity |

### Network Interfaces

| Interface | Zone | IP Address | Description |
|---|---|---|---|
| ethernet1/1 | Untrust | 10.10.100.1/24 | Consumer-side from ACI Service Graph |
| ethernet1/2 | DMZ | 10.10.101.1/24 | Provider-side toward F5 |
| ethernet1/3 | -- | HA2 (keep-alive) | HA control link |
| ethernet1/4 | -- | HA3 (packet forwarding) | HA data link |
| MGT | Management | 10.250.1.10/24 | Management interface |

### Virtual Router

| Parameter | Value |
|---|---|
| Name | VR_DMZ |
| Interfaces | ethernet1/1, ethernet1/2 |
| Static route | 0.0.0.0/0 via 10.10.100.254 (ACI BD gateway) |
| Static route | 10.10.200.0/24 via 10.10.101.254 (toward F5 consumer side via ACI) |
| Static route | 10.10.1.0/24 via 10.10.101.254 (toward Web EPG via ACI) |
| Static route | 10.10.2.0/24 via 10.10.101.254 (toward API EPG via ACI) |

### SSL Decryption (Inbound Inspection)

This is the critical requirement: inbound HTTPS traffic must be decrypted at the Palo Alto before reaching the F5 so that the Palo Alto can perform App-ID, Content-ID (IPS, AV, Anti-Spyware), and WildFire analysis on the cleartext HTTP content.

#### SSL Decryption Mode: Inbound Inspection (SSL Inbound Inspection)

Unlike forward proxy decryption (used for outbound traffic), inbound inspection requires the server's private key to be imported to the Palo Alto so it can decrypt traffic destined for the web servers.

| Parameter | Value |
|---|---|
| Decryption Profile | PROF_Inbound_Decrypt |
| Mode | SSL Inbound Inspection |
| Certificate | Import the web server certificate + private key (e.g., wildcard cert for *.example.com) |
| Protocol Versions | TLS 1.2, TLS 1.3 minimum |
| Decryption Policy Rule Name | RULE_Decrypt_Inbound_HTTPS |
| Source Zone | Untrust |
| Destination Zone | DMZ |
| Destination Address | 203.0.113.0/24 (public VIPs) |
| Service | TCP/443 |
| Action | Decrypt (SSL Inbound Inspection) |
| Decryption Profile Settings | Block sessions with expired certs, block untrusted issuers, block sessions with unsupported cipher suites |

#### Decryption-Re-encryption Flow

1. Client sends TLS ClientHello to public VIP (203.0.113.x)
2. Palo Alto terminates the TLS session using the imported server certificate and private key
3. Palo Alto inspects the cleartext HTTP payload (App-ID identifies the application, Content-ID runs IPS/AV/Anti-Spyware/WildFire)
4. Palo Alto re-encrypts the traffic using TLS to the F5 consumer-side interface
5. F5 terminates TLS (SSL offload), applies load balancing logic
6. F5 sends HTTP (or HTTPS with re-encryption) to backend web server VMs

#### Decryption Considerations

- **Private key security**: The server private key stored on the Palo Alto must be protected. Use Panorama to manage certificate distribution. Consider HSM integration for PA-5400 if available.
- **TLS 1.3 inbound inspection**: TLS 1.3 with ephemeral Diffie-Hellman (ECDHE) does NOT support passive decryption. The Palo Alto must act as a proxy for TLS 1.3, which it supports in PAN-OS 10.x+. Ensure PAN-OS version supports TLS 1.3 inbound inspection.
- **Performance**: SSL decryption is CPU-intensive. The PA-5400 has dedicated hardware for SSL processing. Size appropriately for expected TLS session rate and throughput.
- **Certificate rotation**: Automate certificate updates on the Palo Alto when server certificates are renewed (integrate with cert-manager or manual Ansible workflow).

### Security Policies

| Rule # | Name | Source Zone | Dest Zone | Source | Destination | Application | Service | Action | Security Profiles |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Allow_Inbound_HTTPS | Untrust | DMZ | any | 203.0.113.0/24 | ssl, web-browsing | tcp/443 | Allow | AV, AS, VP, URL, WF, FB |
| 2 | Allow_Inbound_HTTP_Redirect | Untrust | DMZ | any | 203.0.113.0/24 | web-browsing | tcp/80 | Allow | AV, AS, VP, URL |
| 3 | Deny_Untrust_to_DMZ_All | Untrust | DMZ | any | any | any | any | Deny | -- (log) |
| 4 | Allow_DMZ_to_Untrust_Response | DMZ | Untrust | 10.10.101.0/24 | any | any | any | Allow | -- |
| 5 | Deny_All | any | any | any | any | any | any | Deny | -- (log) |

**Security Profile Group: SPG_DMZ_Inbound**

| Profile Type | Profile Name | Settings |
|---|---|---|
| Antivirus | AV_Strict | Block all threat severities, WildFire inline ML |
| Anti-Spyware | AS_Strict | Block C2, DNS sinkhole enabled |
| Vulnerability Protection | VP_Strict | Block critical/high/medium, alert on low |
| URL Filtering | URL_DMZ | Block malware, phishing, C2 categories |
| WildFire Analysis | WF_All | Forward PE, APK, PDF, MS Office, Linux ELF |
| File Blocking | FB_DMZ | Block batch, DLL, EXE, encrypted-zip uploads |

### DoS Protection

| Parameter | Value |
|---|---|
| Zone Protection Profile | ZP_Untrust |
| SYN Flood | SYN cookies, alarm rate 10000, activate 50000, maximum 500000 |
| UDP Flood | Alarm 10000, activate 50000, maximum 500000 |
| ICMP Flood | Alarm 1000, activate 5000, maximum 10000 |
| IP Fragmentation | Discard overlapping fragments |
| Reconnaissance Protection | Block TCP/UDP port scans, host sweeps |

### HA Configuration

| Parameter | Value |
|---|---|
| HA Mode | Active/Passive |
| Group ID | 1 |
| HA1 (control) | Management port or dedicated HA1 interface |
| HA2 (data) | ethernet1/3, IP 169.254.1.1/24 (A), 169.254.1.2/24 (B) |
| HA3 (packet forwarding) | ethernet1/4 |
| Preemption | Disabled (prevent flapping) |
| Failover timers | Hello interval: 1000ms, Heartbeat interval: 1000ms |
| Config Sync | Enabled (all device configuration synchronized) |
| Link Monitoring | ethernet1/1, ethernet1/2 (failover if any link goes down) |
| Path Monitoring | Ping to ACI BD gateway 10.10.100.254 |

---

## 7. F5 BIG-IP i5800 Configuration

### Network Configuration

| VLAN | Tag | Interface | Self IP | Floating Self IP | Purpose |
|---|---|---|---|---|---|
| vlan_outside | 200 | 1.1 | 10.10.200.1/24 (A), 10.10.200.2/24 (B) | 10.10.200.3/24 | Client-side (from Palo Alto) |
| vlan_inside | 201 | 1.2 | 10.10.201.1/24 (A), 10.10.201.2/24 (B) | 10.10.201.3/24 | Server-side (to web/API EPGs) |
| vlan_ha | 300 | 1.3 | 10.10.300.1/24 (A), 10.10.300.2/24 (B) | -- | HA mirroring |

### Route Table

| Destination | Gateway | Description |
|---|---|---|
| 0.0.0.0/0 | 10.10.200.254 | Default route via ACI (back through Palo Alto for return traffic) |
| 10.10.1.0/24 | 10.10.201.254 | Web server subnet via ACI inside |
| 10.10.2.0/24 | 10.10.201.254 | API gateway subnet via ACI inside |

### SSL Profile (Client-side)

| Parameter | Value |
|---|---|
| Profile Name | ssl_client_dmz |
| Certificate | wildcard.example.com (same cert as on Palo Alto for re-encrypted traffic) |
| Private Key | wildcard.example.com.key |
| Protocols | TLS 1.2, TLS 1.3 |
| Ciphers | TLS13-AES128-GCM-SHA256:TLS13-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384 |
| SSL Forward Proxy | Disabled (this is termination, not proxy) |
| Renegotiation | Disabled |

### Virtual Servers

#### vs_dmz_web_https (Web Frontend)

| Parameter | Value |
|---|---|
| Name | vs_dmz_web_https |
| Destination | 10.10.200.10:443 |
| IP Protocol | TCP |
| Client SSL Profile | ssl_client_dmz |
| HTTP Profile | http_standard (insert X-Forwarded-For, X-Forwarded-Proto) |
| OneConnect Profile | oneconnect_default |
| Persistence | Cookie (cookie name: BIG-IP-SRV, HTTP only, secure) |
| SNAT | Automap |
| Default Pool | pool_web_servers |
| iRules | irule_host_routing (route by Host header) |

#### vs_dmz_api_https (API Gateway)

| Parameter | Value |
|---|---|
| Name | vs_dmz_api_https |
| Destination | 10.10.200.11:443 |
| IP Protocol | TCP |
| Client SSL Profile | ssl_client_dmz |
| HTTP Profile | http_api (X-Forwarded-For, X-Request-ID injection) |
| OneConnect Profile | oneconnect_default |
| Persistence | Source Address (timeout 300s) or no persistence for stateless APIs |
| SNAT | Automap |
| Default Pool | pool_api_gateways |

#### vs_dmz_http_redirect

| Parameter | Value |
|---|---|
| Name | vs_dmz_http_redirect |
| Destination | 10.10.200.10:80 and 10.10.200.11:80 |
| iRule | Redirect all HTTP to HTTPS (HTTP::redirect https://[HTTP::host][HTTP::uri]) |

### Pool Definitions

#### pool_web_servers

| Parameter | Value |
|---|---|
| Load Balancing Method | Least Connections (member) |
| Monitor | mon_http_web |
| Members | 10.10.1.10:8080, 10.10.1.11:8080, 10.10.1.12:8080, 10.10.1.13:8080 |
| Action on Service Down | Reselect |
| Slow Ramp Time | 300 seconds (gradual traffic ramp for new members) |

#### pool_api_gateways

| Parameter | Value |
|---|---|
| Load Balancing Method | Round Robin |
| Monitor | mon_http_api |
| Members | 10.10.2.10:8443, 10.10.2.11:8443, 10.10.2.12:8443 |
| Action on Service Down | Reselect |
| Slow Ramp Time | 300 seconds |

### Health Monitors

| Monitor | Type | Send String | Receive String | Interval | Timeout |
|---|---|---|---|---|---|
| mon_http_web | HTTP | GET /health HTTP/1.1\r\nHost: web.example.com\r\n\r\n | "status":"healthy" | 5s | 16s |
| mon_http_api | HTTPS | GET /healthz HTTP/1.1\r\nHost: api.example.com\r\n\r\n | "ok" | 5s | 16s |
| mon_tcp_generic | TCP | -- | -- | 10s | 31s |

### iRule: Host-Based Routing (if using single VIP)

```tcl
when HTTP_REQUEST {
    switch -glob [string tolower [HTTP::host]] {
        "www.example.com" -
        "example.com" {
            pool pool_web_servers
        }
        "api.example.com" {
            pool pool_api_gateways
        }
        default {
            HTTP::respond 404 content "Not Found"
        }
    }
}
```

### F5 HA Configuration

| Parameter | Value |
|---|---|
| HA Type | Active/Standby |
| Config Sync | Device Group: dg_dmz (sync-failover) |
| Network Failover | Enabled (unicast failover via HA VLAN) |
| Mirroring | Connection mirroring on HA VLAN for session persistence |
| Failover Trigger | Loss of network connectivity on vlan_outside or vlan_inside |
| MAC Masquerade | Enabled on traffic VLANs (avoids gratuitous ARP delays) |

---

## 8. OpenStack VMM Integration

### ACI VMM Domain for OpenStack

| Parameter | Value |
|---|---|
| VMM Domain Name | VMM_OpenStack_DMZ |
| Type | OpenStack |
| Controller | OpenStack Keystone endpoint |
| Mechanism Driver | ACI ML2 (cisco_apic_ml2) |
| OpFlex | Enabled on compute nodes |
| VLAN Pool | VLAN_Pool_OpenStack (dynamic, range 2000-2999) |
| Physical Domain | Attached to AEP for compute node ports |

### OpenStack Network-to-EPG Mapping

| OpenStack Network | ACI EPG | Bridge Domain | Subnet |
|---|---|---|---|
| net_dmz_web | EPG_Web | BD_DMZ_Web | 10.10.1.0/24 |
| net_dmz_api | EPG_API | BD_DMZ_API | 10.10.2.0/24 |
| net_app | EPG_App | BD_App | 10.10.10.0/24 |
| net_db | EPG_DB | BD_DB | 10.10.20.0/24 |

OpenStack VMs launched on these networks automatically become ACI endpoints in the corresponding EPGs. Security groups in OpenStack map to ACI Contracts for additional intra-EPG filtering if needed.

---

## 9. Monitoring Approach

### ACI Fabric Monitoring

| What | How | Tool |
|---|---|---|
| Health Scores | APIC health dashboard -- monitor per-Tenant, per-EPG, per-interface scores; alert when score drops below 90 | APIC GUI / API, Prometheus (via ACI exporter) |
| Contract Hit Counts | Atomic counters on Contracts to verify traffic is flowing through expected paths | APIC, Ansible cisco.aci queries |
| TCAM Utilization | Policy CAM usage on leaf switches; alert at 80% | APIC faults, Prometheus |
| Endpoint Tracking | Endpoint table for VM IP/MAC learning; monitor for rogue endpoints | APIC, EDA (Event-Driven Ansible) |
| Fabric Port Health | Interface errors, CRC, input/output drops on spine-leaf links | APIC, streaming telemetry via gNMI to Prometheus |
| Service Graph Health | PBR redirect health group status; verify Palo Alto and F5 are reachable | APIC redirect health policy |
| Faults | Subscribe to APIC fault stream for critical/major/minor faults | APIC webhook to alerting pipeline |

### Palo Alto Monitoring

| What | How | Tool |
|---|---|---|
| Threat Logs | Real-time threat log monitoring for IPS, AV, WildFire alerts | Panorama, syslog to SIEM (Wazuh/Splunk) |
| Traffic Logs | Session logs for all permitted/denied traffic; correlate with ACI flow data | Panorama, syslog forwarding |
| SSL Decryption Metrics | Decrypted session count, decryption errors, unsupported cipher failures | PAN-OS CLI/API, SNMP |
| Session Table | Active session count, session utilization percentage; alert at 80% | SNMP, Panorama |
| HA Status | HA state (active/passive), sync status, failover events | SNMP traps, Panorama alerts |
| CPU/Memory | Data plane and management plane utilization | SNMP, Prometheus (via panos_exporter) |
| GlobalProtect (if used) | VPN tunnel status for admin access | Panorama |
| WildFire Verdicts | Track malware verdicts and zero-day detections | WildFire portal, Panorama |

### F5 BIG-IP Monitoring

| What | How | Tool |
|---|---|---|
| Virtual Server Status | Availability (green/yellow/red), connection count, throughput | F5 Telemetry Streaming to Prometheus |
| Pool Member Health | Individual member status from health monitors; track flapping members | F5 TS, SNMP |
| SSL Metrics | TLS handshakes/sec, cipher usage distribution, certificate expiration | F5 TS, Prometheus |
| Connection Table | Active connections, new connections/sec, server-side connections | F5 TS, iStats |
| Throughput | Client-side and server-side bytes in/out per virtual server | F5 TS to Grafana |
| Response Codes | HTTP 2xx/3xx/4xx/5xx counts per pool and virtual server | F5 HTTP analytics profile, TS |
| HA Sync Status | Config sync status, failover state, traffic group assignment | SNMP, F5 TS |
| iRule Performance | iRule execution time, aborts; identify poorly performing iRules | F5 iStats, TS |

### End-to-End Application Monitoring

| What | How | Tool |
|---|---|---|
| Synthetic Monitoring | External HTTP probes against public VIPs -- verify full path (L3Out -> Palo Alto -> F5 -> backend) | Prometheus Blackbox exporter |
| Response Time | End-to-end latency from client to backend | F5 analytics, application APM |
| SSL Certificate Expiry | Monitor cert expiration on Palo Alto and F5; alert 30 days before | cert-manager or custom Ansible check |
| Log Aggregation | Centralize logs from ACI (syslog), Palo Alto (syslog), F5 (TS), OpenStack VMs | ELK stack or Loki |
| Alerting | PagerDuty/Opsgenie integration for critical alerts | Prometheus Alertmanager |
| Dashboard | Unified Grafana dashboard: ACI health, PA threat stats, F5 pool health, application latency | Grafana with Prometheus/Loki datasources |

### Monitoring Architecture Diagram

```
  [ACI APIC] --syslog/gNMI--> [Prometheus + ACI Exporter] ---> [Grafana]
  [Palo Alto] --syslog-------> [Wazuh/Splunk SIEM]
  [Palo Alto] --SNMP/API-----> [Prometheus + panos_exporter] -> [Grafana]
  [F5 BIG-IP] --TS (push)----> [Prometheus pushgateway] ------> [Grafana]
  [F5 BIG-IP] --SNMP---------> [Prometheus + snmp_exporter] --> [Grafana]
  [OpenStack VMs] --node_exporter/app metrics--> [Prometheus] -> [Grafana]
  [Blackbox Exporter] --synthetic probes-------> [Prometheus] -> [Grafana]
  [All] --alerts--> [Prometheus Alertmanager] ---> [PagerDuty/Opsgenie/Slack]
```

---

## 10. Ansible Automation

All infrastructure is deployed and managed via Ansible. No manual CLI or GUI configuration.

### Required Ansible Collections

```yaml
# requirements.yml
collections:
  - name: cisco.aci
    version: ">=2.9.0"
  - name: paloaltonetworks.panos
    version: ">=2.19.0"
  - name: f5networks.f5_modules
    version: ">=1.28.0"
  - name: openstack.cloud
    version: ">=2.0.0"
  - name: community.general
```

### Ansible Inventory Structure

```
inventory/
  hosts.yml                  # Static inventory
  group_vars/
    all.yml                  # Global vars (NTP, DNS, syslog servers)
    aci.yml                  # APIC credentials, Tenant definitions
    paloalto.yml             # Panorama/PA credentials, zone definitions
    f5.yml                   # F5 credentials, VS/pool definitions
    openstack.yml            # OpenStack auth, network definitions
  host_vars/
    apic1.yml
    pa5400-a.yml
    pa5400-b.yml
    f5-i5800-a.yml
    f5-i5800-b.yml
```

### Playbook Structure

```
playbooks/
  site.yml                        # Master playbook (orchestrates all)
  aci/
    01_tenant_vrf_bd.yml          # Tenant, VRF, Bridge Domains, Subnets
    02_app_profile_epgs.yml       # Application Profile, EPGs, VMM domain binding
    03_contracts_filters.yml      # Contracts, Filters, Subjects
    04_l3out.yml                  # L3Out, External EPG, route control
    05_l4l7_devices.yml           # L4-L7 device registration (PA, F5)
    06_service_graph.yml          # Service Graph template, device selection
    07_pbr_policies.yml           # PBR redirect policies and health groups
    08_contract_graph_binding.yml # Bind Service Graph to Contract Subjects
  paloalto/
    01_interfaces_zones.yml       # Interfaces, zones, virtual router
    02_security_profiles.yml      # AV, AS, VP, URL, WF, FB profiles
    03_decryption.yml             # SSL decryption policy, certificates
    04_security_policies.yml      # Security rules
    05_dos_protection.yml         # Zone protection, DoS profiles
    06_ha_config.yml              # HA pair configuration
    07_commit.yml                 # Commit configuration to device
  f5/
    01_network.yml                # VLANs, Self IPs, routes
    02_ssl_profiles.yml           # SSL client profiles, certificates
    03_monitors.yml               # Health monitors
    04_pools.yml                  # Pools and pool members
    05_virtual_servers.yml        # Virtual servers, profiles, iRules
    06_ha.yml                     # Device group, config sync, failover
  monitoring/
    01_prometheus_targets.yml     # Configure Prometheus scrape targets
    02_grafana_dashboards.yml     # Deploy Grafana dashboards
    03_alertmanager_rules.yml     # Deploy alert rules
  openstack/
    01_networks.yml               # Create OpenStack networks (mapped to EPGs)
    02_security_groups.yml        # Security groups (mapped to Contracts)
    03_instances.yml              # Launch web/API/app/DB VMs
```

### Key Playbook Examples

#### ACI Service Graph Registration (playbooks/aci/06_service_graph.yml)

```yaml
---
- name: Configure ACI Service Graph for Palo Alto + F5 chain
  hosts: apic1
  gather_facts: false
  collections:
    - cisco.aci

  vars:
    aci_tenant: TN_DMZ_Production

  tasks:
    - name: Create L4-L7 Logical Device - Palo Alto
      cisco.aci.aci_rest:
        host: "{{ aci_host }}"
        username: "{{ aci_username }}"
        password: "{{ aci_password }}"
        validate_certs: false
        method: post
        path: /api/mo/uni/tn-{{ aci_tenant }}.json
        content:
          vnsLDevVip:
            attributes:
              name: LDEV_PaloAlto_PA5400
              devtype: PHYSICAL
              funcType: GoTo
              svcType: FW
              contextAware: single-Context
              managed: "yes"
            children:
              - vnsCMgmt:
                  attributes:
                    host: "{{ pa_mgmt_ip }}"
                    name: PA_Mgmt
              - vnsCDev:
                  attributes:
                    name: PA5400-A
                    devCtxLbl: ""
                  children:
                    - vnsCIf:
                        attributes:
                          name: outside
                          vnicName: ethernet1/1
                    - vnsCIf:
                        attributes:
                          name: inside
                          vnicName: ethernet1/2
              - vnsLIf:
                  attributes:
                    name: consumer
                    encap: "vlan-100"
                  children:
                    - vnsRsCIfAttN:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_PaloAlto_PA5400/cDev-PA5400-A/cIf-[outside]"
              - vnsLIf:
                  attributes:
                    name: provider
                    encap: "vlan-101"
                  children:
                    - vnsRsCIfAttN:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_PaloAlto_PA5400/cDev-PA5400-A/cIf-[inside]"

    - name: Create L4-L7 Logical Device - F5 BIG-IP
      cisco.aci.aci_rest:
        host: "{{ aci_host }}"
        username: "{{ aci_username }}"
        password: "{{ aci_password }}"
        validate_certs: false
        method: post
        path: /api/mo/uni/tn-{{ aci_tenant }}.json
        content:
          vnsLDevVip:
            attributes:
              name: LDEV_F5_i5800
              devtype: PHYSICAL
              funcType: GoTo
              svcType: ADC
              contextAware: single-Context
              managed: "yes"
            children:
              - vnsCMgmt:
                  attributes:
                    host: "{{ f5_mgmt_ip }}"
                    name: F5_Mgmt
              - vnsCDev:
                  attributes:
                    name: F5-i5800-A
                  children:
                    - vnsCIf:
                        attributes:
                          name: outside
                          vnicName: "1.1"
                    - vnsCIf:
                        attributes:
                          name: inside
                          vnicName: "1.2"
              - vnsLIf:
                  attributes:
                    name: consumer
                    encap: "vlan-200"
                  children:
                    - vnsRsCIfAttN:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_F5_i5800/cDev-F5-i5800-A/cIf-[outside]"
              - vnsLIf:
                  attributes:
                    name: provider
                    encap: "vlan-201"
                  children:
                    - vnsRsCIfAttN:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_F5_i5800/cDev-F5-i5800-A/cIf-[inside]"

    - name: Create Service Graph Template with two nodes
      cisco.aci.aci_rest:
        host: "{{ aci_host }}"
        username: "{{ aci_username }}"
        password: "{{ aci_password }}"
        validate_certs: false
        method: post
        path: /api/mo/uni/tn-{{ aci_tenant }}.json
        content:
          vnsAbsGraph:
            attributes:
              name: SGT_PaloAlto_F5
              uiTemplateType: UNSPECIFIED
            children:
              - vnsAbsTermNodeCon:
                  attributes:
                    name: T1
              - vnsAbsTermNodeProv:
                  attributes:
                    name: T2
              - vnsAbsNode:
                  attributes:
                    name: N1_PaloAlto
                    funcTemplateType: FW_ROUTED
                    funcType: GoTo
                    managed: "yes"
                    routingMode: Redirect
                  children:
                    - vnsAbsFuncConn:
                        attributes:
                          name: consumer
                          attNotify: "no"
                    - vnsAbsFuncConn:
                        attributes:
                          name: provider
                          attNotify: "no"
                    - vnsRsNodeToLDev:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_PaloAlto_PA5400"
              - vnsAbsNode:
                  attributes:
                    name: N2_F5
                    funcTemplateType: ADC_ONE_ARM
                    funcType: GoTo
                    managed: "yes"
                    routingMode: Redirect
                  children:
                    - vnsAbsFuncConn:
                        attributes:
                          name: consumer
                          attNotify: "no"
                    - vnsAbsFuncConn:
                        attributes:
                          name: provider
                          attNotify: "no"
                    - vnsRsNodeToLDev:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/lDevVip-LDEV_F5_i5800"
              - vnsAbsConnection:
                  attributes:
                    name: C1
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsTermNodeCon-T1/AbsTConn"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsNode-N1_PaloAlto/AbsFConn-consumer"
              - vnsAbsConnection:
                  attributes:
                    name: C2
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsNode-N1_PaloAlto/AbsFConn-provider"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsNode-N2_F5/AbsFConn-consumer"
              - vnsAbsConnection:
                  attributes:
                    name: C3
                    adjType: L3
                    connDir: provider
                    connType: external
                  children:
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsNode-N2_F5/AbsFConn-provider"
                    - vnsRsAbsConnectionConns:
                        attributes:
                          tDn: "uni/tn-{{ aci_tenant }}/AbsGraph-SGT_PaloAlto_F5/AbsTermNodeProv-T2/AbsTConn"

    - name: Bind Service Graph to Contract Subject
      cisco.aci.aci_rest:
        host: "{{ aci_host }}"
        username: "{{ aci_username }}"
        password: "{{ aci_password }}"
        validate_certs: false
        method: post
        path: /api/mo/uni/tn-{{ aci_tenant }}/brc-CON_Internet_to_DMZ/subj-SUB_HTTPS.json
        content:
          vzRsSubjGraphAtt:
            attributes:
              tnVnsAbsGraphName: SGT_PaloAlto_F5
```

#### Palo Alto SSL Decryption (playbooks/paloalto/03_decryption.yml)

```yaml
---
- name: Configure Palo Alto SSL Inbound Inspection
  hosts: pa5400-a
  gather_facts: false
  collections:
    - paloaltonetworks.panos

  tasks:
    - name: Import server certificate for inbound decryption
      paloaltonetworks.panos.panos_import:
        provider: "{{ panos_provider }}"
        category: certificate
        certificate_name: wildcard_example_com
        format: pkcs12
        passphrase: "{{ cert_passphrase }}"
        filename: "{{ playbook_dir }}/files/wildcard.example.com.p12"

    - name: Create SSL decryption profile
      paloaltonetworks.panos.panos_decryption_rule:
        provider: "{{ panos_provider }}"
        name: RULE_Decrypt_Inbound_HTTPS
        source_zones: ['Untrust']
        destination_zones: ['DMZ']
        destination_addresses: ['203.0.113.0/24']
        services: ['service-https']
        action: decrypt
        decryption_type: ssl-inbound-inspection
        ssl_certificate: wildcard_example_com
        decryption_profile: PROF_Inbound_Decrypt

    - name: Create decryption profile
      paloaltonetworks.panos.panos_decryption_profile:
        provider: "{{ panos_provider }}"
        name: PROF_Inbound_Decrypt
        ssl_inbound_proxy:
          block_if_no_resource: true
          block_unsupported_version: true
          block_unsupported_cipher: true

    - name: Commit configuration
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"
```

#### F5 Virtual Server (playbooks/f5/05_virtual_servers.yml)

```yaml
---
- name: Configure F5 BIG-IP Virtual Servers
  hosts: f5-i5800-a
  gather_facts: false
  collections:
    - f5networks.f5_modules

  tasks:
    - name: Create HTTP health monitor for web servers
      f5networks.f5_modules.bigip_monitor_http:
        provider: "{{ f5_provider }}"
        name: mon_http_web
        send: "GET /health HTTP/1.1\\r\\nHost: web.example.com\\r\\n\\r\\n"
        receive: '"status":"healthy"'
        interval: 5
        timeout: 16
        state: present

    - name: Create pool for web servers
      f5networks.f5_modules.bigip_pool:
        provider: "{{ f5_provider }}"
        name: pool_web_servers
        lb_method: least-connections-member
        monitors:
          - mon_http_web
        slow_ramp_time: 300
        state: present

    - name: Add web server pool members
      f5networks.f5_modules.bigip_pool_member:
        provider: "{{ f5_provider }}"
        pool: pool_web_servers
        host: "{{ item.address }}"
        port: "{{ item.port }}"
        name: "{{ item.name }}"
        state: present
      loop:
        - { name: web-01, address: 10.10.1.10, port: 8080 }
        - { name: web-02, address: 10.10.1.11, port: 8080 }
        - { name: web-03, address: 10.10.1.12, port: 8080 }
        - { name: web-04, address: 10.10.1.13, port: 8080 }

    - name: Import SSL certificate
      f5networks.f5_modules.bigip_ssl_certificate:
        provider: "{{ f5_provider }}"
        name: wildcard_example_com
        content: "{{ lookup('file', playbook_dir + '/files/wildcard.example.com.crt') }}"
        state: present

    - name: Import SSL key
      f5networks.f5_modules.bigip_ssl_key:
        provider: "{{ f5_provider }}"
        name: wildcard_example_com
        content: "{{ lookup('file', playbook_dir + '/files/wildcard.example.com.key') }}"
        state: present

    - name: Create client SSL profile
      f5networks.f5_modules.bigip_profile_client_ssl:
        provider: "{{ f5_provider }}"
        name: ssl_client_dmz
        cert_key_chain:
          - cert: wildcard_example_com
            key: wildcard_example_com
        ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384"
        state: present

    - name: Create HTTPS virtual server for web
      f5networks.f5_modules.bigip_virtual_server:
        provider: "{{ f5_provider }}"
        name: vs_dmz_web_https
        destination: 10.10.200.10
        port: 443
        pool: pool_web_servers
        snat: Automap
        profiles:
          - name: ssl_client_dmz
            context: client-side
          - http
          - oneconnect
        default_persistence_profile: cookie
        state: present

    - name: Sync configuration to standby
      f5networks.f5_modules.bigip_configsync_action:
        provider: "{{ f5_provider }}"
        device_group: dg_dmz
        sync_device_to_group: true
```

### Ansible Vault for Secrets

All credentials are encrypted with Ansible Vault:

```bash
# Encrypt secrets
ansible-vault encrypt inventory/group_vars/aci.yml
ansible-vault encrypt inventory/group_vars/paloalto.yml
ansible-vault encrypt inventory/group_vars/f5.yml

# Run playbooks with vault
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --ask-vault-pass
```

### Execution Order (site.yml)

```yaml
---
- name: DMZ Infrastructure Deployment
  import_playbook: aci/01_tenant_vrf_bd.yml
- import_playbook: aci/02_app_profile_epgs.yml
- import_playbook: aci/03_contracts_filters.yml
- import_playbook: aci/04_l3out.yml
- import_playbook: paloalto/01_interfaces_zones.yml
- import_playbook: paloalto/02_security_profiles.yml
- import_playbook: paloalto/03_decryption.yml
- import_playbook: paloalto/04_security_policies.yml
- import_playbook: paloalto/05_dos_protection.yml
- import_playbook: paloalto/06_ha_config.yml
- import_playbook: paloalto/07_commit.yml
- import_playbook: f5/01_network.yml
- import_playbook: f5/02_ssl_profiles.yml
- import_playbook: f5/03_monitors.yml
- import_playbook: f5/04_pools.yml
- import_playbook: f5/05_virtual_servers.yml
- import_playbook: f5/06_ha.yml
- import_playbook: aci/05_l4l7_devices.yml
- import_playbook: aci/06_service_graph.yml
- import_playbook: aci/07_pbr_policies.yml
- import_playbook: aci/08_contract_graph_binding.yml
- import_playbook: openstack/01_networks.yml
- import_playbook: openstack/02_security_groups.yml
- import_playbook: openstack/03_instances.yml
- import_playbook: monitoring/01_prometheus_targets.yml
- import_playbook: monitoring/02_grafana_dashboards.yml
- import_playbook: monitoring/03_alertmanager_rules.yml
```

Note: ACI L4-L7 device registration (step 05) runs after Palo Alto and F5 are configured, because APIC needs to connect to the device management interfaces for managed mode.

---

## 11. Security Summary

| Layer | Control | Technology |
|---|---|---|
| Perimeter | NGFW with App-ID, IPS, AV, WildFire | Palo Alto PA-5400 |
| SSL Visibility | Inbound TLS decryption for content inspection | Palo Alto SSL Inbound Inspection |
| DDoS Mitigation | Zone protection profiles, SYN cookies | Palo Alto DoS Protection |
| Load Balancing | L7 distribution, health checking, session persistence | F5 BIG-IP i5800 |
| Microsegmentation | Inter-EPG contract enforcement at leaf switch | ACI Contracts and Filters |
| East-West Isolation | No direct Web-to-DB or API-to-DB communication | ACI implicit deny (no contract = no access) |
| Network Fabric | Policy-driven SDN, PBR for service insertion | Cisco ACI |
| Threat Intelligence | Cloud sandboxing, zero-day detection | Palo Alto WildFire |
| Logging and SIEM | Centralized log aggregation and correlation | Wazuh/Splunk + ELK/Loki |
| Automation | Reproducible, auditable, idempotent deployments | Ansible with Vault |
| Certificate Management | Automated cert distribution, expiry monitoring | Ansible + Prometheus alerts |

---

## 12. Architectural Decisions and Tradeoffs

| Decision | Rationale | Alternative Considered |
|---|---|---|
| Palo Alto before F5 in Service Graph chain | SSL decryption must happen before load balancing so threat inspection sees all traffic | F5 first (would require F5 to pass decrypted traffic to PA, adding complexity) |
| SSL re-encryption between PA and F5 | Protects traffic on the wire between appliances; F5 handles final offload | Cleartext between PA and F5 (faster but insecure if intermediate network is compromised) |
| ACI PBR (not VLAN stitching) | PBR provides health-aware redirection and supports HA failover natively | VLAN stitching (simpler but no health awareness, manual failover) |
| Routed mode for both appliances | Provides full routing control, supports NAT and SNAT, aligns with ACI PBR best practices | Transparent/bridge mode (simpler but limited functionality, harder to troubleshoot) |
| ACI Contracts for east-west (no PA for east-west) | ACI hardware-enforced microsegmentation is wire-rate and does not add latency; PA is reserved for north-south where L7 inspection is needed | Route all east-west through PA (adds latency, creates bottleneck, PA becomes single point of failure for all traffic) |
| SNAT Automap on F5 | Simplifies return traffic routing (server replies go to F5, not directly to client) | DSR/nPath (higher performance but requires backend routing changes, incompatible with ACI PBR) |
| Active/Passive HA for both PA and F5 | Simpler configuration, deterministic failover, no split-brain risk | Active/Active (higher throughput but more complex, asymmetric routing challenges with ACI PBR) |
| Ansible over Terraform for this stack | cisco.aci, paloaltonetworks.panos, and f5networks.f5_modules collections are mature; Ansible handles imperative device configuration better than Terraform for network appliances | Terraform (better for declarative cloud infra, but network appliance providers are less mature) |
