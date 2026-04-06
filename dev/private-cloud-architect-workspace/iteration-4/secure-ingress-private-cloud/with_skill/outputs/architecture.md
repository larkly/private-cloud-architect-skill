# Secure Ingress Architecture for Healthcare SaaS on Private OpenStack Cloud

## Executive Summary

This document defines a defense-in-depth ingress architecture for exposing a patient portal and REST API to the internet from a private OpenStack cloud running Kubernetes workloads. The design enforces HIPAA compliance, leverages Cisco ACI for network segmentation, prioritizes FLOSS tooling, and addresses the organization's history of DDoS attacks and SQL injection incidents.

The architecture follows the principle: **Internet -> DDoS Mitigation -> Perimeter Firewall -> DMZ -> WAF/Reverse Proxy -> Internal Firewall -> Application Tier -> Data Tier**, with each zone independently controlled and monitored.

---

## 1. Network Zones and DMZ Design

### 1.1 Zone Architecture

The network is divided into five security zones, each implemented as a separate Cisco ACI EPG (Endpoint Group) with contracts enforcing strict inter-zone traffic policies.

```
                                   INTERNET
                                      |
                              [DDoS Mitigation]
                                      |
                            +---------+---------+
                            |  ZONE 1: PERIMETER |
                            |  (OPNsense HA pair)|
                            +---------+---------+
                                      |
                            +---------+---------+
                            |   ZONE 2: DMZ      |
                            | WAF + Reverse Proxy |
                            | API Gateway         |
                            +---------+---------+
                                      |
                            +---------+---------+
                            |  ZONE 3: INTERNAL   |
                            |  FIREWALL           |
                            |  (OPNsense HA pair) |
                            +---------+---------+
                                      |
                     +----------------+----------------+
                     |                                 |
           +---------+---------+             +---------+---------+
           | ZONE 4: APP TIER  |             | ZONE 5: DATA TIER |
           | Kubernetes cluster |             | PostgreSQL DBs    |
           | (patient portal,   |             | (no internet      |
           |  REST API backends)|             |  route, no DMZ    |
           +-------------------+             |  route)           |
                                             +-------------------+
```

### 1.2 Cisco ACI Implementation

Each zone maps to a dedicated ACI construct:

| Zone | ACI Construct | VRF | Bridge Domain | EPG |
|------|--------------|-----|---------------|-----|
| Perimeter | Tenant: `healthcare-prod` | `vrf-perimeter` | `bd-perimeter` | `epg-perimeter-fw` |
| DMZ | Tenant: `healthcare-prod` | `vrf-dmz` | `bd-dmz` | `epg-dmz-ingress` |
| Application | Tenant: `healthcare-prod` | `vrf-internal` | `bd-app` | `epg-app-tier` |
| Data | Tenant: `healthcare-prod` | `vrf-internal` | `bd-data` | `epg-data-tier` |
| Management | Tenant: `healthcare-prod` | `vrf-mgmt` | `bd-mgmt` | `epg-mgmt` |

**ACI Contracts** enforce zone-to-zone communication:

- `contract-internet-to-dmz`: Allow TCP 443 only from perimeter firewall to DMZ WAF/proxy
- `contract-dmz-to-app`: Allow only specific application ports (e.g., TCP 8080, 8443) from DMZ reverse proxy to application tier Kubernetes ingress
- `contract-app-to-data`: Allow TCP 5432 (PostgreSQL) only from specific application pods to database hosts
- `contract-mgmt`: Allow SSH (TCP 22), HTTPS (TCP 443) from management zone bastion hosts only
- **No contract exists between DMZ and Data tier** -- this traffic path is architecturally impossible

**Key ACI design decisions:**

- Separate VRFs for perimeter/DMZ and internal zones enforce routing isolation at the fabric level; traffic between VRFs must traverse a firewall
- ACI microsegmentation contracts use specific filters (protocol, port, source EPG, destination EPG) rather than broad "permit any" rules
- The ACI Kubernetes CNI plugin (`aci-containers-controller`) integrates Kubernetes network policies directly into the ACI fabric, so pod-level microsegmentation is enforced in hardware

