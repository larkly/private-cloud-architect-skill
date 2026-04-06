# Architecture: Private Cloud Platform for HEMMELIG Classified Data

## Norwegian Armed Forces (Forsvaret) -- Classified Processing Environment

---

## 1. Executive Summary

This document defines the architecture for a private cloud platform capable of processing HEMMELIG (Secret) classified data for Forsvaret. The platform is designed to comply with Sikkerhetsloven (the Norwegian Security Act), satisfy accreditation requirements set by Nasjonal sikkerhetsmyndighet (NSM), and enable NATO interoperability for allied data sharing. The architecture is fully air-gapped, physically located in Norway, and operated exclusively by personnel holding HEMMELIG klarering.

**Recommended approach:** A Kubernetes-based platform running on hardened bare-metal infrastructure, with OpenStack considered only if VM-centric legacy workloads dominate. The rationale is detailed in Section 4.

---

## 2. Regulatory and Compliance Framework

### 2.1 Sikkerhetsloven (Norwegian Security Act)

- **Informasjonssikkerhet (Chapter 5):** All information systems processing HEMMELIG must be accredited. Data must be protected against espionage, sabotage, and terrorism.
- **Personellsikkerhet (Chapter 6):** Only personnel with valid sikkerhetsklarering for HEMMELIG may access the system or the physical facilities housing it.
- **Fysisk sikring (Chapter 7):** The facility must be a sperret/kontrollert omrade (restricted/controlled area) with layered physical security.
- **Anskaffelsessikkerhet (Chapter 9):** All procurement must follow security-cleared supply chain processes. Hardware and software must be assessed for supply chain risk.

### 2.2 NSM Accreditation Requirements

- The platform must undergo a formal accreditation process (godkjenning) by NSM before operational use.
- A Sikkerhetsmessig risikovurdering (security risk assessment) must be produced per NSM's Rammeverk for sikkerhetsarkitektur.
- Continuous monitoring and re-accreditation cycles must be planned.
- NSM's guidance on cryptographic solutions (NSM Kryptosikkerhet) must be followed -- only NSM-approved crypto devices and algorithms are permitted for protecting HEMMELIG data.

### 2.3 NATO Interoperability

- Data shared with NATO partners must conform to NATO security classifications (NATO SECRET maps to Norwegian HEMMELIG).
- Cross-domain solutions (CDS) or NATO-approved guard systems are required for any data exchange.
- Compliance with NATO STANAG 4774 (Confidentiality Metadata Label Syntax) and STANAG 4778 (Metadata Binding Mechanism) for data labeling.
- Interoperability with NATO Federated Mission Networking (FMN) standards where applicable.

---

## 3. Personnel and Access Model

### 3.1 Klarering Constraints

| Role | Required Klarering | Pool |
|---|---|---|
| Platform Engineers / SREs | HEMMELIG | 8 cleared personnel |
| Security Officers (Sikkerhetsleder) | HEMMELIG | From the 8 |
| Application Developers (cleared) | HEMMELIG | From the 8 |
| Application Developers (uncleared) | None | 12 remaining personnel |
| Facility Guards / Physical Security | KONFIDENSIELT minimum | External or internal |

### 3.2 Operational Separation

- **Cleared zone (HEMMELIG):** Only the 8 cleared personnel may enter the server rooms, access production systems, deploy to production, or view classified data. All administrative access (SSH, Kubernetes API, console) is restricted to this group.
- **Uncleared zone (development):** The 12 uncleared developers work in a completely separate, unclassified development environment. They write and test code against synthetic/unclassified data. Code is reviewed and promoted to the classified environment exclusively by cleared personnel.
- **Two-person integrity (TPI):** Critical operations (crypto key loading, firmware updates, system accreditation artifacts) require two cleared individuals acting together, per NSM guidance.

### 3.3 Code Promotion Pipeline

```
[UNCLASSIFIED Dev Environment] --> Code Review (cleared) --> Security Scan -->
  Manual Transfer (air-gap) --> [HEMMELIG Build Environment] --> Deployment (cleared only)
```

All code crossing the air-gap must be transferred via approved media (e.g., write-once optical media or NSM-approved data diodes) and scanned for malicious content.

---

## 4. Platform Decision: Kubernetes vs. OpenStack

