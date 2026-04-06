# Secure Ingress Architecture for Healthcare SaaS on OpenStack + Kubernetes

## 1. Executive Summary

This document defines a defense-in-depth ingress architecture for a HIPAA-compliant healthcare SaaS platform running on OpenStack with Kubernetes workloads. The design exposes the patient portal and REST API to the internet while isolating backend services and PostgreSQL databases on strictly internal network segments. Every component recommended is FLOSS (Free/Libre Open Source Software), and the architecture directly addresses the prior DDoS and SQL injection incidents.

---

## 2. Network Topology and DMZ Design

### 2.1 Zone Model

The architecture uses four discrete security zones, each mapped to dedicated Cisco ACI Bridge Domains (BDs) and EPGs (Endpoint Groups) with strict inter-zone contracts.

```
                         INTERNET
                            |
                     [ Border Router ]
                            |
                   ========================
                   |   ZONE 1: EDGE DMZ   |    (ACI EPG: epg-edge-dmz)
                   |  - HAProxy (DDoS)    |
                   |  - ModSecurity WAF   |
                   ========================
                            |
                   ========================
                   |  ZONE 2: APP DMZ     |    (ACI EPG: epg-app-dmz)
                   |  - APISIX Gateway    |
                   |  - K8s Ingress       |
                   |  - Patient Portal    |
                   |  - REST API pods     |
                   ========================
                            |
                   ========================
                   |  ZONE 3: INTERNAL    |    (ACI EPG: epg-internal)
                   |  - Backend services  |
                   |  - Service mesh      |
                   ========================
                            |
                   ========================
                   |  ZONE 4: DATA        |    (ACI EPG: epg-data)
                   |  - PostgreSQL        |
                   |  - Backup agents     |
                   ========================
```

### 2.2 OpenStack Network Mapping

| Zone | Neutron Network | Subnet CIDR | VLAN (ACI) | Purpose |
|------|----------------|-------------|------------|---------|
| Edge DMZ | `net-edge-dmz` | 10.100.1.0/24 | VLAN 100 | Reverse proxy, WAF |
| App DMZ | `net-app-dmz` | 10.100.2.0/24 | VLAN 200 | K8s ingress, public-facing pods |
| Internal | `net-internal` | 10.100.10.0/24 | VLAN 300 | Backend microservices |
| Data | `net-data` | 10.100.20.0/24 | VLAN 400 | PostgreSQL, persistent storage |

### 2.3 Cisco ACI Contracts (Inter-Zone Firewall Rules)

ACI contracts enforce microsegmentation at the fabric level, before packets ever reach a host.

```
Contract: edge-to-app
  Provider: epg-app-dmz
  Consumer: epg-edge-dmz
  Subjects/Filters:
    - TCP 443 (HTTPS) PERMIT
    - TCP 8443 (API gateway admin - from jump host only) PERMIT
    - DENY ALL else

Contract: app-to-internal
  Provider: epg-internal
  Consumer: epg-app-dmz
  Subjects/Filters:
    - TCP 8080-8089 (backend service ports) PERMIT
    - TCP 4222 (NATS messaging) PERMIT
    - DENY ALL else

Contract: internal-to-data
  Provider: epg-data
  Consumer: epg-internal
  Subjects/Filters:
    - TCP 5432 (PostgreSQL) PERMIT
    - DENY ALL else

Contract: deny-edge-to-data
  (Explicit) NO contract between epg-edge-dmz and epg-data
  (Explicit) NO contract between epg-app-dmz and epg-data
```

**Key point:** There is no ACI contract permitting traffic from the Edge DMZ or App DMZ directly to the Data zone. The only path to PostgreSQL is from the Internal zone. This is enforced at the fabric level and cannot be bypassed by host-level misconfiguration.

---

## 3. Edge DMZ: DDoS Mitigation and WAF

### 3.1 HAProxy as the Edge Reverse Proxy and DDoS Absorber

HAProxy (FLOSS, GPLv2) serves as the outermost entry point, deployed on dedicated OpenStack instances (not in Kubernetes) for isolation.