### 1.3 DMZ Design Specifics

The DMZ is a dedicated network segment that hosts only internet-facing proxy and security components. No application logic or data resides here.

**DMZ hosts only:**
- HAProxy (load balancer / reverse proxy)
- Coraza WAF (ModSecurity-compatible FLOSS WAF)
- Apache APISIX (API gateway)
- Suricata IDS/IPS sensor

**DMZ design rules:**
- DMZ servers run a minimal, hardened OS (e.g., Flatcar Container Linux or hardened Debian)
- No database clients or drivers installed on DMZ hosts
- No direct SSH access; management only via bastion host in the management zone
- All DMZ hosts are stateless and replaceable; configuration managed by Ansible
- DMZ servers have no route to the data tier; the ACI fabric enforces this at the network level
- All logs are shipped to the monitoring stack in the management zone via a one-way log forwarding path

---

## 2. Web Application Firewall (WAF)

### 2.1 Technology Selection: Coraza WAF

**Coraza** is a FLOSS WAF engine (Apache 2.0 license) compatible with the OWASP Core Rule Set (CRS). It is the successor to the ModSecurity ecosystem and is a CNCF-associated project.

Deployment: Coraza runs as a plugin within the HAProxy or Envoy reverse proxy in the DMZ, or as a sidecar/middleware in the Kubernetes ingress path.

### 2.2 WAF Rule Configuration

**OWASP Core Rule Set (CRS) v4.x** provides the baseline ruleset. Given the SQL injection history, additional hardening is applied:

| Rule Category | CRS Paranoia Level | Custom Additions |
|--------------|-------------------|------------------|
| SQL Injection (SQLi) | Level 3 (strict) | Custom rules blocking UNION-based, blind, and time-based SQLi patterns specific to PostgreSQL syntax |
| Cross-Site Scripting (XSS) | Level 2 | Additional rules for healthcare-specific input fields (patient names, medical record numbers) |
| Remote Code Execution | Level 2 | Block all non-JSON/non-form content types on API endpoints |
| Local File Inclusion | Level 2 | Default CRS |
| Request size limits | Custom | Max body size 1MB for portal, 5MB for file upload endpoints (with separate upload path) |
| Rate limiting | Custom | Per-IP rate limits at WAF layer as secondary defense |

**WAF operational procedures:**
- CRS rules are updated monthly from the upstream OWASP repository
- New rules are tested in detection-only mode for 7 days before enforcement
- False positives are handled via targeted rule exclusions (not by lowering paranoia level)
- WAF logs are forwarded to the SIEM for correlation with application logs

### 2.3 WAF Placement

```
Internet -> OPNsense Perimeter FW -> HAProxy (TLS termination) -> Coraza WAF (inline) -> Re-encrypt -> Internal FW -> K8s Ingress
```

The WAF inspects decrypted HTTP traffic after TLS termination at the HAProxy layer. Traffic is re-encrypted with internal TLS certificates before crossing the internal firewall into the application tier.

---

## 3. API Gateway

### 3.1 Technology Selection: Apache APISIX

**Apache APISIX** (Apache 2.0 license, FLOSS) is selected as the API gateway. It provides:

- Rate limiting, request throttling, and circuit breaking
- JWT and OAuth2 token validation
- Request/response transformation and validation
- Plugin ecosystem for custom logic
- OpenTelemetry integration for distributed tracing

### 3.2 API Gateway Configuration

**Authentication and Authorization:**
- All REST API requests must carry a valid JWT or OAuth2 bearer token
- APISIX validates tokens against the Keycloak identity provider (FLOSS, deployed in the application tier)
- Token introspection and RBAC enforcement at the gateway level before requests reach backend services
- API keys for service-to-service integrations (partner systems) are managed via APISIX's key-auth plugin with per-consumer rate limits

**Rate Limiting (critical given DDoS history):**

| Endpoint Category | Rate Limit | Burst | Action on Exceed |
|-------------------|-----------|-------|------------------|
| Patient portal pages | 100 req/min per IP | 20 | Return 429, log, trigger alert if sustained |
| REST API (authenticated) | 600 req/min per API key | 50 | Return 429, log |
| REST API (unauthenticated, e.g., login) | 20 req/min per IP | 5 | Return 429, temporary IP block after 3 violations |
| Health check / public endpoints | 30 req/min per IP | 10 | Return 429 |

