# Load Balancer Migration Analysis: Citrix NetScaler VPX to F5 BIG-IP VE / HAProxy (Octavia)

## OpenStack Private Cloud with Cisco ACI Networking

---

## 1. Current State Assessment

### Existing NetScaler Deployment

| Attribute | Detail |
|-----------|--------|
| Platform | Citrix NetScaler VPX (virtual appliance) |
| Scale | ~30 virtual servers across 4 NetScaler HA pairs (8 instances total) |
| Data Centers | 2 (active-active with GSLB) |
| Kubernetes | NetScaler CPX used as ingress controller in on-prem K8s clusters |
| Hypervisor | OpenStack (KVM) |
| Network Fabric | Cisco ACI |

### Capabilities in Active Use

1. **L7 Content Switching** -- URI-path, host-header, and cookie-based routing across virtual servers.
2. **SSL/TLS Offloading** -- Centralized certificate management, TLS 1.2/1.3 termination, SNI-based vserver selection.
3. **GSLB (Global Server Load Balancing)** -- DNS-based traffic steering between two data centers with health-aware failover.
4. **WAF (Application Firewall)** -- OWASP top-10 protection, bot mitigation, custom signature rules, IP reputation.
5. **Health Monitoring** -- HTTP/HTTPS content-match monitors, TCP half-open, ICMP, ECV (extended content verification), custom scripted monitors.
6. **Kubernetes Ingress** -- NetScaler CPX as ingress controller with CRD-based configuration (Citrix Ingress Controller / CIC).

---

## 2. Candidate Platform Comparison

### 2.1 Option A: F5 BIG-IP VE (Virtual Edition)

#### Architecture Overview

F5 BIG-IP VE runs as a virtual appliance inside OpenStack (KVM/libvirt). Each VE instance is licensed by throughput tier (25 Mbps to 10 Gbps). Deployed as HA pairs with network failover via VRRP or device-service clustering (DSC). The BIG-IP LTM (Local Traffic Manager) module handles L4-L7 load balancing; ASM (Advanced Security Manager, now "Advanced WAF") provides WAF; DNS module provides GSLB.

#### Capability Mapping

| NetScaler Capability | F5 Equivalent | Parity | Notes |
|----------------------|---------------|--------|-------|
| L7 Content Switching | LTM iRules + Local Traffic Policies | Full | iRules (Tcl-based) are extremely flexible; Local Traffic Policies provide GUI-driven L7 routing |
| SSL Offloading | LTM SSL profiles + certificate bundles | Full | Hardware-accelerated on physical; software on VE. Supports TLS 1.3, OCSP stapling, certificate transparency |
| GSLB | BIG-IP DNS (formerly GTM) | Full | DNS-based wide-IP with topology records, persistence, health-aware failover. Requires separate DNS module license per VE |
| WAF | Advanced WAF (ASM) | Full | Full OWASP coverage, bot defense, behavioral analytics, DataGuard, IP intelligence. Generally considered more mature than NetScaler AppFW |
| Health Monitoring | LTM Monitors (HTTP, HTTPS, TCP, SIP, external script) | Full | External monitors can run arbitrary scripts; send-string/receive-string equivalents to ECV |
| K8s Ingress | F5 Container Ingress Services (CIS) | Full | CIS watches K8s API and programs BIG-IP LTM. Supports Ingress, Routes (OpenShift), and F5 CRDs. Also F5 NGINX Ingress Controller available |

#### Cisco ACI Integration

- **Service Graph with L4-L7 Device Package**: F5 has a mature, Cisco-validated ACI device package. BIG-IP VE registers as an L4-L7 device in APIC. Service Graphs insert F5 inline (GoTo/GoThrough) or one-arm (redirect) mode.
- **APIC-Driven Automation**: F5 ACI ServiceCenter (an APIC app) allows provisioning VIPs, pools, and profiles directly from APIC. Alternatively, the Terraform ACI provider + F5 BIG-IP provider can orchestrate both sides.
- **VLAN/EPG Stitching**: ACI Service Graph auto-provisions bridge domains and contracts to stitch the F5 VE legs into the correct EPGs. This is well-documented and production-proven in many enterprises.

#### Automation & IaC