### 4.1 Evaluation

| Criterion | Kubernetes | OpenStack |
|---|---|---|
| Operational complexity | Lower (fewer components) | Higher (many services: Nova, Neutron, Cinder, Keystone, etc.) |
| Personnel required for ops | 3-4 FTE | 5-7 FTE |
| Workload model | Containers, modern microservices | VMs, legacy workloads |
| Security hardening surface | Smaller | Larger |
| NATO/allied tooling trend | Kubernetes-native (e.g., NATO DIANA, US DoD Platform One) | Less common in new deployments |
| Air-gap registry support | Native (container registries) | Possible but more complex (Glance images) |
| Auditability | Strong (declarative, GitOps) | Possible but heavier tooling |

### 4.2 Recommendation

**Primary platform: Kubernetes on bare-metal** (using a hardened distribution such as RKE2/k3s from Rancher Federal, or Charmed Kubernetes with security hardening).

Rationale:
- With only 8 cleared operators, operational simplicity is critical. Kubernetes requires fewer people to run.
- NATO allies (especially US DoD via Platform One / Iron Bank) are standardizing on Kubernetes. This eases interoperability.
- Container-based workloads are easier to scan, sign, and verify in an air-gapped pipeline.
- If legacy VM workloads are required, KubeVirt can run VMs inside Kubernetes, avoiding the need for a full OpenStack deployment.