**Request Validation:**
- JSON Schema validation on all API request bodies (reject malformed input before it reaches application code)
- Maximum URL length: 2048 characters
- Allowed HTTP methods: GET, POST, PUT, PATCH, DELETE only (block TRACE, OPTIONS in production)
- Content-Type enforcement: only `application/json` and `application/x-www-form-urlencoded` accepted on API routes
- Request header size limit: 8KB

**API Versioning and Routing:**
- APISIX routes `/api/v1/*` and `/api/v2/*` to appropriate backend Kubernetes services
- Patient portal routes (`/portal/*`) are routed to the portal frontend service
- All other paths return 404 (no path enumeration)

### 3.3 API Gateway Placement

APISIX runs in the DMZ alongside HAProxy and the Coraza WAF. The traffic flow is:

```
HAProxy (TLS term + Coraza WAF) -> APISIX (auth, rate limit, validation) -> Internal FW -> K8s backend services
```

---

## 4. DDoS Mitigation

Given the organization's history of DDoS attacks, a layered DDoS defense is essential.

### 4.1 Layer 3/4 DDoS Protection

**Upstream transit provider scrubbing:**
- Engage the upstream ISP/transit provider for volumetric DDoS scrubbing (BGP-based traffic diversion during attacks)
- Establish pre-arranged BGP communities or Flowspec rules with the provider for rapid activation
- This handles volumetric floods (UDP amplification, SYN floods) before they saturate the WAN link

**OPNsense perimeter firewall:**
- SYN proxy / SYN cookies enabled to mitigate SYN flood attacks
- Connection rate limiting per source IP
- GeoIP filtering: block traffic from countries where no patients or users are expected (reduce attack surface)
- Bogon and RFC1918 filtering on the WAN interface
- Maximum connections per source IP: 100 concurrent

### 4.2 Layer 7 DDoS Protection

- APISIX rate limiting (see Section 3.2) handles application-layer floods
- Coraza WAF blocks malicious request patterns
- **CrowdSec** (FLOSS, MIT license) deployed for collaborative threat intelligence:
  - CrowdSec agent on the HAProxy/APISIX hosts analyzes access logs in real time
  - Known attacker IPs are blocked at the firewall level via the CrowdSec bouncer for OPNsense
  - CrowdSec's community blocklists provide crowd-sourced threat intelligence

### 4.3 DDoS Response Runbook

1. **Detection**: Alertmanager fires alert when request rate exceeds 10x baseline or error rate exceeds 20%
2. **Triage**: On-call engineer confirms attack via Grafana dashboards (traffic source, pattern, affected endpoints)
3. **Automated response**: CrowdSec automatically blocks identified attacker IPs within 30 seconds
4. **Escalation**: If volumetric, contact transit provider to activate scrubbing (target: < 15 minutes)
5. **Manual intervention**: If application-layer, deploy targeted WAF rules and tighten rate limits via APISIX
6. **Recovery**: Monitor for 1 hour after attack subsides, gradually relax emergency rules
7. **Post-incident**: Document in incident log, update CrowdSec scenarios, review WAF rules

---

## 5. Network Segmentation

### 5.1 Kubernetes Network Policies

The Kubernetes cluster uses Cilium as the CNI (integrated with Cisco ACI via the ACI-CNI plugin for fabric-level policy enforcement).

**Default-deny policy** applied to all namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <every-namespace>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Explicit allow policies per service:**

| Source | Destination | Port | Purpose |
|--------|------------|------|---------|
| `namespace: ingress` (nginx-ingress controller) | `namespace: patient-portal` | 8443 | Portal frontend |
| `namespace: ingress` (nginx-ingress controller) | `namespace: api` | 8443 | REST API services |
| `namespace: patient-portal` | `namespace: api` | 8443 | Portal calling API backend |
| `namespace: api` | `namespace: database-proxy` | 5432 | API to PgBouncer/PostgreSQL |
| `namespace: database-proxy` | PostgreSQL hosts (external to K8s) | 5432 | PgBouncer to database |
| All namespaces | `namespace: monitoring` | 9090, 3100 | Prometheus scrape, Loki push |
| None (no ingress from any pod) | Internet | * | **Egress to internet blocked for all application pods** |