| Tool | Support Level |
|------|---------------|
| Terraform | `f5networks/bigip` provider -- full coverage of LTM, ASM policy deployment, DNS/GSLB |
| Ansible | `f5networks.f5_modules` collection -- 150+ modules |
| AS3 (Declarative API) | JSON declarative model for entire application configuration; idempotent, source-controllable |
| FAST Templates | Parameterized AS3 templates for self-service |
| Telemetry Streaming | Push metrics/logs to Prometheus, Splunk, ELK, etc. |
| OpenStack Heat/LBaaS | No native Octavia driver; deployed as standalone VMs managed by Terraform/Ansible |

#### Licensing & Cost

| Item | Estimate (Annual) |
|------|-------------------|
| BIG-IP VE (LTM + DNS + Advanced WAF), 1 Gbps tier, 8 instances | ~$320K-$400K (perpetual licenses amortized over 3 years, or subscription ~$40K-$50K/instance/year) |
| F5 support (Premium) | Included in subscription; ~18% of perpetual |
| Operational labor (F5 expertise) | Medium-High -- F5 is widely known but iRules/AS3 require skilled engineers |
| ACI device package maintenance | Minimal -- Cisco/F5 jointly maintain |

**TCO Note**: F5 is the most expensive option. ELA (Enterprise License Agreement) negotiations or BIG-IQ-managed licensing pools can reduce per-instance cost. The "BIG-IP Next" platform (F5's next-gen) is also worth evaluating for future-proofing, though VE maturity is ahead.

---

### 2.2 Option B: HAProxy via OpenStack Octavia

#### Architecture Overview

OpenStack Octavia is the native LBaaS v2 service. It provisions HAProxy instances inside "amphorae" -- lightweight VMs (or containers in newer deployments) that run HAProxy. Octavia manages the full lifecycle: provisioning, health monitoring, failover, and certificate management. Each load balancer gets a dedicated amphora (or an active-standby amphora pair).

#### Capability Mapping

| NetScaler Capability | Octavia/HAProxy Equivalent | Parity | Notes |
|----------------------|---------------------------|--------|-------|
| L7 Content Switching | Octavia L7 policies + rules (host, path, header, cookie) | Partial | Octavia L7 policies cover common cases. Complex multi-condition switching may need HAProxy config injection or custom provider |
| SSL Offloading | Octavia TLS-terminated listeners + Barbican (secret store) | Full | Barbican stores certs; Octavia configures HAProxy SSL frontend. Supports SNI. TLS 1.3 with HAProxy 2.x+ |
| GSLB | **Not available natively** | None | Octavia has no GSLB feature. Must add a separate DNS-based GSLB layer (see below) |
| WAF | **Not available natively** | None | HAProxy has basic ACLs but no WAF engine. Must add a separate WAF layer (see below) |
| Health Monitoring | Octavia health monitors (HTTP, HTTPS, TCP, PING, TLS-HELLO) | Partial | No equivalent to ECV/scripted monitors. HTTP monitors support expected-codes and url-path but not body content matching |
| K8s Ingress | HAProxy Kubernetes Ingress Controller (haproxytech) or use a separate ingress | Partial | HAProxy Technologies offers a K8s ingress controller, but it is separate from Octavia -- no unified management |

#### GSLB Gap -- Mitigation Options

Since Octavia has no GSLB, you need an external solution:

1. **PowerDNS with Lua scripting or dnsdist** -- Open source, can implement weighted/geo/health-based DNS steering. Requires custom development.
2. **NS1 (managed DNS)** -- SaaS-based, supports health checks and failover steering. Adds external dependency.
3. **Consul + consul-template + DNS** -- HashiCorp Consul's prepared queries can route to healthy DCs. Requires Consul adoption.
4. **Keep NetScaler for GSLB only** -- Run a minimal NetScaler pair solely for GSLB/DNS delegation. Pragmatic but retains licensing.

**Assessment**: None of these provide a seamless, single-platform GSLB equivalent. This is the biggest gap in the Octavia path.

#### WAF Gap -- Mitigation Options

1. **ModSecurity + NGINX/Apache reverse proxy** -- Deploy a WAF tier in front of Octavia VIPs. Well-understood OWASP CRS ruleset. Adds latency and another failure domain.
2. **Coraza WAF (ModSecurity-compatible, Go-based)** -- Can run as a sidecar or standalone proxy. Newer, gaining traction.
3. **Cloud-native WAF (e.g., AWS WAF via CloudFront)** -- Not applicable for private cloud.
4. **Commercial WAF (e.g., Imperva, Signal Sciences/Fastly)** -- Agent-based or reverse-proxy. Adds cost.

**Assessment**: Adding a separate WAF layer significantly increases architectural complexity and operational burden. It also makes ACI Service Graph insertion more complex (two inline appliances).

#### Cisco ACI Integration

- **Octavia + ACI**: There is **no native ACI Service Graph device package for Octavia/HAProxy**. The amphorae are standard Nova VMs with ports on Neutron networks. ACI sees them as regular endpoints, not managed L4-L7 devices.
- **Workarounds**:
  - Use ACI "unmanaged" L4-L7 device mode -- manually configure the service graph to redirect traffic to the amphora VIP port. Loses dynamic lifecycle management.
  - Use Neutron `opflex` ML2 plugin (Cisco ACI's OpenStack integration) to place amphora interfaces in the correct EPGs. This works for basic connectivity but is not "Service Graph aware."
  - Accept that Octavia operates outside of ACI's service graph model and manage traffic steering via OpenStack Neutron/allowed-address-pairs and ACI contracts at the EPG level.

- **Verdict**: ACI Service Graph integration is weak to nonexistent. If your operations team relies heavily on APIC-driven service insertion and traffic visibility, Octavia is a poor fit.

#### Automation & IaC

| Tool | Support Level |
|------|---------------|
| Terraform | `openstack` provider has full Octavia (loadbalancer, listener, pool, member, monitor, L7 policy) resources |
| Ansible | `openstack.cloud` collection includes LB modules |
| OpenStack CLI/API | Native -- Octavia is a first-class OpenStack service |
| Heat | Full support via `OS::Octavia::*` resources |
| Barbican | Integrated certificate/secret management |

**Strength**: Octavia is the most "OpenStack-native" option. If your IaC is heavily OpenStack-centric (Heat, Terraform OpenStack provider), this is the smoothest integration path for basic LB.

#### Licensing & Cost

| Item | Estimate (Annual) |
|------|-------------------|
| Software licensing | $0 (open source) |
| Compute for amphorae (8 HA pairs equivalent: ~16 small VMs) | Negligible incremental cost on existing OpenStack |
| GSLB solution (PowerDNS or SaaS) | $0-$50K depending on approach |
| WAF solution (ModSecurity or commercial) | $0-$100K+ depending on approach |
| Operational labor | High -- multiple systems to operate, monitor, troubleshoot; no single vendor support |
| Engineering effort for GSLB + WAF integration | 2-4 engineer-months |

**TCO Note**: Deceptively cheap at first glance. The hidden cost is operational complexity from stitching together 3-4 systems (Octavia + GSLB + WAF + K8s ingress) that NetScaler or F5 provide in a single platform. Staff retraining and on-call complexity are real costs.

---

### 2.3 Option C: Stay on Citrix NetScaler (Status Quo + Modernize)

#### Rationale

Before migrating, evaluate whether the pain points driving this evaluation can be resolved within NetScaler.

#### Current Capabilities

All 6 capabilities are already in production and working. No gaps.

#### Cisco ACI Integration

- Citrix has an ACI device package for NetScaler VPX. Service Graph integration is available but has historically been **less mature** than F5's ACI integration. Cisco's reference architectures favor F5 or native ACI PBR for service insertion.
- If ACI Service Graph is not heavily used today (i.e., you manage NetScaler VIPs independently of APIC), this may be a non-issue.

#### Automation & IaC

| Tool | Support Level |
|------|---------------|
| Terraform | `citrixadc/citrixadc` provider -- reasonable coverage |
| Ansible | `citrix.adc` collection -- covers most config objects |
| NITRO API | RESTful API for full programmatic control |
| ADM (Application Delivery Management) | Centralized management, analytics, license pooling (Citrix's equivalent of F5 BIG-IQ) |
| Kubernetes (CPX + CIC) | Citrix Ingress Controller + CPX is mature; supports Ingress, Gateway API CRDs |

#### Licensing & Cost

| Item | Estimate (Annual) |
|------|-------------------|
| NetScaler VPX Premium (8 instances, assuming existing perpetual licenses with active support) | ~$150K-$250K support renewal |
| NetScaler ADM (if used) | ~$30K-$50K |
| Operational labor | Known -- existing team has tribal knowledge |
| Migration risk & cost | $0 (no migration) |

#### Risks of Staying

- **Broadcom Acquisition**: Citrix was acquired by Cloud Software Group (backed by Vista Equity and Elliott). Licensing changes, support quality degradation, and product roadmap uncertainty are real concerns. This is likely the primary driver for evaluating alternatives.
- **Community & Ecosystem Shrinkage**: NetScaler's community and third-party integration ecosystem is smaller than F5's and much smaller than the HAProxy/open-source ecosystem.
- **Talent Availability**: NetScaler expertise is harder to hire for than F5 or HAProxy.

---

## 3. Side-by-Side Comparison Matrix

| Criterion | F5 BIG-IP VE | HAProxy/Octavia | NetScaler (Stay) |
|-----------|-------------|-----------------|-------------------|
| **L7 Content Switching** | Excellent (iRules + policies) | Good (Octavia L7 policies; limited complex logic) | Excellent (content switching policies) |
| **SSL Offloading** | Excellent | Good (via Barbican) | Excellent |
| **GSLB** | Excellent (BIG-IP DNS) | Not available (requires external solution) | Excellent (built-in GSLB) |
| **WAF** | Excellent (Advanced WAF) | Not available (requires external solution) | Good (AppFW; less mature than F5 ASM) |
| **Health Monitoring** | Excellent (external script monitors) | Adequate (basic HTTP/TCP/PING) | Excellent (ECV, scripted) |
| **K8s Ingress** | Good (CIS + optional NGINX IC) | Adequate (separate HAProxy IC) | Good (CPX + CIC) |
| **ACI Service Graph** | Excellent (mature device package, Cisco-validated) | Poor (no device package; manual workarounds) | Fair (device package exists but less mature) |
| **Terraform/Ansible** | Excellent (AS3 + provider) | Excellent (native OpenStack) | Good (NITRO-based provider) |
| **Declarative API** | Excellent (AS3, idempotent) | Good (OpenStack API is declarative) | Fair (NITRO is imperative; StyleBooks add declarative layer) |
| **Observability** | Excellent (Telemetry Streaming to Prometheus/Splunk/ELK) | Good (HAProxy stats + Prometheus exporter) | Good (ADM + SNMP + syslog; Prometheus support via exporter) |
| **Vendor Risk** | Low (F5 is publicly traded, stable, dominant in ADC market) | None (open source) | High (Broadcom/CSG ownership; licensing uncertainty) |
| **Talent Pool** | Large | Very Large (HAProxy/Linux skills) | Shrinking |
| **Migration Effort** | Medium (config translation + ACI re-integration) | High (config translation + build GSLB + build WAF + ACI workarounds) | None |
| **Annual TCO (est.)** | $350K-$450K | $100K-$200K (with GSLB+WAF additions) | $180K-$300K |
| **3-Year TCO (est.)** | $1.0M-$1.35M | $500K-$800K (incl. engineering buildout) | $540K-$900K |

---

## 4. Recommendation

### Primary Recommendation: Migrate to F5 BIG-IP VE

**Rationale:**

1. **Feature completeness**: F5 is the only alternative that provides a 1:1 capability replacement for every NetScaler feature in use -- L7 switching, SSL offload, GSLB, WAF, advanced health checks, and Kubernetes ingress -- in a single platform. This eliminates the architectural fragmentation risk of the Octavia path.

2. **ACI Service Graph**: F5 has the strongest ACI integration of any ADC vendor. The device package is Cisco-validated and actively maintained. If your ACI operations team uses Service Graphs for traffic steering and policy enforcement, F5 is the natural fit.

3. **Automation maturity**: AS3 (Application Services 3) provides a truly declarative, JSON-based configuration model that is idempotent and source-controllable. Combined with Terraform, Ansible, and Telemetry Streaming, F5 offers the most complete automation story.

4. **Vendor stability**: F5 is a publicly traded, ADC-focused company with a clear roadmap (BIG-IP Next, distributed cloud services). No acquisition risk comparable to Citrix/Broadcom.

5. **Migration path clarity**: Every NetScaler concept has a direct F5 equivalent. Citrix-to-F5 migration tooling exists (F5's "Journeys" migration tool can parse NetScaler `ns.conf` and generate AS3 declarations).

### When to Choose HAProxy/Octavia Instead

Choose Octavia if **all** of the following are true:
- You do not need GSLB (single data center, or willing to build DNS-based steering separately)
- WAF can be handled by a separate inline proxy or is not required
- ACI Service Graph integration is not important to your operations
- Minimizing licensing cost is the overriding priority
- Your team has strong Linux/open-source operational skills

### When to Stay on NetScaler

Stay if:
- Broadcom/CSG licensing terms remain acceptable after renewal negotiation
- The current deployment is stable and the team is productive
- Migration budget and risk window are not available in the near term
- You can negotiate a favorable multi-year ELA that locks in pricing

---

## 5. Migration Plan: NetScaler to F5 BIG-IP VE

### Phase 0: Foundation (Weeks 1-4)

**Objective**: Establish F5 platform in OpenStack + ACI, validate base connectivity.

| Task | Detail |
|------|--------|
| Procure F5 licenses | Negotiate VE subscription (LTM + DNS + Advanced WAF). Consider BIG-IQ for centralized management if managing 8+ instances |
| Deploy BIG-IP VE images | Upload VE qcow2 to Glance. Create Heat/Terraform templates for consistent VM provisioning (4 vCPU, 8 GB RAM minimum per VE) |
| ACI Service Graph setup | Install F5 device package on APIC. Create L4-L7 device cluster objects for each VE HA pair. Validate GoTo/GoThrough service graph with a test application |
| Network plumbing | Provision VLAN interfaces on VE matching existing NetScaler SNIP/VIP subnets. Verify ACI contracts and EPG-to-VE stitching |
| HA configuration | Configure F5 DSC (Device Service Clustering) for each pair. Validate failover with traffic flowing |
| Automation baseline | Deploy AS3 extension on all VEs. Create Terraform workspace with `f5networks/bigip` provider. Validate a "hello world" AS3 declaration |
| Monitoring | Configure Telemetry Streaming to push to existing monitoring stack (Prometheus/Grafana or Splunk). Validate dashboards |

### Phase 1: Non-Production Migration (Weeks 5-8)

**Objective**: Migrate dev/staging virtual servers to F5, validate functional parity.

| Task | Detail |
|------|--------|
| Config extraction | Export NetScaler `ns.conf` from non-prod pairs. Run through F5 Journeys tool to generate baseline AS3 |
| Manual review | Review each virtual server's content-switching policies, SSL profiles, monitor bindings, and WAF policies. Map to F5 LTM policies, SSL profiles, and ASM policies |
| SSL certificate migration | Export certs/keys from NetScaler. Import into F5 (or integrate with existing cert management -- e.g., HashiCorp Vault, Venafi). Validate SNI mappings |
| WAF policy migration | Recreate NetScaler AppFW profiles as F5 ASM policies. Start in "transparent" (logging-only) mode. Tune false positives |
| Deploy AS3 declarations | Apply per-application AS3 declarations to F5 VEs. Validate L7 routing, SSL termination, pool member health |
| Parallel traffic testing | Use DNS weighting or ACI policy-based redirect to send a percentage of non-prod traffic to F5 while keeping NetScaler active. Compare response behavior |
| Health monitor validation | Verify all health monitors are firing correctly. Pay special attention to ECV equivalents (send-string/receive-string on F5 HTTP monitors) |
| K8s ingress evaluation | Deploy F5 CIS in non-prod K8s cluster alongside existing CPX. Validate Ingress resource handling |

### Phase 2: Production Migration -- Low-Risk Services (Weeks 9-14)

**Objective**: Migrate production virtual servers starting with lowest-risk, lowest-traffic services.

| Task | Detail |
|------|--------|
| Service inventory & prioritization | Rank 30 virtual servers by risk (traffic volume, business criticality, complexity). Migrate simplest first |
| Batch 1 (10 services) | Migrate simple L7 VIPs with basic content switching. Use DNS cutover (short TTL) or ACI contract update to redirect client EPG traffic to F5 service graph |
| Validation | Full functional testing per service. Monitor error rates, latency percentiles, WAF false positives |
| Rollback plan | Keep NetScaler config intact. Rollback = revert DNS or ACI contract to point at NetScaler. Target rollback time: < 5 minutes |
| Batch 2 (10 services) | Migrate medium-complexity services (multiple pools, complex L7 rules, mTLS) |
| Batch 3 (10 services) | Migrate remaining services including highest-traffic / most complex |

### Phase 3: GSLB Migration (Weeks 12-16, overlapping with Phase 2)

**Objective**: Migrate GSLB from NetScaler to F5 BIG-IP DNS.

| Task | Detail |
|------|--------|
| Deploy BIG-IP DNS module | Enable DNS module on designated VE pair (or deploy dedicated DNS VEs if load warrants) in each data center |
| Recreate GSLB config | Map NetScaler GSLB vservers, services, and sites to F5 wide-IPs, pools, and data centers. Configure GTM monitors |
| DNS delegation | Update authoritative DNS delegation for GSLB-managed FQDNs to point at F5 DNS listeners. Use short TTL during transition |
| Validate failover | Simulate DC failure; verify F5 DNS steers traffic to surviving DC within expected TTL window |
| Remove NetScaler GSLB | Once F5 DNS is stable (2+ weeks), decommission NetScaler GSLB configuration |

### Phase 4: Kubernetes Ingress Migration (Weeks 14-18)

**Objective**: Replace NetScaler CPX + CIC with F5 CIS (or F5 NGINX Ingress Controller).

| Task | Detail |
|------|--------|
| Evaluate CIS vs NGINX IC | CIS programs an external BIG-IP (centralized model). NGINX IC runs in-cluster (distributed model). Choose based on architecture preference |
| Deploy in production K8s | Install CIS via Helm. Configure it to watch Ingress/Route resources and program BIG-IP VE pools |
| Migrate ingress resources | Update Ingress annotations from Citrix-specific to F5-specific. Test each service endpoint |
| Decommission CPX | Remove CPX DaemonSets/Deployments and CIC once all ingress traffic is flowing through F5 |

### Phase 5: Decommission & Optimization (Weeks 18-22)

| Task | Detail |
|------|--------|
| Decommission NetScaler VPX | Shut down NetScaler VMs after 2-week burn-in with no traffic. Remove ACI service graph objects |
| Cancel NetScaler licenses | Terminate Citrix support/subscription contracts |
| Optimize F5 | Right-size VE throughput licenses based on observed traffic. Tune ASM policies out of transparent mode into blocking mode |
| Documentation | Update runbooks, on-call procedures, architecture diagrams |
| IaC finalization | Ensure 100% of F5 config is in Terraform/AS3. No manual "ClickOps" config drift |

---

## 6. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| F5 VE performance on KVM is insufficient for peak traffic | Low | High | Lab-test with realistic traffic profiles before production. Use SR-IOV or DPDK for NIC passthrough if needed |
| iRule conversion errors from NetScaler policies | Medium | Medium | Dedicated review of each iRule. Parallel traffic testing in Phase 1 catches logic errors |
| WAF false positives after ASM migration | High | Medium | Run ASM in transparent mode for 2+ weeks per service. Use learning suggestions to tune |
| ACI Service Graph complexity delays timeline | Medium | Medium | Engage Cisco TAC / F5 professional services for ACI integration. Use validated design guides |
| Team lacks F5 expertise | Medium | High | Budget for F5 Certified Administrator training (2-3 engineers). Consider F5 professional services for Phase 0-1 |
| GSLB DNS propagation causes brief outage during cutover | Low | High | Use short TTLs (60s) starting 48 hours before migration. Test with synthetic monitors |
| Broadcom/CSG raises NetScaler renewal price during migration, creating urgency | Medium | Low | Migration plan already addresses this. Accelerate timeline if needed |

---

## 7. Automation Architecture (Target State with F5)

```
+--------------------+       +---------------------+
|   Git Repository   |       |   HashiCorp Vault   |
|  (AS3 Declarations |       |  (SSL Certs, Keys)  |
|   + Terraform HCL) |       +----------+----------+
+---------+----------+                  |
          |                             |
          v                             v
+---------+----------+       +----------+----------+
|   CI/CD Pipeline   +------>+   Terraform Apply   |
| (GitLab CI/GitHub  |       | (OpenStack + F5 +   |
|  Actions/Jenkins)  |       |  ACI providers)     |
+---------+----------+       +----------+----------+
          |                             |
          v                             v
+---------+--------------------------+--+----------+
|              Cisco APIC                           |
|  (ACI Service Graph + L4-L7 Device Package)       |
+---+---------------------------+-------------------+
    |                           |
    v                           v
+---+----------+      +---------+--------+
| F5 BIG-IP VE |      | F5 BIG-IP VE     |
| Pair (DC1)   |      | Pair (DC2)        |
| LTM+DNS+ASM  |      | LTM+DNS+ASM      |
+---+----------+      +---------+---------+
    |                           |
    v                           v
+---+---------------------------+---------+
|          Application Pools              |
|  (OpenStack VMs / K8s Pods via CIS)     |
+-----------------------------------------+
```

### CI/CD Workflow

1. Engineer commits AS3 declaration change to Git branch.
2. CI pipeline runs `terraform plan` showing delta.
3. Peer review + approval.
4. Merge triggers `terraform apply` -- F5 provider pushes AS3 declaration; ACI provider updates contracts/service graph if needed.
5. Telemetry Streaming pushes health data to Prometheus. Automated rollback if error rate exceeds threshold (optional, via pipeline).

---

## 8. ACI Service Graph Detail

### F5 Device Package Integration Steps

1. **Download F5 ACI Device Package** from F5 Downloads (matching BIG-IP version).
2. **Import into APIC**: Fabric > L4-L7 Services > Packages > Import.
3. **Create L4-L7 Device**: Register each F5 VE HA pair as a "device cluster" in APIC, specifying management IP, credentials, and interface mappings (internal/external legs).
4. **Create Service Graph Template**: Define the graph with an F5 function node. Select GoTo (one-arm / Source NAT) or GoThrough (inline / routed) mode.
5. **Apply Service Graph to Contract**: Attach the graph to the contract between consumer and provider EPGs. APIC auto-provisions BDs, VLANs, and redirect policies.
6. **Push L4-L7 Parameters**: APIC sends pool/VIP/profile parameters to F5 via the device package API. Alternatively, manage these via AS3 and let ACI handle only the network stitching (hybrid model -- recommended for flexibility).

### Recommended ACI Model: Hybrid

- **ACI manages**: Network stitching (VLAN allocation, EPG association, PBR/service graph redirect).
- **F5 AS3 manages**: Application-layer config (VIPs, pools, iRules, SSL profiles, ASM policies).
- **Benefit**: Decouples application delivery config from network fabric config. Each team (NetOps vs. AppDelivery) manages their domain.

---

## 9. Decision Summary

| | Recommended? | Rationale |
|---|---|---|
| **F5 BIG-IP VE** | **Yes** (Primary) | Best feature parity, strongest ACI integration, mature automation, acceptable TCO premium for reduced operational risk |
| **HAProxy/Octavia** | No (unless constraints change) | GSLB and WAF gaps are disqualifying for this environment. Poor ACI Service Graph support. Lower cost does not offset operational complexity of multi-system architecture |
| **NetScaler (Stay)** | Conditional | Acceptable short-term if Broadcom licensing is negotiated favorably. Not recommended long-term due to vendor risk, shrinking ecosystem, and weaker ACI integration |

### Recommended Timeline

- **Phase 0-1**: Months 1-2 (foundation + non-prod)
- **Phase 2**: Months 2-4 (production migration in batches)
- **Phase 3-4**: Months 3-5 (GSLB + K8s, overlapping with Phase 2)
- **Phase 5**: Month 5-6 (decommission + optimize)
- **Total**: ~6 months for full migration, including burn-in and decommission

### Immediate Next Steps

1. **Request F5 VE trial licenses** -- validate performance on your OpenStack/KVM environment.
2. **Engage F5 SE** -- request ACI-validated design review specific to your APIC version and fabric topology.
3. **Export NetScaler configs** -- run `show ns runningConfig` on all pairs; feed into F5 Journeys tool for initial conversion assessment.
4. **Identify 3 pilot services** -- low-risk, well-understood VIPs for Phase 1 proof of concept.
5. **Budget approval** -- present TCO comparison to leadership with emphasis on vendor risk mitigation (Broadcom) and operational simplification.
