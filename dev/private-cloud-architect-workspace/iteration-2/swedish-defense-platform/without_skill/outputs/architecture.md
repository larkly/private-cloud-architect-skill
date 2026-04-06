# Architecture Document: Private Cloud Platform for HEMLIG Classified Data Processing

**Classification:** HEMLIG/Restricted (This document describes the architecture; actual implementation details at HEMLIG level require separate handling)

**Version:** 1.0
**Date:** 2026-03-20
**Status:** Draft for Review
**Intended Audience:** System Architects, CISO, Security Officer (Säkerhetsskyddschef), FMV/MUST Accreditation Board

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Regulatory and Legal Framework](#2-regulatory-and-legal-framework)
3. [Threat Model and Security Objectives](#3-threat-model-and-security-objectives)
4. [Architecture Overview](#4-architecture-overview)
5. [Physical Infrastructure](#5-physical-infrastructure)
6. [Network Architecture](#6-network-architecture)
7. [Compute Platform: Kubernetes](#7-compute-platform-kubernetes)
8. [Storage Architecture](#8-storage-architecture)
9. [Identity, Access, and Clearance Management](#9-identity-access-and-clearance-management)
10. [Cryptographic Services](#10-cryptographic-services)
11. [Air-Gap Strategy and Supply Chain](#11-air-gap-strategy-and-supply-chain)
12. [Logging, Auditing, and SIEM](#12-logging-auditing-and-siem)
13. [Backup, Disaster Recovery, and Continuity](#13-backup-disaster-recovery-and-continuity)
14. [Operational Model and Staffing](#14-operational-model-and-staffing)
15. [Accreditation Roadmap](#15-accreditation-roadmap)
16. [Risk Register](#16-risk-register)
17. [Appendices](#17-appendices)

---

## 1. Executive Summary

This document defines the target architecture for a private, air-gapped cloud platform designed to process data classified at the Swedish HEMLIG (Secret) level. The platform is built for a Swedish defense contractor (försvarsindustri) and must comply with Säkerhetsskyddslagen (2018:585), its associated regulations (Säkerhetsskyddsförordningen 2021:955), and the prescriptive guidance issued by Försvarets materielverk (FMV) and Militära underrättelse- och säkerhetstjänsten (MUST).

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Kubernetes-based orchestration** | Mature ecosystem, declarative security policies, Swedish defense community familiarity |
| **Fully air-gapped** | Mandatory for HEMLIG; no bidirectional connectivity to any unclassified network |
| **All infrastructure in Sweden** | Data sovereignty requirement; all hardware in Swedish-territory facilities |
| **Hardware-rooted trust** | TPM 2.0, measured boot, HSM-backed certificate infrastructure |
| **12-person cleared operations team** | Only personnel with Säpo-vetted HEMLIG clearance touch the classified environment |

### Platform Capabilities

- Container orchestration for classified workloads
- Bare-metal and virtualized compute
- Encrypted-at-rest and in-transit storage
- CI/CD pipeline (air-gapped)
- Centralized logging and audit trail for accreditation compliance
- Capacity for ~200 classified workloads concurrently

---

## 2. Regulatory and Legal Framework

### 2.1 Primary Legislation

| Law / Regulation | Relevance |
|------------------|-----------|
| **Säkerhetsskyddslagen (2018:585)** | Overarching security protection act; defines obligations for entities handling classified information |
| **Säkerhetsskyddsförordningen (2021:955)** | Implementing regulation; details on information security, physical security, personnel security |
| **Försvarsmaktens föreskrifter (FFS)** | Military-specific security requirements |
| **MUST IT-säkerhetsinstruktion** | Technical IT security requirements for systems processing classified information |
| **Försvarets materielverks föreskrifter** | FMV-specific accreditation and procurement guidance |
| **Offentlighets- och sekretesslagen (2009:400)** | Governs secrecy of classified information |

### 2.2 Classification Level: HEMLIG

Swedish classification levels are:

1. **HEMLIG/RESTRICTED (H/R)** -- Limited damage to Sweden's security
2. **HEMLIG/CONFIDENTIAL (H/C)** -- Damage to Sweden's security
3. **HEMLIG/SECRET (H/S)** -- Serious damage to Sweden's security
4. **HEMLIG/TOP SECRET (H/TS)** -- Exceptionally serious damage

This platform targets **HEMLIG/SECRET (H/S)**. This is the second-highest classification level and imposes stringent requirements on physical security, personnel vetting, technical countermeasures, and auditing.

### 2.3 Accreditation Authority

- **FMV** (Försvarets materielverk) is responsible for system accreditation of defense materiel IT systems.
- **MUST** (Militära underrättelse- och säkerhetstjänsten) provides signals intelligence protection guidance and approves cryptographic solutions.
- The organization's **Säkerhetsskyddschef** (Security Protection Officer) is legally responsible for the security protection within the organization and acts as the liaison to accreditation authorities.

### 2.4 International Frameworks (Informative)

While Swedish law is primary, interoperability with NATO standards is relevant for a defense contractor:

- **NATO AC/322-D(2004)0024-REV4** -- NATO Information Assurance
- **CC (Common Criteria)** -- For evaluated products
- **ISO 27001/27002** -- As a baseline (not sufficient alone for HEMLIG)

---

## 3. Threat Model and Security Objectives

### 3.1 Threat Actors

| Actor | Capability | Relevance |
|-------|-----------|-----------|
| Nation-state intelligence services | Very High | Primary threat; HEMLIG data is a strategic target |
| Advanced Persistent Threats (APTs) | High | Supply chain, insider, zero-day exploitation |
| Insider threat (cleared personnel) | Medium-High | 12 cleared staff with deep access |
| Insider threat (non-cleared personnel) | Medium | 28 engineers who must be kept away from classified data |
| Organized crime | Medium | Ransomware, extortion |
| Physical intrusion | Medium | Facility break-in |

### 3.2 Security Objectives (CIA + Accountability)

1. **Confidentiality**: No HEMLIG data may leave the accredited boundary. No uncleared person may access HEMLIG data. No foreign intelligence service may gain access.
2. **Integrity**: All data and system configurations must be tamper-evident. Unauthorized modification must be detected and attributed.
3. **Availability**: The platform must support defense program timelines. RTO < 4 hours, RPO < 1 hour for critical workloads.
4. **Accountability**: Every action by every user and system component must be logged, attributed to a cleared individual, and retained for the period mandated by FMV.

### 3.3 TEMPEST / Emanation Security

HEMLIG/SECRET processing requires TEMPEST countermeasures per MUST guidelines:

- Facility must meet SDIP-27 Level B (NATO AMSG-720B) or as specified by MUST
- Shielded rooms (skärmade rum) or zones with verified attenuation
- All cabling must be fiber optic within the classified zone where possible; copper must be filtered
- Equipment must be TEMPEST-approved or used within an approved installation

---

## 4. Architecture Overview

### 4.1 High-Level Architecture Diagram (Textual)

```
                    +-----------------------------------------+
                    |        UNCLASSIFIED ZONE                |
                    |  (Developer workstations, internet,     |
                    |   unclassified CI/CD, artifact staging) |
                    +-----------------+-----------------------+
                                      |
                              [DATA DIODE / MANUAL MEDIA]
                              (One-way transfer only, into
                               classified zone)
                                      |
                                      v
+=====================================================================+
|                     CLASSIFIED ZONE (HEMLIG/SECRET)                  |
|                     Physical Security Boundary                       |
|  +---------------------------------------------------------------+  |
|  |                    MANAGEMENT PLANE                            |  |
|  |  +------------+  +------------+  +-----------+  +-----------+ |  |
|  |  | Kubernetes |  |   Vault    |  |  GitLab   |  |   SIEM    | |  |
|  |  | Control    |  |   (HSM-    |  |  (Air-gap |  | (Elastic/ | |  |
|  |  | Plane (HA) |  |   backed)  |  |   clone)  |  |  Splunk)  | |  |
|  |  +------------+  +------------+  +-----------+  +-----------+ |  |
|  |  +------------+  +------------+  +-----------+                |  |
|  |  | Container  |  |   PKI /    |  | Artifact  |               |  |
|  |  | Registry   |  |   CA       |  | Registry  |               |  |
|  |  | (Harbor)   |  |   (EJBCA)  |  | (Nexus)   |               |  |
|  |  +------------+  +------------+  +-----------+                |  |
|  +---------------------------------------------------------------+  |
|                                                                      |
|  +---------------------------------------------------------------+  |
|  |                      DATA PLANE                               |  |
|  |  +-------------------+  +-------------------+                 |  |
|  |  | Worker Node Pool  |  | Worker Node Pool  |                |  |
|  |  | (General Purpose) |  | (GPU / HPC)       |                |  |
|  |  | 10-20 nodes       |  | 4-8 nodes         |                |  |
|  |  +-------------------+  +-------------------+                |  |
|  |                                                               |  |
|  |  +-------------------+  +-------------------+                 |  |
|  |  | Ceph Storage      |  | Backup Storage    |                |  |
|  |  | Cluster (RBD,     |  | (Encrypted, off-  |                |  |
|  |  |  CephFS, RGW)     |  |  site capable)    |                |  |
|  |  +-------------------+  +-------------------+                |  |
|  +---------------------------------------------------------------+  |
|                                                                      |
|  +---------------------------------------------------------------+  |
|  |                    INFRASTRUCTURE SERVICES                    |  |
|  |  DNS (CoreDNS) | NTP (Chrony, GPS) | LDAP (FreeIPA)         |  |
|  |  Monitoring (Prometheus/Grafana) | Log Shipping (Fluentd)    |  |
|  +---------------------------------------------------------------+  |
+=====================================================================+
```

### 4.2 Design Principles

1. **Defense in depth**: Multiple independent security layers; no single point of compromise leads to data exfiltration.
2. **Zero trust within the classified zone**: Even inside the air-gap, all service-to-service communication is mutually authenticated and encrypted (mTLS).
3. **Immutable infrastructure**: Nodes are provisioned from signed, measured images. No SSH in production; changes only via GitOps.
4. **Least privilege**: RBAC at every layer -- Kubernetes, OS, storage, network.
5. **Auditability**: Every API call, login, file access, and network connection is logged to a tamper-evident audit log.
6. **Separation of duties**: No single person can deploy code, approve it, and access production data.
7. **Swedish sovereignty**: All hardware, software supply chain verification, and operational control remain within Sweden.

---

## 5. Physical Infrastructure

### 5.1 Facility Requirements

The datacenter hosting the classified zone must meet the requirements of Säkerhetsskyddsförordningen for HEMLIG/SECRET:

| Requirement | Implementation |
|-------------|---------------|
| **Location** | Sweden, preferably underground or hardened facility |
| **Physical perimeter** | Skalskydd with multiple layers: outer perimeter, building, server room, rack-level |
| **Access control** | Multi-factor: badge + biometric + PIN at classified zone entry |
| **Intrusion detection** | 24/7 monitored alarm system with tamper detection on racks |
| **TEMPEST shielding** | SDIP-27 Level B compliant room or enclosure |
| **Power** | Dual redundant feeds, UPS, diesel generator (N+1) |
| **Cooling** | Redundant cooling; liquid cooling for GPU nodes |
| **Fire suppression** | Gas-based (Novec/FM-200), not water |
| **Visitor control** | Escorted access only; visitor log; no electronics permitted |
| **CCTV** | Recorded surveillance of all entry points and server room; 90-day retention |

### 5.2 Hardware Selection

All hardware must be procured through a vetted supply chain. FMV may require specific vendors or supply chain attestations.

#### Compute Nodes

| Component | Specification | Notes |
|-----------|--------------|-------|
| **Server platform** | HPE ProLiant DL380 Gen11 or Dell PowerEdge R760 | Widely used in European defense; supply chain auditable |
| **CPU** | Intel Xeon Sapphire Rapids or later | Must support Intel TXT / TME for measured boot and memory encryption |
| **RAM** | 512 GB DDR5 ECC per node minimum | Memory encryption (TME) enabled |
| **Boot** | 2x NVMe M.2 in RAID-1, LUKS2 encrypted | Measured boot chain via TPM 2.0 |
| **TPM** | TPM 2.0 (discrete, not firmware) | Hardware root of trust for measured boot |
| **NIC** | 2x 25 GbE (management) + 2x 100 GbE (data) | SR-IOV capable for network performance |
| **GPU nodes** | NVIDIA A100/H100 (if ML/AI workloads) | NVIDIA GPU attestation for confidential computing |
| **BMC/IPMI** | Dedicated management network, firmware pinned | iLO/iDRAC on isolated VLAN; credentials in Vault |

#### Storage Nodes

| Component | Specification |
|-----------|--------------|
| **Platform** | Dedicated Ceph OSD nodes, 4-8 nodes |
| **Drives** | 12x NVMe SSD per node (high IOPS) or mixed NVMe/HDD for tiering |
| **Network** | 2x 100 GbE for Ceph cluster network |
| **Encryption** | dm-crypt/LUKS2 on all OSDs; keys from Vault/HSM |

#### HSM (Hardware Security Module)

| Component | Specification |
|-----------|--------------|
| **Model** | Thales Luna Network HSM 7 or Utimaco SecurityServer |
| **Deployment** | HA pair, physically installed in the classified zone |
| **Certification** | FIPS 140-2 Level 3 minimum; CC EAL4+ preferred |
| **Use cases** | Root CA key, Vault unseal keys, Kubernetes signing keys, disk encryption key wrapping |

### 5.3 Rack Layout (Conceptual)

```
Rack 1-2:   Kubernetes Control Plane (3 masters, quorum)
Rack 3-6:   Kubernetes Worker Nodes (general purpose)
Rack 7-8:   GPU/HPC Worker Nodes
Rack 9-10:  Ceph Storage Nodes
Rack 11:    Management Services (Vault, GitLab, Harbor, SIEM)
Rack 12:    Network Equipment (switches, firewalls, data diode)
Rack 13:    HSM, Backup Controllers, KVM console
```

All racks must have:
- Physical locks (key + electronic)
- Tamper-evident seals on unused ports
- Cable management preventing unauthorized taps
- Asset tags and serial number registry

---

## 6. Network Architecture

### 6.1 Network Zones

The network is divided into strictly separated zones:

```
Zone 0: UNCLASSIFIED (outside the security boundary)
   |
   | [DATA DIODE] (hardware-enforced one-way)
   |
Zone 1: TRANSFER ZONE (classified, staging area for inbound content)
   |
   | [FIREWALL + IDS/IPS]
   |
Zone 2: MANAGEMENT ZONE (classified, control plane and ops tools)
   |
   | [FIREWALL + NETWORK POLICY]
   |
Zone 3: WORKLOAD ZONE (classified, application workloads)
   |
   | [FIREWALL + NETWORK POLICY]
   |
Zone 4: DATA ZONE (classified, storage and databases)
   |
   | [FIREWALL + NETWORK POLICY]
   |
Zone 5: BMC/IPMI ZONE (classified, out-of-band management, highly restricted)
```

### 6.2 Data Diode

A **hardware data diode** (e.g., Advenica SecuriCDS Data Diode -- a Swedish product specifically designed for Swedish defense use) provides the only connectivity between the unclassified and classified zones.

- **Direction**: Unclassified -> Classified (inbound only)
- **Purpose**: Software updates, container images (after scanning), configuration data
- **No return path**: The diode physically prevents any data from leaving the classified zone
- **Approval process**: All content transferred through the diode must be approved by the security officer and scanned by the transfer zone's malware analysis pipeline

Advenica is a Swedish company with products evaluated for Swedish defense use, making them a strong candidate for regulatory alignment.

### 6.3 Internal Network Design

| Parameter | Design |
|-----------|--------|
| **Fabric** | Spine-leaf (Clos) topology for east-west traffic |
| **Spine switches** | 2x 100 GbE spine switches (redundant) |
| **Leaf switches** | 1 per rack pair, 25/100 GbE uplinks |
| **Overlay** | VXLAN with BGP EVPN for Kubernetes networking |
| **CNI** | Cilium (eBPF-based) -- provides NetworkPolicy enforcement, encryption (WireGuard), and deep observability |
| **Service mesh** | Istio or Linkerd for mTLS between all services (evaluate based on MUST guidance) |
| **DNS** | CoreDNS (internal), no external DNS resolution |
| **NTP** | Chrony synced to GPS receiver (no internet NTP) |
| **Firewall** | Host-based (nftables) + Kubernetes NetworkPolicy + inter-zone hardware firewalls |

### 6.4 Network Policy Model

Kubernetes NetworkPolicy (enforced via Cilium) follows a **default-deny** model:

```yaml
# Default deny all ingress and egress
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

Every workload must explicitly declare its allowed network connections. Cilium's L7 policies provide application-layer visibility and enforcement (HTTP method/path, gRPC service, DNS name filtering).

### 6.5 No Egress from Classified Zone

There is **no network path** for data to leave the classified zone electronically. This is enforced by:

1. **Physical topology**: No cable or fiber connects classified zone to any external network in the outbound direction.
2. **Data diode**: Hardware-enforced unidirectional transfer (inbound only).
3. **Switch ACLs**: Even within the classified zone, no route exists to any external address space.
4. **Host firewalls**: All nodes have nftables rules dropping traffic to non-RFC1918 addresses (defense in depth).

---

## 7. Compute Platform: Kubernetes

### 7.1 Distribution Selection

**Recommendation: RKE2 (Rancher Kubernetes Engine 2)** also known as "RKE Government."

| Factor | RKE2 | OpenShift | Vanilla k8s (kubeadm) |
|--------|-------|-----------|----------------------|
| Air-gap support | Excellent, first-class | Good | Manual |
| CIS hardening | Built-in, default CIS 1.6+ profile | Available | Manual |
| FIPS 140-2 mode | Yes (FIPS-validated crypto) | Yes | Manual |
| SELinux integration | Yes | Yes | Manual |
| Lifecycle management | Good (Rancher optional) | Mature | Manual |
| Footprint / complexity | Medium | Heavy | Light |
| Swedish defense usage | Growing adoption | Some usage | Rare for production |

RKE2 is purpose-built for air-gapped, security-hardened environments and ships with CIS benchmark compliance out of the box. It uses containerd as the runtime (not Docker) and embeds etcd.

### 7.2 Cluster Topology

```
CONTROL PLANE (3 nodes, HA):
  rke2-cp-01  (Rack 1)  -- etcd + control plane
  rke2-cp-02  (Rack 1)  -- etcd + control plane
  rke2-cp-03  (Rack 2)  -- etcd + control plane

  Load Balancer: HAProxy/keepalived VIP for API server

WORKER POOL - GENERAL (10-20 nodes):
  rke2-worker-01 through rke2-worker-20  (Racks 3-6)
  Labels: node-role=general, security-zone=workload

WORKER POOL - GPU (4-8 nodes):
  rke2-gpu-01 through rke2-gpu-08  (Racks 7-8)
  Labels: node-role=gpu, nvidia.com/gpu=present
  Taints: gpu=true:NoSchedule (only GPU workloads scheduled here)

INFRASTRUCTURE POOL (3-5 nodes):
  rke2-infra-01 through rke2-infra-05  (Rack 11)
  Labels: node-role=infrastructure
  Runs: monitoring, logging, GitLab runners, Harbor, Vault
  Taints: infrastructure=true:NoSchedule
```

### 7.3 Hardening Measures

#### 7.3.1 CIS Kubernetes Benchmark

RKE2 ships with CIS 1.6+ hardening by default. Additional measures:

- **API server**:
  - `--anonymous-auth=false`
  - `--authorization-mode=RBAC,Node`
  - `--audit-log-path` and `--audit-log-maxage=365`
  - `--encryption-provider-config` (etcd encryption at rest using HSM-backed keys)
  - `--tls-min-version=VersionTLS13`
  - Admission controllers: `PodSecurity`, `NodeRestriction`, `AlwaysPullImages`

- **etcd**:
  - Encrypted at rest (AES-GCM-256, key from Vault/HSM)
  - Client certificate authentication only
  - Separate PKI from Kubernetes API PKI
  - Snapshot backups every hour to encrypted storage

- **Kubelet**:
  - `--anonymous-auth=false`
  - `--authorization-mode=Webhook`
  - `--protect-kernel-defaults=true`
  - `--read-only-port=0`
  - Certificate rotation enabled

#### 7.3.2 Pod Security Standards

Enforce the **Restricted** Pod Security Standard at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: classified-workload
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

This prevents:
- Privileged containers
- Host namespace sharing
- Host path mounts
- Running as root
- Privilege escalation
- Non-default capabilities

#### 7.3.3 Runtime Security

- **Seccomp profiles**: Custom seccomp profiles for all workloads (default: RuntimeDefault)
- **AppArmor/SELinux**: SELinux enforcing on all nodes; mandatory SELinux context for all pods
- **Read-only root filesystem**: Enforced for all application containers
- **No writable hostPath**: Blocked by admission policy
- **Image signature verification**: Cosign/Notation signatures verified by a policy engine (Kyverno or OPA Gatekeeper)
- **Runtime threat detection**: Falco deployed as DaemonSet for syscall monitoring

#### 7.3.4 Supply Chain Security for Container Images

All container images must:

1. Be built from source in the unclassified zone
2. Be scanned by Trivy/Grype for CVEs (critical/high must be remediated)
3. Be signed with Cosign using a key from the classified zone's PKI
4. Be transferred via the data diode to the classified zone's Harbor registry
5. Be re-scanned in the transfer zone before promotion to the production registry
6. Have their SBOM (Software Bill of Materials) stored and auditable
7. Be verified by admission controller before deployment

### 7.4 GitOps Deployment Model

**Tool: Flux CD** (or ArgoCD) running inside the classified zone.

```
Unclassified Zone:                Classified Zone:

Developer writes code  ------>    [Data Diode]
Code review (GitLab)              |
Build & scan pipeline             v
Sign artifacts                    Transfer Zone GitLab (mirror)
                                  |
                                  v
                                  Flux CD watches classified GitLab
                                  |
                                  v
                                  Deploys to Kubernetes cluster
                                  (after policy validation)
```

- All deployments are declarative (Helm charts or Kustomize manifests in Git)
- No `kubectl apply` or `kubectl exec` in production (enforced by RBAC and audit)
- Deployment requires two-person approval in GitLab merge request
- Flux reconciliation interval: 1 minute
- Drift detection: Flux alerts if cluster state diverges from Git

---

## 8. Storage Architecture

### 8.1 Ceph Cluster

Ceph provides unified storage (block, filesystem, object) for the classified zone.

| Parameter | Value |
|-----------|-------|
| **Deployment** | Rook-Ceph operator on Kubernetes |
| **OSD nodes** | 4-8 dedicated storage nodes |
| **Replication** | 3x replication (size=3, min_size=2) |
| **Encryption** | dm-crypt on all OSDs, keys from Vault (HSM-backed) |
| **Network** | Dedicated 100 GbE cluster network, separated from public |
| **Interfaces** | RBD (block) for PVs, CephFS (filesystem) for shared, RGW (S3) for object |

### 8.2 Storage Classes

```yaml
# Encrypted block storage (default)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-block-encrypted
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  encrypted: "true"
  encryptionKMSID: vault-kms
reclaimPolicy: Delete
allowVolumeExpansion: true

# Encrypted shared filesystem
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-filesystem-encrypted
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  encrypted: "true"
reclaimPolicy: Delete
```

### 8.3 Data Classification Labeling

All PersistentVolumeClaims must carry a classification label:

```yaml
metadata:
  labels:
    classification: "hemlig-secret"
    data-owner: "<responsible-team>"
    retention-policy: "<policy-name>"
```

A Kyverno policy enforces that these labels are present and valid.

### 8.4 Data Destruction

When storage is decommissioned or a PV is deleted:

- **Logical deletion**: Ceph zeroes blocks on delete (configurable)
- **Cryptographic erasure**: Destroying the encryption key renders data unrecoverable
- **Physical destruction**: When drives are decommissioned, they must be physically destroyed per FMV guidelines (degaussing + shredding for magnetic media; shredding for SSDs)
- **Chain of custody**: All destruction must be documented with witness signatures

---

## 9. Identity, Access, and Clearance Management

### 9.1 Identity Provider

**FreeIPA** (or Red Hat IdM) deployed inside the classified zone:

- Kerberos authentication for all services
- LDAP directory for user attributes including clearance level
- Integrated certificate authority for user certificates
- Synced with the organization's HR system via one-way import (data diode)
- Two-factor authentication: Smart card (PKI) + PIN for all access

### 9.2 Clearance-Integrated RBAC

The 12 cleared engineers are organized into roles:

| Role | Count | Kubernetes RBAC | Vault Policy | GitLab Role |
|------|-------|-----------------|-------------|-------------|
| **Platform Admin** | 2 | cluster-admin (break-glass only) | admin | Owner |
| **Platform Operator** | 4 | namespace-admin for infra namespaces | operator | Maintainer |
| **Application Deployer** | 3 | namespace-admin for app namespaces | deployer | Developer |
| **Security Auditor** | 2 | view-only cluster-wide + audit-log access | auditor | Reporter |
| **Security Officer** | 1 | view-only + policy-admin | security-admin | Owner |

#### Key Principles

- **No shared accounts**: Every action is attributable to an individual
- **No standing privileged access**: Cluster-admin requires break-glass procedure (Vault-issued short-lived credential + dual approval)
- **Session recording**: All terminal sessions on classified systems are recorded (using Teleport or similar)
- **Automatic timeout**: Sessions expire after 15 minutes of inactivity
- **Just-in-time access**: Privileged access is requested, approved by a second person, and time-limited (max 4 hours)

### 9.3 Kubernetes Authentication

```
User -> Smart Card (mTLS client cert) -> API Server
                                           |
                                           v
                                    OIDC / Webhook authn
                                    (validates against FreeIPA)
                                           |
                                           v
                                    RBAC authorization
                                    (role bindings per namespace)
```

Service accounts use projected service account tokens (short-lived, bound) rather than long-lived secrets.

### 9.4 Non-Cleared Personnel Boundary

The 28 non-cleared engineers:

- Have **zero access** to the classified zone (network, physical, logical)
- Work in the unclassified zone: write code, build images, run unclassified tests
- Their artifacts cross the air gap only after security review and data diode transfer
- They never see classified test data, configurations, or deployment manifests that contain classified information

---

## 10. Cryptographic Services

### 10.1 HSM-Backed Vault

HashiCorp Vault Enterprise (or the open-source version if licensing permits under defense procurement rules) deployed in HA mode:

| Function | Configuration |
|----------|--------------|
| **Unseal** | Auto-unseal via HSM (Thales Luna PKCS#11) |
| **Secret engine** | KV v2 for static secrets, PKI for certificates, Transit for encryption-as-a-service |
| **Authentication** | Kubernetes auth method (service accounts), LDAP (humans), AppRole (automation) |
| **Audit** | All operations logged to syslog -> SIEM |
| **HA** | 3-node Raft cluster with HSM-backed unseal |

### 10.2 PKI Architecture

```
Root CA (offline, HSM-protected, in safe)
  |
  +-- Intermediate CA: Infrastructure
  |     +-- Kubernetes API server certs
  |     +-- etcd peer/client certs
  |     +-- Node certificates
  |
  +-- Intermediate CA: Services
  |     +-- mTLS certificates for all services
  |     +-- Ingress TLS certificates
  |
  +-- Intermediate CA: Users
  |     +-- Smart card certificates
  |     +-- Code signing certificates
  |
  +-- Intermediate CA: Data Encryption
        +-- Disk encryption key-wrapping keys
        +-- Database encryption keys
```

- Root CA private key is generated and stored on HSM; never exported
- Root CA certificate validity: 10 years
- Intermediate CA validity: 3 years, rotated
- Leaf certificate validity: 90 days maximum, auto-rotated by Vault/cert-manager
- All certificates use RSA-4096 or ECDSA P-384 (per MUST guidance)
- Certificate revocation via CRL and OCSP (internal)

### 10.3 Encryption Standards

| Use Case | Algorithm | Key Management |
|----------|-----------|---------------|
| Data at rest (disk) | AES-256-XTS (LUKS2) | Key in Vault, wrapped by HSM |
| Data at rest (etcd) | AES-256-GCM | Kubernetes encryption provider, key in Vault |
| Data at rest (Ceph) | AES-256-XTS (dm-crypt) | Key in Vault, wrapped by HSM |
| Data in transit (mTLS) | TLS 1.3 (AES-256-GCM, ChaCha20-Poly1305) | Certificates from Vault PKI |
| Data in transit (Ceph) | Messenger v2 encryption | Ceph native |
| Code/image signing | ECDSA P-384 or RSA-4096 | Signing key in Vault/HSM |

### 10.4 MUST-Approved Cryptography

For HEMLIG/SECRET, MUST may require specific Swedish- or NATO-approved cryptographic implementations. Consult MUST early in the accreditation process. If MUST mandates national cryptographic devices (e.g., for specific communication links), these must be sourced through FMV.

---

## 11. Air-Gap Strategy and Supply Chain

### 11.1 Inbound Transfer Process

```
UNCLASSIFIED ZONE                     CLASSIFIED ZONE

1. Developer pushes code
   to unclassified GitLab

2. CI pipeline builds,
   scans, signs artifacts

3. Artifacts staged to
   transfer server

4. Security review of          -----> 5. Data diode transfers
   transfer manifest                     artifacts to transfer zone
   (human approval required)
                                      6. Transfer zone:
                                         - Malware scan (ClamAV + YARA)
                                         - Signature verification
                                         - SBOM analysis
                                         - Manual approval by cleared operator

                                      7. Artifacts promoted to
                                         production Harbor registry

                                      8. Flux CD deploys from
                                         approved artifacts
```

### 11.2 Outbound Transfer (Data Exfiltration Prevention)

**There is no electronic outbound path.** This is non-negotiable for HEMLIG/SECRET.

If data must leave the classified zone (e.g., for reporting to FMV):

1. Data is reviewed by the security officer
2. Explicit declassification or downgrading decision (if applicable)
3. Data is copied to approved removable media (encrypted USB, approved by MUST)
4. Media is physically transported via approved courier
5. Full chain of custody documentation
6. Media is tracked in an asset register

### 11.3 Software Supply Chain

#### Operating System

- **Recommendation**: Red Hat Enterprise Linux 9 (RHEL) or Ubuntu Pro with FIPS mode
- RHEL is preferred due to FIPS-validated cryptographic modules and SELinux maturity
- Base OS images are built with Packer, hardened per CIS RHEL 9 benchmark + STIGs
- Images are versioned, signed, and stored in the classified zone
- OS updates are transferred via data diode monthly (or as needed for critical CVEs)

#### Container Base Images

- Minimal base images: Red Hat UBI Micro, Distroless, or Alpine (hardened)
- All base images are rebuilt internally, not pulled from public registries
- SBOM generated for every image (Syft)
- Vulnerability scan (Trivy) with policy: zero critical, zero high unpatched

#### Kubernetes Components

- RKE2 air-gap bundle includes all system images
- Updated quarterly (or on critical CVE)
- Verified against RKE2 checksums and signatures

### 11.4 Patch Management

| Component | Frequency | Process |
|-----------|-----------|---------|
| OS (RHEL) | Monthly + emergency | Staged rollout: dev -> staging -> prod (within classified zone) |
| Kubernetes | Quarterly + emergency | RKE2 upgrade bundle via data diode |
| Container images | Monthly + emergency | Rebuild, scan, sign, transfer, deploy |
| Firmware (BIOS/BMC) | Bi-annually + emergency | Vendor-verified, transferred via approved media |
| HSM firmware | As needed | Vendor on-site with cleared escort |

---

## 12. Logging, Auditing, and SIEM

### 12.1 Log Architecture

```
All Nodes & Services
  |
  v
Fluentd / Fluent Bit (DaemonSet)
  |
  v
Kafka (internal, encrypted, HA)
  |
  +---> Elasticsearch (hot-warm-cold architecture)
  |       |
  |       +---> Kibana (dashboards, search)
  |
  +---> Long-term archive (Ceph S3, encrypted, WORM)
  |
  +---> SIEM (Splunk or Elastic SIEM)
          |
          +---> Alerting -> On-call pager (internal)
```

### 12.2 What Is Logged

| Source | Events |
|--------|--------|
| **Kubernetes API server** | All API calls (audit log level: RequestResponse for write, Metadata for read) |
| **Kubernetes kubelet** | Container lifecycle events |
| **Falco** | Suspicious syscalls, file access, network connections |
| **Vault** | All secret access, authentication, policy changes |
| **GitLab** | All git operations, merge requests, CI pipeline runs |
| **Harbor** | Image push/pull, scan results, replication |
| **FreeIPA** | Authentication, authorization, account changes |
| **OS** | SSH/PAM auth (auditd), sudo, file integrity (AIDE) |
| **Network** | Firewall logs, Cilium flow logs, DNS queries |
| **Physical** | Door access, CCTV events (metadata), alarm events |
| **Data diode** | Transfer logs (inbound side) |

### 12.3 Audit Log Protection

- Audit logs are written to a **separate, append-only** storage volume
- Log integrity is protected by **hash chains** (each log entry includes a hash of the previous entry)
- Logs are replicated to a second storage location (different rack)
- Log access is restricted to the Security Auditor and Security Officer roles
- Log deletion is **impossible** without physical storage destruction (WORM policy on Ceph RGW)
- Retention period: minimum 5 years (or as specified by FMV)

### 12.4 SIEM Correlation Rules

The SIEM must have detection rules for at least:

- Unauthorized API access attempts
- Privilege escalation (both OS and Kubernetes)
- Anomalous data access patterns
- Failed authentication brute force
- Container escape attempts (Falco alerts)
- Drift from GitOps-declared state
- Unauthorized process execution
- Network policy violations
- Time synchronization anomalies (potential log tampering)
- USB/removable media connection (should never happen on classified nodes)

### 12.5 Incident Response Integration

- SIEM alerts trigger incident response procedures
- Incident classification per Säkerhetsskyddslagen (security-impacting incident reporting to Säkerhetspolisen within mandated timeframes)
- Forensic capability: ability to snapshot nodes, dump memory, capture network traffic
- Chain of custody for digital evidence

---

## 13. Backup, Disaster Recovery, and Continuity

### 13.1 Backup Strategy

| Data | Method | Frequency | Retention | Storage |
|------|--------|-----------|-----------|---------|
| etcd snapshots | Automated CronJob | Every hour | 30 days | Encrypted Ceph S3 |
| Kubernetes manifests | Git (source of truth) | Continuous | Indefinite | GitLab + backup |
| Persistent volumes | Ceph RBD snapshots + Velero | Daily | 30 daily, 12 monthly | Ceph + off-site |
| Vault data | Vault Raft snapshots | Every 4 hours | 30 days | Encrypted Ceph S3 |
| GitLab | GitLab backup rake task | Daily | 90 days | Encrypted Ceph S3 |
| SIEM/Logs | Ceph S3 lifecycle | Continuous | 5 years | WORM Ceph S3 |
| Configuration (IaC) | Git | Continuous | Indefinite | GitLab |

### 13.2 Off-Site Backup

Off-site backups for HEMLIG/SECRET data must:

- Be stored in a facility with equivalent physical security classification
- Be encrypted with keys held in the primary HSM (backup HSM at off-site)
- Be transported by approved courier with chain-of-custody documentation
- Be tested for restoration quarterly

### 13.3 Disaster Recovery

| Scenario | RTO | RPO | Procedure |
|----------|-----|-----|-----------|
| Single node failure | 5 minutes | 0 (replicated) | Kubernetes auto-reschedules pods |
| Rack failure | 15 minutes | 0 (replicated across racks) | Pods reschedule; Ceph rebuilds |
| Control plane failure | 30 minutes | < 1 hour (etcd snapshot) | Restore etcd from snapshot, rejoin cluster |
| Total site loss | 48-72 hours | < 24 hours | Restore from off-site backup to secondary site |
| Encryption key loss | N/A | N/A | **Prevented** by HSM HA pair + offline backup of wrapped keys |

### 13.4 DR Testing

- **Quarterly**: Restore individual workloads from backup
- **Semi-annually**: Simulate control plane failure and recovery
- **Annually**: Full DR exercise including off-site restoration
- All DR tests documented and reviewed during accreditation audits

---

## 14. Operational Model and Staffing

### 14.1 Team Structure (12 Cleared Personnel)

```
Security Officer (Säkerhetsskyddschef liaison)     [1 person]
  Reports to: CISO / CEO
  Responsibilities: Policy, accreditation, incident classification,
                    transfer approvals, audit review

Platform Team Lead                                  [1 person]
  Responsibilities: Architecture decisions, FMV/MUST coordination,
                    capacity planning

Platform Engineers (Infrastructure & Kubernetes)    [4 persons]
  Responsibilities: Cluster operations, node lifecycle, storage,
                    networking, monitoring, patching

Application/DevSecOps Engineers                     [3 persons]
  Responsibilities: CI/CD pipeline, image builds, Flux/GitOps,
                    application support, secret management

Security Engineers / Auditors                       [2 persons]
  Responsibilities: SIEM operations, vulnerability management,
                    penetration testing, compliance monitoring,
                    log review

On-call / Backup                                    [1 person]
  Responsibilities: Cross-trained for all roles, covers absences
```

### 14.2 Shift Coverage

- **Business hours** (07:00-17:00): Full team available
- **On-call rotation**: 2 persons on call 24/7 (one platform, one security)
- **Physical access**: On-call personnel must be able to reach the facility within 2 hours
- **Minimum staffing**: At least 2 cleared personnel must be present for any physical access to the classified zone (two-person integrity rule)

### 14.3 Non-Cleared Engineers (28 Persons)

The 28 non-cleared engineers contribute from the unclassified zone:

- **Application development**: Write application code in unclassified GitLab
- **Unclassified testing**: Test with synthetic/unclassified data
- **Documentation**: Architecture docs (unclassified versions), API specs
- **Tooling**: Build and maintain CI/CD tooling, monitoring dashboards (unclassified instances)
- **Support**: Provide L2/L3 application support for unclassified issues; escalate classified issues to cleared team

### 14.4 Separation of Duties Matrix

| Action | Initiator | Approver | Executor |
|--------|-----------|----------|----------|
| Deploy application | App Engineer | Platform Engineer + Security Officer | Flux CD (automated) |
| Transfer artifact through diode | App Engineer (unclass) | Security Officer | Platform Engineer |
| Access production data | App Engineer | Security Officer | (time-limited access via Vault) |
| Modify network policy | Platform Engineer | Security Officer | Flux CD (automated) |
| Break-glass cluster-admin | Platform Engineer | Platform Lead + Security Officer | Vault (time-limited) |
| Decommission storage | Platform Engineer | Security Officer | Platform Engineer (witnessed) |
| Rotate HSM keys | Security Engineer | Security Officer + Platform Lead | Security Engineer |

### 14.5 Training Requirements

All 12 cleared personnel must complete:

- Säkerhetsskyddsutbildning (annually)
- Kubernetes security training (e.g., CKS certification encouraged)
- Incident response procedures training (bi-annually)
- TEMPEST awareness training (at onboarding)
- Social engineering awareness (annually)
- Specific platform operational procedures (at onboarding + on change)

---

## 15. Accreditation Roadmap

### 15.1 Accreditation Process Overview

FMV/MUST accreditation for a HEMLIG/SECRET system follows a structured process:

```
Phase 1: PREPARATION (Months 1-3)
  - Appoint Säkerhetsskyddschef and system security officer
  - Engage FMV accreditation team early
  - Conduct Säkerhetsskyddsanalys (security protection analysis)
  - Conduct Särskild säkerhetsskyddsbedömning (special security assessment)
  - Produce System Security Plan (SSP)
  - Produce TEMPEST assessment request

Phase 2: DESIGN & DOCUMENTATION (Months 3-8)
  - Architecture document (this document)
  - Security Target (ST) document
  - Risk assessment (per ISO 27005 / MUST methodology)
  - TEMPEST survey and mitigation plan
  - Physical security design and approval
  - Cryptographic plan (for MUST approval)
  - Operational security procedures
  - Incident response plan
  - Configuration management plan

Phase 3: IMPLEMENTATION (Months 6-14)
  - Build facility (if needed) with approved physical security
  - TEMPEST installation and verification
  - Hardware procurement and supply chain verification
  - Platform installation and hardening
  - Security controls implementation
  - Initial vulnerability assessment

Phase 4: TESTING & EVALUATION (Months 12-18)
  - Security testing (penetration testing by approved evaluator)
  - TEMPEST measurements by MUST-approved lab
  - Configuration audit against SSP
  - Operational readiness review
  - Tabletop exercises for incident scenarios
  - FMV security audit

Phase 5: ACCREDITATION DECISION (Months 16-20)
  - FMV reviews all documentation and test results
  - MUST reviews cryptographic implementation
  - Residual risk acceptance by system owner (systemägare)
  - Accreditation decision (godkännande) by FMV
  - Conditions or restrictions noted (villkor)

Phase 6: OPERATIONS & MAINTENANCE (Ongoing)
  - Continuous monitoring
  - Annual security review
  - Re-accreditation on significant change
  - Accreditation typically valid for 3 years (with annual review)
```

### 15.2 Key Deliverables for Accreditation

| Document | Description | Approver |
|----------|-------------|----------|
| Säkerhetsskyddsanalys | Identifies what needs protection and from what | Organization |
| System Security Plan (SSP) | Complete security architecture and controls | FMV |
| TEMPEST plan | Emanation security design and countermeasures | MUST |
| Cryptographic plan | All cryptographic usage and key management | MUST |
| Risk assessment | Threats, vulnerabilities, risks, mitigations | FMV |
| Physical security plan | Facility security design | FMV + Länsstyrelsen |
| Personnel security plan | Clearance management, roles, training | Organization + Säpo |
| Configuration management plan | How the system is maintained and changed | FMV |
| Incident response plan | How incidents are detected, responded to, reported | FMV + Säpo |
| Operational procedures | Day-to-day operating procedures | FMV |
| Test reports | Penetration test, TEMPEST measurement, audit results | FMV / MUST |

### 15.3 Timeline Estimate

Realistic timeline for a HEMLIG/SECRET platform accreditation: **18-24 months** from project initiation to accreditation decision. This assumes:

- Existing facility that can be upgraded (new build adds 6-12 months)
- Experienced team (new team adds 3-6 months for training)
- Early engagement with FMV/MUST (delays in engagement add directly to timeline)
- No novel technology requiring separate evaluation

---

## 16. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Residual Risk |
|----|------|-----------|--------|------------|---------------|
| R01 | Supply chain compromise of hardware | Medium | Critical | Vetted suppliers, firmware verification, TPM measured boot | Low |
| R02 | Insider threat from cleared personnel | Low-Medium | Critical | Two-person rule, session recording, least privilege, background monitoring | Low |
| R03 | Zero-day vulnerability in Kubernetes | Medium | High | Defense in depth, runtime monitoring (Falco), rapid patching via data diode, network segmentation | Medium |
| R04 | Loss of key personnel (12-person team) | Medium | High | Cross-training, documentation, knowledge management, pipeline for new clearances | Medium |
| R05 | HSM failure | Low | Critical | HA pair, offline backup keys, vendor SLA | Low |
| R06 | TEMPEST emanation leakage | Low | Critical | MUST-approved installation, periodic measurement, shielded facility | Low |
| R07 | Data diode misconfiguration | Low | Critical | Hardware-enforced (not software), regular verification | Very Low |
| R08 | Accreditation delay | Medium | High | Early FMV/MUST engagement, experienced security advisor | Medium |
| R09 | Air-gap bypass (covert channel) | Very Low | Critical | No wireless devices, TEMPEST, physical security, device control | Very Low |
| R10 | Ceph data loss | Low | High | 3x replication, daily snapshots, off-site backup, tested recovery | Low |
| R11 | Clearance revocation of critical staff | Low | High | Redundancy in roles, succession planning | Medium |
| R12 | Delayed patch deployment due to air-gap | Medium | Medium | Risk-based prioritization, monthly transfer cadence, emergency procedure for critical CVEs | Medium |

---

## 17. Appendices

### Appendix A: Technology Stack Summary

| Layer | Technology | Version Policy |
|-------|-----------|---------------|
| Hardware | HPE/Dell servers, Thales HSM | Vendor-supported, firmware pinned |
| OS | RHEL 9 (FIPS mode, CIS hardened) | Latest minor, patched monthly |
| Container runtime | containerd (via RKE2) | Bundled with RKE2 |
| Kubernetes | RKE2 | N-1 version policy |
| CNI | Cilium | Latest stable |
| Service mesh | Istio (evaluate) | Latest stable |
| Storage | Rook-Ceph | Latest stable with Rook operator |
| Secrets | HashiCorp Vault + HSM | Latest stable |
| PKI | Vault PKI + EJBCA (Root CA) | Latest stable |
| GitOps | Flux CD | Latest stable |
| CI/CD | GitLab (air-gapped) | Latest stable |
| Container registry | Harbor | Latest stable |
| Artifact registry | Nexus Repository | Latest stable |
| Monitoring | Prometheus + Grafana | Latest stable |
| Logging | Fluentd + Elasticsearch + Kibana | Latest stable |
| SIEM | Elastic SIEM or Splunk | Latest stable |
| Runtime security | Falco | Latest stable |
| Policy engine | Kyverno | Latest stable |
| Identity | FreeIPA | Latest stable |
| Backup | Velero + Ceph snapshots | Latest stable |
| Data diode | Advenica SecuriCDS | Vendor-managed firmware |
| Scanning | Trivy, Grype, ClamAV, YARA | Updated via data diode |

### Appendix B: Network Port Matrix (Classified Zone Internal)

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Workers | API Server | 6443 | TCP/TLS | Kubernetes API |
| API Server | etcd | 2379-2380 | TCP/mTLS | etcd client/peer |
| All nodes | CoreDNS | 53 | TCP/UDP | DNS resolution |
| All nodes | Chrony (NTP) | 123 | UDP | Time sync |
| All nodes | Fluentd | 24224 | TCP/TLS | Log shipping |
| Cilium agents | Cilium agents | 4240 | TCP | Health checks |
| Workers | Ceph MON | 6789, 3300 | TCP | Ceph monitor |
| Workers | Ceph OSD | 6800-7300 | TCP | Ceph data |
| All | Vault | 8200 | TCP/TLS | Secrets |
| All | Harbor | 443 | TCP/TLS | Container registry |
| All | GitLab | 443, 22 | TCP/TLS | Git, CI |
| All | FreeIPA | 443, 88, 389, 636 | TCP | LDAP, Kerberos |
| BMC network | BMC VIP | 443 | TCP/TLS | Out-of-band management |

All unlisted port combinations are **denied** by default at every enforcement point.

### Appendix C: Compliance Mapping

| Säkerhetsskyddslagen Requirement | Architecture Control |
|----------------------------------|---------------------|
| Informationssäkerhet (Information security) | Encryption at rest/transit, access control, air-gap, classification labeling |
| Fysisk säkerhet (Physical security) | Hardened facility, access control, CCTV, intrusion detection, TEMPEST |
| Personalsäkerhet (Personnel security) | Säpo clearances, RBAC mapped to clearance, two-person rule, training |
| Säkerhetsskyddsanalys (Security protection analysis) | Risk register, threat model, continuous monitoring |
| Tillträdesbegränsning (Access restriction) | Multi-factor physical access, network segmentation, RBAC, just-in-time access |
| Signalskydd (Signals protection) | MUST-approved crypto, HSM, TEMPEST countermeasures |
| IT-säkerhet (IT security) | CIS hardening, pod security, runtime monitoring, SIEM, vulnerability management |
| Rapportering av säkerhetsskyddsincidenter | SIEM alerting, incident response plan, Säpo reporting procedures |

### Appendix D: Glossary

| Term | Definition |
|------|-----------|
| **Säkerhetsskyddslagen** | The Swedish Protective Security Act (2018:585) |
| **Säkerhetsskyddsförordningen** | The Swedish Protective Security Ordinance (2021:955) |
| **Säkerhetsskyddschef** | Security Protection Officer; legally responsible person within the organization |
| **FMV** | Försvarets materielverk (Swedish Defence Materiel Administration) |
| **MUST** | Militära underrättelse- och säkerhetstjänsten (Swedish Military Intelligence and Security Service) |
| **Säpo** | Säkerhetspolisen (Swedish Security Service); handles personnel security vetting |
| **HEMLIG** | Swedish classification marking; sub-levels: Restricted, Confidential, Secret, Top Secret |
| **TEMPEST** | NATO specification for emanation security countermeasures |
| **SDIP-27** | NATO standard for TEMPEST facility zoning |
| **Data diode** | Hardware device enforcing one-way data transfer |
| **RKE2** | Rancher Kubernetes Engine 2 ("RKE Government"); security-focused Kubernetes distribution |
| **HSM** | Hardware Security Module; tamper-resistant device for cryptographic key management |
| **mTLS** | Mutual TLS; both client and server authenticate via certificates |
| **GitOps** | Operational model where Git is the single source of truth for declarative infrastructure |
| **WORM** | Write Once Read Many; storage policy preventing modification of written data |
| **CIS** | Center for Internet Security; publishes hardening benchmarks |
| **SBOM** | Software Bill of Materials; inventory of software components |

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-20 | Architecture Team | Initial draft |

**Review Required By:**
- [ ] Säkerhetsskyddschef
- [ ] CISO
- [ ] FMV Accreditation Officer
- [ ] MUST Signals Protection Advisor
- [ ] Platform Team Lead
- [ ] Legal Counsel

---

*This document is intended for internal use within the organization's accreditation process. Distribution is limited to personnel with appropriate clearance and need-to-know. The actual system implementation may require adjustments based on FMV/MUST feedback during the accreditation process.*