### 5.2 Database Isolation

PostgreSQL databases run on dedicated hosts (not inside Kubernetes) in the data tier:

- **No internet route**: The data tier VRF has no default route to the internet. It cannot initiate or receive internet connections.
- **No DMZ route**: No ACI contract permits traffic between DMZ and data tier.
- **Access only from application tier**: Only the PgBouncer connection pooler (running in Kubernetes `database-proxy` namespace) can reach PostgreSQL on TCP 5432.
- **Encryption in transit**: TLS 1.3 enforced on all PostgreSQL connections (`sslmode=verify-full` on clients).
- **Encryption at rest**: LUKS full-disk encryption on all database server volumes.
- **Authentication**: SCRAM-SHA-256 authentication (not md5). Credentials stored in HashiCorp Vault and injected via External Secrets Operator.
- **Audit logging**: PostgreSQL `pgaudit` extension enabled, logging all DDL and DML operations on tables containing PHI.

### 5.3 Management Plane Isolation

- Management traffic (SSH, Kubernetes API, IPMI/Redfish, ACI APIC) runs on a dedicated management VRF (`vrf-mgmt`)
- Management VRF is not routable from any application, DMZ, or internet-facing zone
- All administrative access requires:
  1. VPN connection to the management network (WireGuard, FLOSS)
  2. Authentication to the bastion host with MFA (FreeIPA + TOTP)
  3. Session recording via `tlog` (FLOSS session recording for audit)
- The Kubernetes API server is only accessible from the management network; it is not exposed to the DMZ or internet

---

## 6. TLS and Certificate Management

### 6.1 TLS Architecture

| Segment | TLS Version | Certificate Source | Key Size |
|---------|------------|-------------------|----------|
| Internet to HAProxy (perimeter) | TLS 1.2 + 1.3 | Let's Encrypt (public CA) via cert-manager | RSA 4096 or ECDSA P-384 |
| HAProxy to APISIX (within DMZ) | TLS 1.3 | Internal CA (step-ca, FLOSS) | ECDSA P-256 |
| APISIX to K8s Ingress (DMZ to app) | TLS 1.3 | Internal CA (step-ca) | ECDSA P-256 |
| K8s service-to-service | mTLS via Cilium | Internal CA (cert-manager) | ECDSA P-256 |
| App to PostgreSQL | TLS 1.3 | Internal CA (step-ca) | ECDSA P-256 |