**Deployment:** Two HAProxy nodes in active-passive with VRRP (via keepalived) on `net-edge-dmz`.

**DDoS mitigation configuration:**

```haproxy
global
    maxconn 50000
    tune.ssl.default-dh-param 2048

defaults
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    option  httpclose

frontend ft_public
    bind *:443 ssl crt /etc/haproxy/certs/portal.pem alpn h2,http/1.1
    # --- Rate limiting ---
    stick-table type ip size 200k expire 60s store conn_rate(10s),http_req_rate(10s),bytes_out_rate(60s)
    http-request track-sc0 src
    # Reject IPs opening more than 100 connections in 10s
    http-request deny deny_status 429 if { sc_conn_rate(0) gt 100 }
    # Reject IPs making more than 200 HTTP requests in 10s
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 200 }
    # Slowloris protection: tarpit slow clients
    timeout http-request 5s
    # GeoIP blocking (optional, via MaxMind GeoLite2 with haproxy-lua)
    # Drop connections from embargoed countries or non-service regions

    default_backend bk_waf

backend bk_waf
    server waf1 10.100.1.11:8080 check
    server waf2 10.100.1.12:8080 check backup
```

**Additional DDoS measures:**

- **SYN cookies** enabled at the Linux kernel level (`net.ipv4.tcp_syncookies = 1`).
- **Connection limits per source IP** via nftables on the HAProxy hosts as a secondary layer.
- **fail2ban** monitoring HAProxy logs for repeat 429s, auto-blocking at the nftables level.

### 3.2 ModSecurity WAF (with OWASP Core Rule Set)

ModSecurity v3 (FLOSS, Apache 2.0) deployed as a reverse proxy layer behind HAProxy, running on the same Edge DMZ instances or as sidecar containers.

**Why ModSecurity + CRS specifically addresses your SQL injection history:**

The OWASP Core Rule Set (CRS) v4 includes comprehensive SQLi detection rules (rule group 942xxx) that catch:
- Union-based injection
- Boolean-based blind injection
- Time-based blind injection
- Stacked queries
- Common evasion techniques (encoding, comments, case manipulation)

**Deployment model:** ModSecurity integrated with Nginx (libmodsecurity3 + nginx connector) as a reverse proxy.

```nginx
# /etc/nginx/modsecurity.conf
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess Off
SecRequestBodyLimit 1048576
SecRequestBodyNoFilesLimit 131072

# OWASP CRS v4
Include /etc/modsecurity/crs/crs-setup.conf
Include /etc/modsecurity/crs/rules/*.conf

# Custom: extra paranoia for SQL injection (paranoia level 2 for 942xxx rules)
SecAction "id:900000,phase:1,nolog,pass,t:none,setvar:tx.blocking_paranoia_level=2"

# Custom: block requests with SQL keywords in unexpected parameters
SecRule ARGS "@rx (?i)(union\s+select|sleep\s*\(|benchmark\s*\(|load_file|into\s+outfile)" \
    "id:100001,phase:2,deny,status:403,msg:'Custom SQLi pattern blocked',severity:2"

# Audit log for SIEM ingestion
SecAuditEngine RelevantOnly
SecAuditLogRelevantStatus "^(?:5|4(?!04))"
SecAuditLogFormat JSON
SecAuditLogType Serial
SecAuditLog /var/log/modsecurity/audit.json
```

**Tuning process:**
1. Deploy in `SecRuleEngine DetectionOnly` for two weeks.
2. Analyze false positives from audit logs.
3. Add targeted rule exclusions (never disable entire rule groups).
4. Switch to `SecRuleEngine On` (blocking mode).
5. Continuously monitor and tune.

---

## 4. API Gateway: Apache APISIX

### 4.1 Why APISIX

Apache APISIX (FLOSS, Apache 2.0) is the API gateway deployed in the App DMZ Kubernetes namespace. It replaces the need for a commercial API gateway and provides:

- Route-level rate limiting (complementing HAProxy's IP-level limits)
- JWT/OAuth2 validation at the edge of Kubernetes
- Request/response transformation
- Built-in observability (Prometheus metrics, structured logging)
- Plugin architecture for custom security policies

### 4.2 Deployment in Kubernetes

```yaml
# APISIX deployed in namespace: api-gateway (on net-app-dmz)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apisix
  namespace: api-gateway
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: apisix
        image: apache/apisix:3.9-debian
        ports:
        - containerPort: 9080   # HTTP (internal redirect to HTTPS at HAProxy)
        - containerPort: 9443   # HTTPS
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
```

### 4.3 Key APISIX Route Policies

**Patient Portal route:**
```json
{
  "uri": "/portal/*",
  "upstream": {
    "type": "roundrobin",
    "nodes": { "patient-portal-svc.app-dmz.svc.cluster.local:8080": 1 }
  },
  "plugins": {
    "limit-req": { "rate": 50, "burst": 100, "key_type": "var", "key": "remote_addr" },
    "csrf": { "key": "unique-csrf-key" },
    "cors": {
      "allow_origins": "https://portal.example.com",
      "allow_methods": "GET,POST,PUT,DELETE,OPTIONS",
      "allow_headers": "Authorization,Content-Type",
      "max_age": 3600
    }
  }
}
```

**REST API route:**
```json
{
  "uri": "/api/v1/*",
  "upstream": {
    "type": "roundrobin",
    "nodes": { "rest-api-svc.app-dmz.svc.cluster.local:8080": 1 }
  },
  "plugins": {
    "jwt-auth": {},
    "limit-req": { "rate": 100, "burst": 200, "key_type": "var", "key": "consumer_name" },
    "ip-restriction": { "blacklist": [] },
    "request-validation": {
      "body_schema": { "type": "object" },
      "rejected_msg": "Invalid request body"
    }
  }
}
```

**Security plugins enabled globally:**
- `request-id`: Injects a unique X-Request-ID for traceability across all zones.
- `prometheus`: Exposes metrics on a cluster-internal port only.
- `syslog`: Ships access logs to the central SIEM.

---

## 5. Kubernetes Network Segmentation

### 5.1 Namespace Isolation

```
Namespaces:
  api-gateway      -> net-app-dmz    (APISIX, ingress controller)
  patient-portal   -> net-app-dmz    (Portal frontend and BFF)
  rest-api         -> net-app-dmz    (REST API pods)
  backend          -> net-internal   (Business logic services)
  data-access      -> net-internal   (DB connection pooling, PgBouncer)
  monitoring       -> net-internal   (Prometheus, Grafana, Loki)
  security         -> net-internal   (Falco, log shippers)
```

### 5.2 Kubernetes Network Policies (Calico)

Calico (FLOSS, Apache 2.0) is the CNI plugin, chosen for its support of ACI integration and GlobalNetworkPolicy.

**Default deny all ingress and egress in every namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend  # Applied to every namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow API gateway to reach patient-portal and rest-api:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-gateway
  namespace: patient-portal
spec:
  podSelector:
    matchLabels:
      app: patient-portal
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          zone: app-dmz
      podSelector:
        matchLabels:
          app: apisix
    ports:
    - protocol: TCP
      port: 8080
```

**Allow backend to reach PostgreSQL (via data-access namespace only):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-pgbouncer
  namespace: data-access
spec:
  podSelector:
    matchLabels:
      app: pgbouncer
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          zone: internal
      podSelector:
        matchLabels:
          role: backend-service
    ports:
    - protocol: TCP
      port: 6432
```

**Critical: No policy ever allows pods in `api-gateway`, `patient-portal`, or `rest-api` namespaces to reach `data-access` or any database port.** The only path is: API pod -> backend service (internal zone) -> PgBouncer (internal zone) -> PostgreSQL (data zone, via ACI contract).

### 5.3 Service Mesh: Linkerd

Linkerd (FLOSS, Apache 2.0) provides mutual TLS (mTLS) for all pod-to-pod communication within the cluster.

- **Why Linkerd over Istio:** Lighter footprint, simpler operations, no Envoy CVE surface, CNCF graduated project.
- **mTLS everywhere:** All inter-service calls are encrypted with automatically rotated certificates. Even if an attacker gains access to a pod, they cannot sniff traffic between other pods.
- **Authorization policies:** Linkerd `Server` and `ServerAuthorization` resources restrict which service accounts can call which services, independent of network policies (defense in depth).

```yaml
apiVersion: policy.linkerd.io/v1beta3
kind: ServerAuthorization
metadata:
  name: rest-api-to-backend-only
  namespace: backend
spec:
  server:
    name: backend-service
  client:
    meshTLS:
      serviceAccounts:
      - name: rest-api
        namespace: rest-api
```

---

## 6. TLS and Certificate Management

### 6.1 TLS Termination Points

| Hop | TLS Termination | Certificate Source |
|-----|----------------|-------------------|
| Client -> HAProxy | HAProxy terminates TLS 1.3 | cert-manager + Let's Encrypt (or internal CA) |
| HAProxy -> ModSecurity/WAF | Re-encrypted (TLS 1.3 internal) | Internal CA via cert-manager |
| WAF -> APISIX | Re-encrypted (TLS 1.3 internal) | Internal CA via cert-manager |
| APISIX -> App pods | Linkerd mTLS (transparent) | Linkerd trust anchor |
| App -> Backend | Linkerd mTLS | Linkerd trust anchor |
| Backend -> PostgreSQL | TLS 1.3 (verify-full) | Internal CA, client certs |

**cert-manager** (FLOSS, Apache 2.0) manages all certificate lifecycle. PostgreSQL client certificates are issued from a dedicated internal CA and rotated every 90 days.

### 6.2 TLS Hardening

```
# HAProxy TLS configuration
ssl-default-bind-ciphers TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
ssl-default-bind-ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
ssl-default-bind-options ssl-min-ver TLSv1.3 no-tls-tickets
```

---

## 7. Database Protection (PostgreSQL)

Given the prior SQL injection incident, the database layer gets explicit hardened treatment.

### 7.1 Network Isolation

- PostgreSQL runs on dedicated OpenStack instances on `net-data` (VLAN 400).
- The **only** hosts that can reach port 5432 are PgBouncer pods in the `data-access` namespace, enforced by both ACI contracts and OpenStack security groups.
- No Kubernetes pod has a direct route to `net-data`. PgBouncer instances run on nodes with dual-homed NICs (one on `net-internal`, one on `net-data`).

### 7.2 Application-Level Defenses

- **Parameterized queries only:** Enforced via code review policy and CI static analysis (e.g., semgrep rules for raw SQL detection).
- **PgBouncer connection pooling:** Limits max connections, prevents connection exhaustion attacks.
- **PostgreSQL `pg_hba.conf`:** Only accepts connections from PgBouncer IPs with client certificate authentication (`hostssl all all 10.100.10.0/24 cert`).
- **Read replicas** for the patient portal's read-heavy queries, connected via a separate PgBouncer pool with a read-only PostgreSQL role.
- **pgAudit** (FLOSS) enabled for logging all DML/DDL operations for HIPAA audit trail.

### 7.3 Backup Security

- Backups via **pgBackRest** (FLOSS) to an internal object store (OpenStack Swift or MinIO).
- Backup traffic stays on `net-data`; no backup data traverses any DMZ network.
- Backups encrypted at rest with AES-256 (key managed via HashiCorp Vault, FLOSS BSL but can use the older MPL-licensed fork OpenBao if strict FLOSS is required).

---

## 8. Monitoring and Observability

### 8.1 Stack Overview (All FLOSS)

| Component | Role | License |
|-----------|------|---------|
| Prometheus | Metrics collection | Apache 2.0 |
| Grafana | Dashboards and alerting | AGPL 3.0 |
| Loki | Log aggregation | AGPL 3.0 |
| Promtail / Alloy | Log shipping | Apache 2.0 |
| Falco | Runtime security / anomaly detection | Apache 2.0 |
| Suricata | Network IDS on Edge DMZ | GPL 2.0 |
| Wazuh | Host IDS, HIPAA compliance checking | GPL 2.0 |
| Elastiflow / ntopng | NetFlow analysis from ACI | GPL 3.0 |

### 8.2 Key Dashboards and Alerts

**DDoS / Availability Dashboard (Grafana):**
- HAProxy connection rate per source IP (from HAProxy Prometheus exporter)
- HTTP 429 rate (rate-limited requests)
- HAProxy backend queue depth
- Alert: `haproxy_conn_rate_per_ip > 500/min` for more than 2 minutes -> PagerDuty

**WAF Dashboard:**
- ModSecurity blocked requests by rule ID (parsed from JSON audit log via Loki)
- SQLi attempt rate (rule group 942xxx)
- Top blocked source IPs
- Alert: `modsec_sqli_block_rate > 10/min` -> PagerDuty + auto-block IP via fail2ban

**API Gateway Dashboard:**
- APISIX request rate, latency p50/p95/p99 per route
- JWT auth failure rate
- Rate-limit trigger count per consumer
- Alert: `apisix_auth_failure_rate > 50/min` -> Security team Slack + PagerDuty

**Kubernetes Security Dashboard:**
- Falco alerts (unexpected process execution, shell in container, sensitive file access)
- Network policy deny counts (Calico metrics)
- Pod-to-pod traffic anomalies (Linkerd tap metrics)
- Alert: Any Falco `Critical` or `Error` -> Immediate PagerDuty

**Database Dashboard:**
- PostgreSQL active connections, query duration p99
- pgAudit log anomalies (unusual DDL, bulk DELETE/UPDATE)
- PgBouncer pool saturation
- Alert: `pg_connections > 80% max` or `query_duration_p99 > 5s` -> Ops PagerDuty

### 8.3 Suricata Network IDS

Suricata (FLOSS, GPL 2.0) deployed on the Edge DMZ hosts in IDS mode, inspecting traffic between HAProxy and the WAF.

```yaml
# suricata.yaml excerpt
af-packet:
  - interface: eth0  # Edge DMZ interface
    cluster-type: cluster_flow
    defrag: yes
rule-files:
  - suricata.rules           # ET Open ruleset (free)
  - custom-healthcare.rules  # ePHI exfiltration patterns
outputs:
  - eve-log:
      enabled: yes
      filetype: syslog  # Ship to Loki/Wazuh
      types:
        - alert
        - http
        - dns
        - tls
```

**Custom rule example (ePHI exfiltration detection):**
```
alert http any any -> $EXTERNAL_NET any (msg:"Potential ePHI in response body - SSN pattern"; \
  flow:to_client,established; content:"200"; http_stat_code; \
  pcre:"/\b\d{3}-\d{2}-\d{4}\b/R"; sid:1000001; rev:1;)
```

### 8.4 Centralized Audit Logging for HIPAA

All components ship logs to a dedicated Loki instance in the `monitoring` namespace:

- **HAProxy** access logs (who connected, when, from where)
- **ModSecurity** audit logs (blocked attacks, rule matches)
- **APISIX** access logs (API calls, authenticated consumer, response codes)
- **Linkerd** access logs (inter-service calls)
- **pgAudit** logs (all database queries with user attribution)
- **Falco** alerts (runtime anomalies)
- **OS-level** auditd logs (via Wazuh agent)

**Retention:** HIPAA requires 6 years for audit logs. Loki is configured with S3-compatible (MinIO) long-term storage with lifecycle policies. Logs are immutable once written (MinIO object locking).

---

## 9. Incident Response Plan

### 9.1 Classification Matrix

| Severity | Criteria | Response Time | Examples |
|----------|----------|--------------|---------|
| P1 - Critical | Active data breach, ePHI exposure, complete service outage | 15 minutes | SQLi with data exfiltration, ransomware, DB compromise |
| P2 - High | Active attack without confirmed breach, partial outage | 30 minutes | Sustained DDoS, WAF bypass attempt, unauthorized API access |
| P3 - Medium | Anomalous activity, policy violation | 4 hours | Brute-force login attempts, unusual DB query patterns |
| P4 - Low | Informational, minor policy deviation | Next business day | Failed vulnerability scan, single blocked SQLi probe |

### 9.2 DDoS Response Runbook

Given the history of DDoS attacks:

```
TRIGGER: haproxy_conn_rate alert OR manual report of degradation

1. ASSESS (0-5 min)
   - Check Grafana DDoS dashboard: confirm attack pattern
   - Identify source IP ranges (HAProxy stick-table dump)
   - Classify: volumetric, application-layer, or protocol attack

2. MITIGATE (5-15 min)
   - Application-layer DDoS:
     a. Lower HAProxy rate limits: conn_rate threshold from 100 to 20
     b. Enable APISIX emergency rate-limit plugin (pre-configured, disabled by default)
     c. If targeted at specific URI: add HAProxy ACL to block/tarpit
   - Volumetric DDoS:
     a. Engage upstream ISP blackhole routing for attack source CIDRs
     b. If available, activate upstream scrubbing service
     c. As last resort: enable GeoIP blocking for non-service countries
   - Verify mitigation: monitor HAProxy backend response time returning to baseline

3. INVESTIGATE (15-60 min)
   - Correlate with Suricata IDS alerts for embedded attack payloads
   - Check if DDoS is a diversion for another attack (review Falco, WAF logs)
   - Document source IPs, attack vectors, duration

4. RECOVER
   - Gradually restore normal rate limits
   - Add persistent nftables blocks for confirmed attack sources
   - Update Suricata custom rules if new patterns identified

5. POST-INCIDENT
   - Write incident report within 24 hours
   - Update DDoS runbook with lessons learned
   - If ePHI availability was affected >4 hours: HIPAA breach assessment
```

### 9.3 SQL Injection Response Runbook

Given the prior SQL injection incident:

```
TRIGGER: ModSecurity 942xxx alert cluster OR pgAudit anomaly alert

1. ASSESS (0-5 min)
   - Review ModSecurity audit log: was the request blocked or did it pass?
   - If blocked: lower severity, continue monitoring
   - If passed (WAF bypass): ESCALATE TO P1 IMMEDIATELY

2. CONTAIN (5-15 min for P1)
   - If active exploitation confirmed:
     a. Block source IP at HAProxy level (immediate)
     b. Block at nftables level on all Edge DMZ hosts
     c. If payload reached database: REVOKE application DB user privileges
     d. Engage on-call DBA to assess DB integrity
   - If WAF bypass: enable ModSecurity paranoia level 3 temporarily

3. INVESTIGATE (concurrent with containment)
   - Trace request via X-Request-ID across all log sources:
     HAProxy -> ModSecurity -> APISIX -> App -> pgAudit
   - Determine: what data was queried/modified/exfiltrated
   - Check pgAudit logs for any queries not matching application patterns
   - Review pg_stat_activity snapshots (if captured)

4. ERADICATE
   - Patch vulnerable application code (parameterized query fix)
   - Add specific ModSecurity rule for the attack vector
   - Run semgrep scan on entire codebase for similar patterns
   - Rotate all database credentials

5. RECOVER
   - Restore DB from verified clean backup if data was modified
   - Re-enable application DB user with least-privilege review
   - Verify application functionality

6. POST-INCIDENT (HIPAA SPECIFIC)
   - Determine if ePHI was accessed or exfiltrated
   - If yes: initiate HIPAA Breach Notification process
     - Document for covered entity within 24 hours
     - 60-day notification window to HHS begins
   - Conduct root cause analysis
   - Update WAF rules, add regression test to CI/CD pipeline
```

### 9.4 Automated Response Actions

Pre-approved automated responses that do not require human approval:

| Trigger | Automated Action | Tool |
|---------|-----------------|------|
| IP exceeds 500 req/min after 429s | Block IP for 1 hour at nftables | fail2ban |
| ModSecurity blocks 5+ SQLi from same IP | Block IP for 24 hours | fail2ban + custom script |
| Falco detects shell exec in production pod | Kill pod, alert P1 | Falco + falcosidekick + K8s |
| Suricata detects ePHI pattern in outbound traffic | Log + alert P1 (do not auto-block to avoid availability impact) | Suricata + Wazuh |

---

## 10. HIPAA-Specific Controls Summary

| HIPAA Requirement | Implementation |
|-------------------|---------------|
| Access Control (164.312(a)) | JWT auth at APISIX, Linkerd service-to-service authz, PostgreSQL role-based access |
| Audit Controls (164.312(b)) | pgAudit, ModSecurity audit log, APISIX access logs, Wazuh, 6-year Loki retention |
| Integrity Controls (164.312(c)) | TLS 1.3 everywhere, Linkerd mTLS, parameterized queries, WAF SQLi protection |
| Transmission Security (164.312(e)) | TLS 1.3 at every hop, no plaintext ePHI in transit |
| ePHI at Rest | PostgreSQL TDE or LUKS volume encryption, encrypted backups |
| Breach Notification (164.408) | Incident response runbooks with HIPAA breach assessment steps |
| Business Associate Agreements | Required for any third-party component with ePHI access (none in this FLOSS stack) |

---

## 11. Traffic Flow Summary

A complete request from an external client to a database query traverses these layers:

```
Client (TLS 1.3)
  -> HAProxy [Edge DMZ] (DDoS filtering, rate limiting, TLS termination/re-encryption)
    -> ModSecurity+Nginx [Edge DMZ] (WAF inspection, SQLi/XSS/OWASP Top 10 blocking)
      -> APISIX [App DMZ, K8s] (JWT auth, API rate limiting, routing)
        -> Patient Portal / REST API pod [App DMZ, K8s] (application logic)
          -> [Linkerd mTLS] Backend Service [Internal, K8s] (business logic)
            -> [Linkerd mTLS] PgBouncer [Internal, K8s] (connection pooling)
              -> PostgreSQL [Data zone, OpenStack VM] (TLS 1.3 + client cert)
```

**Seven security inspection/enforcement points** between the internet and the database. No single point of failure in the security chain.

---

## 12. FLOSS Component Summary

| Component | Version | License | Purpose |
|-----------|---------|---------|---------|
| HAProxy | 2.9+ | GPLv2 | Edge reverse proxy, DDoS mitigation |
| ModSecurity | 3.x | Apache 2.0 | Web Application Firewall |
| OWASP CRS | 4.x | Apache 2.0 | WAF rule set |
| Apache APISIX | 3.9+ | Apache 2.0 | API gateway |
| Calico | 3.27+ | Apache 2.0 | Kubernetes CNI + network policy |
| Linkerd | 2.15+ | Apache 2.0 | Service mesh, mTLS |
| cert-manager | 1.14+ | Apache 2.0 | Certificate lifecycle |
| Prometheus | 2.51+ | Apache 2.0 | Metrics |
| Grafana | 10.x | AGPL 3.0 | Dashboards |
| Loki | 3.x | AGPL 3.0 | Log aggregation |
| Falco | 0.37+ | Apache 2.0 | Runtime security |
| Suricata | 7.x | GPL 2.0 | Network IDS |
| Wazuh | 4.7+ | GPL 2.0 | Host IDS, compliance |
| PgBouncer | 1.22+ | ISC | Connection pooling |
| pgAudit | 16+ | PostgreSQL | Database audit logging |
| pgBackRest | 2.50+ | MIT | Database backups |
| fail2ban | 1.0+ | GPL 2.0 | Automated IP blocking |
| nftables | 1.0+ | GPL 2.0 | Host firewall |
| OpenBao | 2.x | MPL 2.0 | Secrets management (FLOSS fork of Vault) |
| keepalived | 2.2+ | GPL 2.0 | VRRP for HAProxy HA |