**If VM-centric workloads dominate**, consider OpenStack deployed via a minimal, hardened profile (e.g., Canonical's Charmed OpenStack with MicroStack, or StarlingX for edge/defense). However, this increases operational burden significantly for a team of 8.

---

## 5. Architecture Overview

### 5.1 Physical Architecture

```
+================================================================+
|  SPERRET OMRADE (Restricted Area) -- Norwegian Sovereign Soil  |
|  Physical Security: NSM-approved, 24/7 guarded, CCTV, alarms  |
|                                                                |
|  +----------------------------------------------------------+  |
|  |  RACK ZONE A (Compute)         RACK ZONE B (Compute)     |  |
|  |  - 6x bare-metal nodes         - 6x bare-metal nodes     |  |
|  |  - Kubernetes workers           - Kubernetes workers      |  |
|  |  - Encrypted local storage      - Encrypted local storage |  |
|  +----------------------------------------------------------+  |
|                                                                |
|  +----------------------------------------------------------+  |
|  |  RACK ZONE C (Control Plane + Storage)                    |  |
|  |  - 3x control plane nodes (HA etcd)                       |  |
|  |  - Ceph storage cluster (3+ OSD nodes, replicated)        |  |
|  |  - Container registry (Harbor, air-gapped)                |  |
|  +----------------------------------------------------------+  |
|                                                                |
|  +----------------------------------------------------------+  |
|  |  NETWORK ZONE                                             |  |
|  |  - Air-gapped: NO connection to internet or uncleared nets|  |
|  |  - Internal network: 10/25 GbE spine-leaf                 |  |
|  |  - Management network: separate VLAN, cleared access only |  |
|  |  - NATO Gateway: data diode + CDS (see Section 6)        |  |
|  +----------------------------------------------------------+  |
|                                                                |
|  +----------------------------------------------------------+  |
|  |  CRYPTO ZONE                                              |  |
|  |  - NSM-approved COMSEC devices (bulk encryptors)          |  |
|  |  - Key management system (offline root CA)                |  |
|  +----------------------------------------------------------+  |
+================================================================+
```

### 5.2 Logical Architecture

```
+---------------------------------------------------------------+
|  GitOps Layer (Flux CD / ArgoCD -- air-gapped)                |
|  - Declarative cluster state                                   |
|  - Policy-as-code (OPA/Gatekeeper, Kyverno)                  |
+---------------------------------------------------------------+
|  Application Layer                                             |
|  - Mission workloads (containers)                              |
|  - NATO interop services (NFFI, MTF, OTH-Gold adapters)      |
|  - Data processing pipelines                                   |
+---------------------------------------------------------------+
|  Platform Services                                             |
|  - Service mesh (Istio/Linkerd -- mTLS everywhere)            |
|  - Monitoring (Prometheus, Grafana, Loki -- all internal)     |
|  - Secrets management (HashiCorp Vault, air-gapped)           |
|  - Image signing & verification (Cosign/Notary)               |
|  - Admission control (images must be signed, scanned)         |
+---------------------------------------------------------------+
|  Kubernetes Cluster (RKE2 or equivalent hardened distro)       |
|  - RBAC with OIDC (Keycloak, internal)                        |
|  - Pod Security Standards: Restricted                          |
|  - Network Policies: default-deny, explicit allow             |
|  - Audit logging: all API calls logged to immutable store     |
+---------------------------------------------------------------+
|  Infrastructure                                                |
|  - Bare-metal (no hypervisor unless KubeVirt needed)          |
|  - Ceph (encrypted at rest, replicated)                        |
|  - Spine-leaf network (25 GbE, hardware-segmented)            |
|  - TPM 2.0 on all nodes (measured boot, attestation)          |
+---------------------------------------------------------------+
```

---

## 6. Air-Gap and NATO Interoperability

### 6.1 Air-Gap Design

The platform has **zero connectivity** to any unclassified network or the public internet. All data ingress/egress occurs through:

1. **Physical media transfer:** Write-once optical media (BD-R) or NSM-approved USB devices, used for code/image promotion from the unclassified development environment. Each transfer is logged, scanned, and approved by a cleared operator.
2. **Data diode (one-way):** For specific sensor feeds or intelligence inputs that flow into the classified environment. Hardware-enforced unidirectionality (e.g., Advenica SecuriCDS, Waterfall Security). No data can traverse the diode in the reverse direction.

### 6.2 NATO Data Sharing Gateway

For sharing data with NATO partners at NATO SECRET level:

```
[HEMMELIG Platform] --> NSM-approved CDS/Guard --> [NATO WAN / BICES / NSWAN]
```

- A **cross-domain solution (CDS)** or security guard is required. This must be an NSM-accredited product (e.g., solutions from Thales, Advenica, or Everfox/Forcepoint).
- The CDS enforces:
  - Content inspection and sanitization
  - Classification label validation (STANAG 4774/4778)
  - Protocol break (no direct TCP sessions cross the boundary)
  - Full audit trail of all transferred data
- The NATO-side connection uses NSM/NATO-approved bulk encryptors (e.g., Thales Datacryptor or equivalent TEMPEST-rated devices).
- Norwegian HEMMELIG data must be explicitly releaseled (REL TO NATO or specific nations) before transfer. The CDS enforces release markings.

### 6.3 Classification Labeling

All data objects within the platform carry machine-readable classification labels:

- Implemented via Kubernetes labels/annotations and enforced by admission webhooks.
- Labels follow STANAG 4774 syntax: `classification: NATO SECRET`, `releasability: REL TO NATO`, `national caveats: NO EYES ONLY`, etc.
- OPA/Gatekeeper policies prevent workloads from processing data above their authorized level.

---

## 7. Security Architecture

### 7.1 Defense in Depth

| Layer | Control |
|---|---|
| Physical | NSM-approved facility, sperret omrade, 24/7 guards, CCTV, intrusion detection |
| Network | Air-gapped, spine-leaf with microsegmentation, default-deny network policies, no east-west traffic without explicit policy |
| Host | Hardened OS (e.g., RHEL STIG or Ubuntu Pro FIPS), SELinux enforcing, CIS benchmarks, TPM 2.0 measured boot, UEFI Secure Boot |
| Container | Signed images only (Cosign), vulnerability-scanned (Trivy/Grype), read-only root filesystems, non-root execution, seccomp/AppArmor profiles |
| Application | mTLS via service mesh, OIDC authentication (Keycloak), RBAC per workload, no shared service accounts |
| Data | Encryption at rest (LUKS + Ceph encryption, NSM-approved algorithms), encryption in transit (TLS 1.3 / mTLS), data classification labels |
| Audit | Immutable audit logs (append-only, shipped to separate log aggregation nodes), Kubernetes audit policy set to RequestResponse for all resources |

### 7.2 Cryptography

- **At rest:** AES-256 (via LUKS for node disks, Ceph encryption for cluster storage). Keys managed by HashiCorp Vault with auto-unseal via TPM.
- **In transit:** TLS 1.3 with NSM-approved cipher suites. mTLS between all services via service mesh.
- **For NATO links:** NSM-approved bulk encryptors. Key material managed per COMSEC procedures with physical key loading ceremonies (TPI required).
- **Certificate Authority:** Offline root CA, intermediate CA online within the cluster. Short-lived certificates (24h) for workloads via cert-manager.

### 7.3 Identity and Access Management

- **Keycloak** (internal, air-gapped) as OIDC provider.
- All personnel authenticate with smart card (PKI) + PIN (two-factor).
- Kubernetes RBAC roles mapped to Keycloak groups.
- No shared accounts. All actions attributable to individual cleared personnel.
- Session recording for all administrative access (e.g., Teleport or similar).

### 7.4 Supply Chain Security

- All hardware procured through security-cleared supply chains per Sikkerhetsloven Chapter 9.
- Hardware inspection upon delivery (tamper evidence, firmware verification).
- Software Bill of Materials (SBOM) maintained for all deployed components.
- Container images built internally from source where possible; third-party images rebuilt from trusted base images and scanned.
- All images stored in air-gapped Harbor registry, signed with Cosign, verified at admission.

---

## 8. Storage Architecture

### 8.1 Ceph Cluster

- Minimum 3 OSD nodes for replication (replication factor 3).
- All OSDs on encrypted drives (LUKS + dm-crypt).
- Provides block storage (RBD for Kubernetes PVs), object storage (RGW for S3-compatible APIs), and file storage (CephFS) as needed.
- Separate CRUSH rules to ensure data replicas span physical racks/zones.

### 8.2 Data Retention and Destruction

- Data retention policies enforced per Forsvaret and NSM requirements.
- Data destruction follows NSM's guidelines for sanitization of storage media at HEMMELIG level (degaussing + physical destruction for decommissioned drives).
- Kubernetes PersistentVolume reclaim policy set to `Delete` with crypto-erase verification.

---

## 9. Monitoring, Logging, and Incident Response

### 9.1 Monitoring Stack

All monitoring is internal -- no telemetry leaves the air-gapped environment.

- **Metrics:** Prometheus + Thanos (long-term storage on Ceph).
- **Dashboards:** Grafana (internal).
- **Logs:** Fluentd/Fluent Bit --> Loki or Elasticsearch (immutable, append-only indices).
- **Alerts:** Alertmanager with on-call routing to cleared personnel only.

### 9.2 Security Monitoring

- **Falco** for runtime container behavior monitoring (syscall anomaly detection).
- **Kubernetes audit logs** shipped to immutable storage and analyzed for anomalies.
- **Network flow logs** captured and analyzed for lateral movement detection.
- **SIEM integration:** Logs aggregated into a SIEM (e.g., Elastic Security or Wazuh) for correlation and alerting.

### 9.3 Incident Response

- Security incidents reported to NSM per Sikkerhetsloven requirements.
- Incident response plan developed and exercised per NSM's guidance.
- Forensic imaging capability maintained within the air-gapped environment.
- Cleared incident response team drawn from the 8 HEMMELIG-cleared personnel.

---

## 10. Deployment and Operations

### 10.1 GitOps Workflow (Air-Gapped)

```
[Unclassified Dev] --code--> [Optical Media] --scan--> [Classified Git (Gitea/GitLab)]
                                                              |
                                                              v
                                                      [Flux CD / ArgoCD]
                                                              |
                                                              v
                                                      [Kubernetes Cluster]
```

- Internal Git server (Gitea or GitLab, air-gapped) holds all cluster configuration and application manifests.
- Flux CD or ArgoCD reconciles desired state from Git to the cluster.
- All changes go through merge requests reviewed by cleared personnel.

### 10.2 Day-2 Operations

- **Patching:** OS and Kubernetes patches are downloaded in the unclassified environment, reviewed, scanned, transferred via air-gap, and applied in maintenance windows.
- **Capacity planning:** Prometheus metrics used for trend analysis. New hardware procurement follows security-cleared supply chain process (lead time: plan for 3-6 months).
- **Backup:** Velero for Kubernetes resource backup, Ceph snapshots for persistent data. Backups encrypted and stored on separate media. Tested quarterly.
- **Disaster recovery:** Cold standby site or documented rebuild procedures. RTO/RPO defined per Forsvaret mission requirements.

### 10.3 Staffing Model (8 Cleared Personnel)

| Function | FTE Allocation | Notes |
|---|---|---|
| Platform Engineering / SRE | 3 | Kubernetes, Ceph, networking |
| Security Operations | 2 | SIEM, incident response, accreditation |
| Release Engineering | 1 | Air-gap transfers, image builds, GitOps |
| Sikkerhetsleder (Security Officer) | 1 | Compliance, NSM liaison, audits |
| Technical Lead / Architect | 1 | Architecture decisions, NATO interop |

This is tight. Cross-training is essential. Consider requesting additional klarering for 2-4 more team members to reduce single-person dependencies.

---

## 11. Accreditation Roadmap

| Phase | Activities | Duration |
|---|---|---|
| 1. Pre-study | Threat assessment, risk analysis (sikkerhetsmessig risikovurdering), architecture documentation | 2-3 months |
| 2. Design | Detailed security architecture, select and procure hardware/software, NSM consultation | 3-4 months |
| 3. Build | Deploy infrastructure, harden, configure, build air-gapped CI/CD | 3-4 months |
| 4. Test | Penetration testing, compliance verification, documentation review | 2-3 months |
| 5. Accreditation | Submit to NSM, address findings, obtain godkjenning | 2-4 months |
| 6. Operate | Go-live, continuous monitoring, periodic re-accreditation | Ongoing |

**Estimated timeline to initial operating capability: 12-18 months.**

---

## 12. Key Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Only 8 cleared personnel | Bus factor, burnout, insufficient coverage for 24/7 ops | Cross-train aggressively, request additional klarering, automate everything possible |
| Air-gap slows patching | Vulnerability exposure window | Prioritize critical patches, maintain a fast-track transfer process, defense-in-depth compensates |
| NSM accreditation delays | Project timeline slip | Engage NSM early and continuously, use pre-approved components where possible |
| Supply chain compromise | Hardware/software backdoors | Procure through cleared channels, inspect hardware, maintain SBOM, use reproducible builds |
| NATO interop complexity | Integration delays with allies | Adopt NATO FMN standards early, test with NATO exercises (e.g., Coalition Warrior Interoperability Exercise) |
| Ceph/K8s operational complexity | Outages in production | Invest in training, maintain runbooks, practice failure scenarios |

---

## 13. Hardware Bill of Materials (Indicative)

| Component | Quantity | Specification |
|---|---|---|
| Compute nodes (workers) | 12 | 2x AMD EPYC / Intel Xeon, 512GB RAM, 2x 1TB NVMe (OS), 25GbE NIC, TPM 2.0 |
| Control plane nodes | 3 | 2x CPU, 256GB RAM, 2x 1TB NVMe, 25GbE NIC, TPM 2.0 |
| Storage nodes (Ceph OSD) | 6 | 1x CPU, 128GB RAM, 8x 4TB NVMe (OSD), 2x 25GbE NIC, TPM 2.0 |
| Spine switches | 2 | 100GbE, managed, TEMPEST-rated if required |
| Leaf switches | 4 | 25GbE, managed |
| Management switch | 1 | 1GbE, out-of-band management |
| Data diode | 1-2 | NSM-approved (e.g., Advenica) |
| Cross-domain solution | 1 | NSM-accredited guard for NATO gateway |
| Bulk encryptors | 2 | NSM-approved, for NATO WAN link |
| UPS + PDU | As needed | Redundant power |

---

## 14. Summary of Key Architectural Decisions

1. **Kubernetes over OpenStack** -- fewer operators needed, aligns with NATO ally trends, simpler security surface.
2. **Bare-metal deployment** -- no hypervisor layer reduces attack surface; KubeVirt available if VMs are needed.
3. **RKE2 or equivalent hardened distribution** -- FIPS-capable, STIG-aligned, built for air-gapped defense deployments.
4. **Ceph for storage** -- unified storage (block/object/file), proven at scale, open source with no vendor lock-in.
5. **GitOps with air-gapped Git** -- declarative, auditable, reproducible deployments.
6. **NSM-approved crypto only** -- no compromises on encryption for HEMMELIG data.
7. **NATO interop via CDS and STANAG compliance** -- data sharing without compromising the classified environment.
8. **Strict personnel separation** -- uncleared developers never touch classified systems; cleared operators control all production access.