- **step-ca** (Smallstep, Apache 2.0 license) serves as the internal PKI / certificate authority
- cert-manager handles automatic certificate issuance and renewal for all Kubernetes workloads
- External certificates (Let's Encrypt) are renewed automatically with 30-day lead time
- Internal certificates have 90-day lifetimes with automatic rotation
- TLS cipher suites are restricted to AEAD ciphers only (AES-GCM, ChaCha20-Poly1305)

---

## 7. HIPAA Compliance Controls

The ingress architecture implements the following HIPAA Security Rule requirements:

| HIPAA Requirement | Implementation |
|-------------------|---------------|
| Access Control (164.312(a)) | Keycloak RBAC, APISIX JWT validation, Cilium network policies, ACI contracts |
| Audit Controls (164.312(b)) | pgaudit, WAF logs, APISIX access logs, Suricata alerts, all forwarded to Wazuh SIEM |
| Integrity Controls (164.312(c)) | WAF input validation, API schema validation, TLS everywhere, LUKS encryption at rest |
| Transmission Security (164.312(e)) | TLS 1.3 on all segments, mTLS for service-to-service, no unencrypted traffic between zones |
| Person/Entity Authentication (164.312(d)) | MFA via FreeIPA + TOTP for admin access, OAuth2/JWT for API consumers |
| PHI Minimum Necessary | API gateway enforces field-level filtering; only required PHI fields returned per endpoint |
| BAA Compliance | No PHI leaves the private cloud; no third-party cloud services process PHI |

**HIPAA audit logging specifics:**
- All access to PHI-containing API endpoints is logged with: timestamp, authenticated user/API key, source IP, endpoint, HTTP method, response code
- Logs are immutable (append-only storage), retained for 6 years per HIPAA retention requirements
- Log integrity is verified via SHA-256 checksums on daily log bundles

---

## 8. Monitoring, Logging, and Alerting

### 8.1 Monitoring Stack (all FLOSS)

| Component | Tool | Purpose |
|-----------|------|---------|
| Metrics | Prometheus + Thanos | Infrastructure and application metrics with long-term retention |
| Logs | Loki | Centralized log aggregation |
| Tracing | Tempo | Distributed request tracing |
| Dashboards | Grafana | Unified visualization |
| Alerting | Alertmanager | Alert routing and deduplication |
| SIEM | Wazuh | Security event correlation, compliance reporting |
| IDS/IPS | Suricata | Network intrusion detection at perimeter |
| Threat Intel | CrowdSec | Collaborative IP reputation and automated blocking |
| Network Monitoring | LibreNMS | Switch, firewall, and ACI fabric health |

### 8.2 Ingress-Specific Dashboards

**Grafana dashboards for the ingress path:**

1. **Perimeter Overview**: Requests/sec, bandwidth, connection count, GeoIP distribution, DDoS indicators
2. **WAF Dashboard**: Blocked requests by rule category (SQLi, XSS, RCE), false positive rate, CRS rule hit distribution
3. **API Gateway Dashboard**: Request rate by endpoint, latency percentiles (p50, p95, p99), error rates by status code, rate limit triggers, authentication failures
4. **Network Segmentation Health**: ACI contract hit counts, Cilium network policy drops, inter-zone traffic flows
5. **Database Access Audit**: Connection count to PostgreSQL, query latency, pgaudit event counts, failed authentication attempts
6. **Certificate Expiry**: Days until expiry for all TLS certificates across zones

### 8.3 Critical Alerts

| Alert | Condition | Severity | Response |
|-------|-----------|----------|----------|
| DDoS Detected | Request rate > 10x baseline for 5 min | Critical | Page on-call, activate DDoS runbook |
| SQLi Attempt Spike | WAF SQLi blocks > 50/min | High | Investigate source IPs, verify WAF rules, check application logs |
| Certificate Expiry | Any cert < 14 days to expiry | High | Investigate cert-manager, manual renewal if needed |
| API Auth Failures | > 100 failed auth attempts/min from single IP | High | Auto-block via CrowdSec, investigate for credential stuffing |
| Database Connection Anomaly | PostgreSQL connections from unexpected source | Critical | Immediate investigation, potential network policy breach |
| ACI Contract Violation | Traffic matching no contract (implicit deny hit) | Medium | Investigate for misconfiguration or lateral movement attempt |
| WAF Rule Update Failed | CRS update cron failed | Medium | Manual update, verify rule integrity |
| Firewall HA Failover | OPNsense CARP failover event | High | Verify failover, investigate root cause of primary failure |

### 8.4 Log Pipeline

```
All sources -> Fluentd/Fluent Bit (log collector) -> Loki (storage/query)
                                                  -> Wazuh (SIEM correlation)
                                                  -> Long-term archive (S3-compatible MinIO, immutable bucket)
```

- Logs are shipped in real time (< 30 second delay)
- Wazuh correlates WAF, firewall, IDS, API gateway, and application logs for security events
- MinIO stores immutable log archives for HIPAA's 6-year retention requirement
- Log access is restricted to the security team and auditors via RBAC

---

## 9. Incident Response

### 9.1 Incident Classification

| Severity | Definition | Response Time | Examples |
|----------|-----------|---------------|----------|
| P1 - Critical | Active breach, PHI exposure, service down | 15 min | Data exfiltration detected, ransomware, complete service outage |
| P2 - High | Active attack, no confirmed breach | 30 min | Sustained DDoS, credential stuffing, SQLi attempts bypassing WAF |
| P3 - Medium | Suspicious activity, no active impact | 4 hours | Unusual traffic patterns, failed penetration attempts, single WAF bypass |
| P4 - Low | Informational, potential future risk | Next business day | Vulnerability scan results, configuration drift detected |

### 9.2 Incident Response Procedures

**Phase 1: Detection and Triage (0-15 minutes)**
- Automated detection via Wazuh SIEM correlation rules, Suricata IDS alerts, CrowdSec detections
- Alertmanager pages on-call engineer with severity and initial context
- On-call triages via Grafana dashboards and Wazuh alert details
- Classify incident severity per table above

**Phase 2: Containment (15-60 minutes)**
- **Network isolation**: Block attacker IPs at OPNsense perimeter firewall via CrowdSec bouncer or manual rule
- **Kubernetes isolation**: Apply emergency NetworkPolicy to isolate compromised namespace
- **ACI emergency contract**: APIC can push emergency deny contracts to isolate an entire EPG within seconds
- **Credential rotation**: If credentials compromised, rotate via Vault immediately; revoke all active sessions in Keycloak
- **Database lockdown**: If SQLi confirmed, immediately restrict database access to read-only; activate PgBouncer pause to halt all queries while investigating

**Phase 3: Eradication and Recovery (1-24 hours)**
- Identify root cause (WAF bypass, unpatched vulnerability, misconfiguration, insider)
- Deploy fix (WAF rule update, application patch, network policy correction)
- Restore from known-good state if system integrity is uncertain (immutable infrastructure: redeploy from Ansible/ArgoCD)
- Verify fix via targeted testing

**Phase 4: Post-Incident (24-72 hours)**
- Blameless post-mortem document
- Timeline reconstruction from Wazuh/Loki logs
- HIPAA breach assessment: determine if PHI was accessed/exposed; if yes, initiate HIPAA breach notification process (60-day deadline for HHS notification)
- Update WAF rules, network policies, and monitoring alerts based on lessons learned
- Update incident response runbooks
- Present findings to security team and management

### 9.3 SQL Injection-Specific Response (given prior incident)

Because the organization experienced an SQL injection incident, the following specific controls are in place:

**Prevention:**
- Coraza WAF with CRS Paranoia Level 3 for SQLi rules (strictest practical level)
- APISIX JSON Schema validation rejects malformed input before it reaches application code
- Application code uses parameterized queries exclusively (enforced via code review and SAST scanning)
- PostgreSQL user privileges follow least-privilege: application database users have only SELECT/INSERT/UPDATE/DELETE on specific tables, no DDL privileges, no access to `pg_catalog` or system tables

**Detection:**
- Wazuh has custom SQLi detection rules correlating WAF blocks with application error logs
- pgaudit logs all SQL statements; anomalous query patterns (e.g., UNION SELECT, information_schema queries) trigger alerts
- Suricata rules detect SQL injection patterns in network traffic as a secondary detection layer

**Response:**
- If SQLi bypasses WAF: immediately add targeted WAF rule, activate PgBouncer pause, investigate scope
- Forensic queries against pgaudit logs to determine exactly what data was accessed
- If PHI accessed: initiate HIPAA breach notification process

### 9.4 Automated Response Capabilities

Event-Driven Ansible (EDA) provides automated response for well-understood attack patterns:

| Trigger | Automated Action |
|---------|-----------------|
| CrowdSec detection of known attacker IP | Add to OPNsense block list via Ansible |
| WAF SQLi blocks > threshold | Temporarily increase WAF paranoia level, alert security team |
| Certificate expiry < 7 days | Force cert-manager renewal, alert if renewal fails |
| ACI implicit deny hit from application tier | Isolate source pod, create forensic snapshot, alert |
| PostgreSQL authentication failure spike | Temporarily block source IP at network level, alert DBA |

---

## 10. Infrastructure as Code

All components in this architecture are defined and deployed via IaC:

| Component | Tool | Repository |
|-----------|------|-----------|
| OPNsense firewall rules | Ansible (`community.general.opnsense`) | `infra-firewall` |
| Cisco ACI configuration | Ansible (`cisco.aci` collection) + OpenTofu (`aci` provider) | `infra-aci` |
| HAProxy + Coraza WAF | Ansible roles | `infra-dmz` |
| Apache APISIX | Helm chart via ArgoCD | `k8s-platform` |
| Kubernetes network policies | ArgoCD (GitOps) | `k8s-policies` |
| Cilium configuration | Helm chart via ArgoCD | `k8s-platform` |
| Prometheus/Grafana/Loki | Helm charts via ArgoCD | `k8s-monitoring` |
| Wazuh | Ansible roles | `infra-siem` |
| PostgreSQL | Ansible roles | `infra-database` |
| cert-manager + step-ca | Helm charts via ArgoCD | `k8s-platform` |
| CrowdSec | Ansible roles (DMZ) + Helm chart (K8s) | `infra-dmz` / `k8s-security` |

**GitOps workflow:**
- All changes go through pull requests with mandatory review
- ArgoCD syncs Kubernetes resources from Git (single source of truth)
- Ansible changes are applied via AWX with approval workflows for production
- OpenTofu ACI changes require plan review before apply
- Drift detection runs every 10 minutes; drift triggers an alert and automatic reconciliation

---

## 11. Architecture Decision Records

### ADR-001: FLOSS WAF Selection

**Decision**: Use Coraza WAF with OWASP CRS instead of commercial WAF (F5, Fortinet).

**Rationale**: Coraza is Apache 2.0 licensed, actively maintained as a CNCF-adjacent project, fully compatible with the industry-standard OWASP CRS ruleset, and avoids commercial licensing costs. The ops team's preference for FLOSS aligns with this choice. Commercial WAF would be reconsidered only if Coraza proves insufficient for the traffic volume or if a specific compliance auditor requires a commercial product.

### ADR-002: API Gateway Selection

**Decision**: Use Apache APISIX instead of Kong or commercial API gateways.

**Rationale**: APISIX is Apache 2.0 licensed (fully FLOSS, unlike Kong's dual-license model), offers equivalent functionality for our use case (rate limiting, JWT validation, request transformation), and has strong community adoption. Its etcd-based configuration store integrates well with our existing infrastructure patterns.

### ADR-003: OPNsense over pfSense

**Decision**: Use OPNsense for perimeter and internal firewalls.

**Rationale**: OPNsense is BSD-licensed (FLOSS), has a more active development community, better plugin ecosystem (including CrowdSec bouncer), and a cleaner codebase than pfSense. It provides enterprise-grade stateful firewalling, HA via CARP, and Suricata IDS/IPS integration built-in.

### ADR-004: PostgreSQL Outside Kubernetes

**Decision**: Run PostgreSQL on dedicated hosts, not inside Kubernetes.

**Rationale**: For HIPAA-regulated databases containing PHI, the operational simplicity and proven reliability of PostgreSQL on dedicated hosts outweighs the convenience of Kubernetes operators. Dedicated hosts provide clearer security boundaries, simpler backup/restore procedures, predictable I/O performance, and easier compliance auditing. The data tier's network isolation is also simpler to enforce with dedicated hosts in their own ACI EPG.

### ADR-005: CrowdSec for Collaborative Threat Intelligence

**Decision**: Deploy CrowdSec for distributed threat intelligence and automated blocking.

**Rationale**: CrowdSec (MIT license, FLOSS) provides crowd-sourced IP reputation data and automated blocking, directly addressing the DDoS threat. Unlike Fail2ban (which is single-host), CrowdSec shares threat intelligence across the community and supports bouncers for OPNsense, HAProxy, and Kubernetes. This gives us early warning of attacker IPs before they target our infrastructure.

---

## 12. Component Summary

| Component | Product | License | Zone | Purpose |
|-----------|---------|---------|------|---------|
| Perimeter Firewall | OPNsense (HA pair) | BSD | Perimeter | Stateful firewall, GeoIP filtering, SYN protection |
| Internal Firewall | OPNsense (HA pair) | BSD | Between DMZ and App | Zone separation enforcement |
| WAF | Coraza + OWASP CRS | Apache 2.0 | DMZ | HTTP attack prevention (SQLi, XSS, etc.) |
| Reverse Proxy / LB | HAProxy | GPLv2 | DMZ | TLS termination, load balancing |
| API Gateway | Apache APISIX | Apache 2.0 | DMZ | Rate limiting, auth, request validation |
| IDS/IPS | Suricata | GPLv2 | Perimeter | Network intrusion detection |
| Threat Intel | CrowdSec | MIT | DMZ + Perimeter | Collaborative blocking, IP reputation |
| Kubernetes CNI | Cilium + ACI plugin | Apache 2.0 | App Tier | Pod networking, network policies |
| K8s Ingress Controller | nginx-ingress | Apache 2.0 | App Tier | Internal ingress routing |
| Identity Provider | Keycloak | Apache 2.0 | App Tier | OAuth2/OIDC, user management |
| Internal CA / PKI | step-ca | Apache 2.0 | Mgmt | Internal TLS certificates |
| Certificate Management | cert-manager | Apache 2.0 | App Tier | Automated cert issuance/renewal |
| Connection Pooler | PgBouncer | ISC | App Tier | PostgreSQL connection pooling |
| Database | PostgreSQL | PostgreSQL License | Data Tier | Patient data, application data |
| Database Auditing | pgaudit | PostgreSQL License | Data Tier | SQL audit logging |
| Secrets Management | HashiCorp Vault | BSL* | Mgmt | Credential storage and rotation |
| Metrics | Prometheus + Thanos | Apache 2.0 | Mgmt | Metrics collection and long-term storage |
| Logs | Loki | AGPLv3 | Mgmt | Log aggregation |
| Tracing | Tempo | AGPLv3 | Mgmt | Distributed tracing |
| Dashboards | Grafana | AGPLv3 | Mgmt | Visualization |
| SIEM | Wazuh | GPLv2 | Mgmt | Security event correlation |
| Network Monitoring | LibreNMS | GPLv3 | Mgmt | Infrastructure monitoring |
| Log Archive | MinIO | AGPLv3 | Mgmt | Immutable long-term log storage |
| GitOps | ArgoCD | Apache 2.0 | Mgmt | Kubernetes declarative deployment |
| Automation | Ansible + AWX | GPLv3 | Mgmt | Infrastructure automation |
| IaC | OpenTofu | MPL 2.0 | Mgmt | ACI and infrastructure provisioning |

*Note: HashiCorp Vault uses the Business Source License (BSL), which is not FLOSS. Alternative: OpenBao (FLOSS fork, MPL 2.0) should be evaluated for future migration once it reaches production maturity. For now, Vault is retained due to its proven stability and broad ecosystem integration for HIPAA-sensitive credential management.

---

## 13. Deployment Sequence

1. **Phase 1 - Network Foundation**: Deploy ACI tenant, VRFs, bridge domains, EPGs, and contracts via OpenTofu + Ansible
2. **Phase 2 - Perimeter**: Deploy OPNsense HA pairs (perimeter + internal), configure base firewall rules
3. **Phase 3 - DMZ**: Deploy HAProxy, Coraza WAF, APISIX, Suricata on hardened DMZ hosts via Ansible
4. **Phase 4 - Kubernetes Platform**: Deploy Cilium CNI with ACI integration, nginx-ingress, cert-manager, step-ca
5. **Phase 5 - Data Tier**: Deploy PostgreSQL with pgaudit, LUKS encryption, TLS, Vault-managed credentials
6. **Phase 6 - Security Stack**: Deploy Wazuh SIEM, CrowdSec, configure all log forwarding
7. **Phase 7 - Monitoring**: Deploy Prometheus, Loki, Tempo, Grafana, Alertmanager, configure dashboards and alerts
8. **Phase 8 - Application Deployment**: Deploy patient portal and REST API services via ArgoCD
9. **Phase 9 - Validation**: Penetration testing (internal and third-party), WAF bypass testing, DDoS simulation, HIPAA compliance audit
10. **Phase 10 - Go-Live**: Cutover DNS, enable production traffic, 24/7 monitoring for first 2 weeks
