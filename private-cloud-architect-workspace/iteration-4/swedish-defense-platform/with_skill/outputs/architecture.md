# Architecture: Private Cloud Platform for HEMLIG Classified Data Processing

## Swedish Defense Contractor -- Säkerhetsskydd-Compliant Kubernetes Platform

---

## 1. Executive Summary

This document defines the architecture for an air-gapped private cloud platform capable of processing HEMLIG (Secret) classified data under Swedish law. The platform is designed for accreditation by FMV/MUST, full compliance with Säkerhetsskyddslagen (2018:585), and operation exclusively by Säpo-cleared personnel within Swedish territory.

The architecture is Kubernetes-based, using RKE2 as the hardened distribution, deployed on bare-metal servers in a physically secured facility. All operations are air-gapped with no connectivity to lower-classification networks or the internet. Data ingress/egress is controlled via hardware data diodes and manual transfer procedures with strict chain-of-custody.

---

## 2. Regulatory and Compliance Framework

### 2.1 Primary Authority: Swedish National Framework

The following Swedish authorities and legal instruments govern this architecture. These are the **primary** requirements -- not US DISA STIGs, not NIST 800-53, not FedRAMP. Those may be referenced for technical detail where Swedish guidance is silent, but the Swedish framework always takes precedence.

| Authority / Instrument | Role |
|---|---|
| **Säkerhetsskyddslagen (2018:585)** | Primary legal framework for protective security (säkerhetsskydd) |
| **Säkerhetsskyddsförordningen (2021:955)** | Implementing regulation with detailed requirements |
| **FMV (Försvarets materielverk)** | Defense materiel administration; system accreditation authority for defense contractors |
| **MUST (Militära underrättelse- och säkerhetstjänsten)** | Military intelligence and security service; sets security requirements for military classified systems |
| **Säpo (Säkerhetspolisen)** | Swedish Security Service; manages personnel security clearance process |
| **MSB (Myndigheten för samhällsskydd och beredskap)** | Civil contingency agency; provides supplementary cybersecurity guidance |
| **FRA (Försvarets radioanstalt)** | Signals intelligence agency; approves cryptographic products for classified use |

### 2.2 Classification Level: HEMLIG

Sweden's classification hierarchy:

| Level | Description |
|---|---|
| ÖPPEN | Open / unclassified |
| BEGRÄNSAT HEMLIG | Restricted -- limited damage if disclosed |
| HEMLIG | Secret -- serious damage to national security if disclosed |
| KVALIFICERAT HEMLIG | Top Secret -- exceptionally grave damage if disclosed |

This platform processes **HEMLIG** data. Architectural controls are set accordingly:
- Physical air gap required (not logical isolation)
- All personnel must hold HEMLIG-level security clearance via Säpo
- Processing must occur within Sweden
- FRA-approved cryptographic products mandatory
- FMV/MUST accreditation required before operational use

### 2.3 International Context

As a Swedish defense contractor:
- **EU obligations**: NIS2 Directive compliance (Sweden is an EU member state), GDPR for any personal data processed alongside classified data, EUCS (EU Cybersecurity Certification Scheme) awareness
- **NATO alignment**: Sweden is a NATO member; interoperability with NATO SECRET (NS) standards must be considered for any future cross-national data sharing. NATO-approved crypto is required for NATO-classified data (distinct from national FRA-approved products)
- **Data sovereignty**: HEMLIG data must remain within Swedish territory at all times. No cloud provider, no foreign data center, no cross-border replication

### 2.4 Hierarchy of Standards Applied

1. **Säkerhetsskyddslagen + FMV/MUST requirements** -- primary, authoritative
2. **FRA cryptographic product approvals** -- mandatory for all encryption
3. **MSB cybersecurity guidelines** -- supplementary technical guidance
4. **CIS Benchmarks** -- technical hardening where Swedish guidance does not provide specific OS/application-level detail (validated against Swedish requirements before adoption)
5. **NIST 800-190 (Container Security)** -- reference for container-specific hardening patterns (informative, not authoritative)

---

## 3. Personnel and Operational Constraints

### 3.1 Staffing Model

| Category | Count | Clearance |
|---|---|---|
| Total engineering staff | 40 | Various |
| HEMLIG-cleared engineers | 12 | HEMLIG via Säpo |
| Uncleared engineers | 28 | No access to classified systems |

**Architectural implication**: Only the 12 cleared engineers can access, operate, or maintain the classified platform. This constrains:

- **On-call rotation**: Maximum 12 people in the rotation. With a sustainable 1-in-4 schedule, this means 3 on-call teams of 4. Plan for fatigue and burnout.
- **Knowledge concentration risk**: Bus factor is critically low. Cross-training across all platform components is mandatory.
- **Vendor/contractor access**: No vendor remote support. All vendor interaction must occur in unclassified contexts (documentation, training) or with cleared vendor personnel on-site.
- **Development workflow**: The 28 uncleared engineers can develop applications on unclassified networks. Code is transferred into the classified environment via the data import process (Section 7). Only cleared engineers perform deployment, testing, and operations on the classified platform.

### 3.2 Recommended Team Structure (12 Cleared Engineers)

| Role | Count | Responsibilities |
|---|---|---|
| Platform Lead / Architect | 1 | Architecture decisions, FMV/MUST liaison, accreditation documentation |
| Kubernetes Platform Engineers | 4 | Cluster operations, upgrades, GitOps, troubleshooting |
| Infrastructure / Bare-metal Engineers | 3 | Hardware, OS, storage (Ceph), networking, firmware |
| Security Engineer | 2 | Compliance automation, audit log review, vulnerability management, CDS operations |
| Application Support / DevOps | 2 | Assist application teams, CI/CD pipeline operation, image management |

All 12 engineers must be cross-trained on at least two roles.

---

## 4. Physical Infrastructure

### 4.1 Facility Requirements

The data center must be a **säkerhetsskyddat utrymme** (security-protected area) meeting HEMLIG requirements:

- **Physical access control**: Multi-factor entry (badge + biometric), mantrap, 24/7 surveillance, armed response
- **Intrusion detection**: Motion sensors, door sensors, CCTV with recording and retention per FMV/MUST requirements
- **TEMPEST/emanation security**: Equipment must meet SDIP-27 Zone B requirements at minimum. Evaluate Zone A for specific high-sensitivity workloads. Shielded room (Faraday cage) may be required depending on FMV/MUST risk assessment
- **Red/black separation**: Classified (red) and unclassified (black) cabling must be physically separated -- separate conduits, minimum separation distances per MUST guidance
- **Location**: Within Sweden. Preferably in a facility already accredited for HEMLIG processing
- **Power**: Redundant UPS + diesel generator. Minimum N+1 power distribution
- **Fire suppression**: Gas-based (FM-200/Novec) to protect equipment

### 4.2 Network Physical Separation

This is a true air gap. There is **no** network path from this environment to any unclassified network, the internet, or any lower-classification system.

```
                    ┌─────────────────────────────────────────────────┐
                    │          HEMLIG AIR-GAPPED ENVIRONMENT          │
                    │                                                 │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
                    │  │  Mgmt    │  │Production │  │ Storage  │     │
                    │  │ Network  │  │ Network   │  │ Network  │     │
                    │  └────┬─────┘  └────┬──────┘  └────┬─────┘     │
                    │       │             │              │            │
                    │  ┌────┴─────────────┴──────────────┴─────┐     │
                    │  │         Spine-Leaf Fabric              │     │
                    │  │    (Dedicated HEMLIG switches)         │     │
                    │  └───────────────────────────────────────-┘     │
                    │                                                 │
                    └──────────────────────┬──────────────────────────┘
                                           │
                              ┌─────────────┴──────────────┐
                              │   DATA DIODE / CDS ROOM    │
                              │  (Physically controlled)    │
                              │                             │
                              │  ┌────────────────────┐    │
                              │  │  Advenica           │    │
                              │  │  SecuriCDS DD1000   │    │
                              │  │  (One-way: IN only) │    │
                              │  └────────┬───────────┘    │
                              │           │                 │
                              │  ┌────────┴───────────┐    │
                              │  │  Import scanning    │    │
                              │  │  station            │    │
                              │  └────────────────────┘    │
                              └────────────────────────────┘
                                           │
                              ┌─────────────┴──────────────┐
                              │   UNCLASSIFIED NETWORK     │
                              │  (Separate facility/room)  │
                              └────────────────────────────┘
```

### 4.3 Hardware Bill of Materials (Reference Sizing)

For a platform supporting approximately 50-100 application workloads with moderate compute requirements:

| Component | Spec | Quantity | Notes |
|---|---|---|---|
| **Control plane nodes** | 2x Intel Xeon Gold 6448Y (32C), 512 GB ECC RAM, 2x 1.92 TB NVMe (OS mirror), 2x 25GbE | 3 | Dedicated to K8s control plane + etcd |
| **Worker nodes** | 2x Intel Xeon Gold 6448Y (32C), 1 TB ECC RAM, 2x 1.92 TB NVMe (OS), 2x 25GbE (workload), 2x 25GbE (storage) | 8 | Application workloads |
| **Storage nodes (Ceph)** | 2x Intel Xeon Silver 4416+ (20C), 512 GB ECC RAM, 2x 960 GB NVMe (OS), 8x 7.68 TB NVMe (OSD), 2x 25GbE (public), 2x 25GbE (cluster) | 5 | Ceph distributed storage |
| **Spine switches** | 100GbE capable, TEMPEST-rated or approved | 2 | Spine-leaf fabric |
| **Leaf switches** | 25GbE access, 100GbE uplink, TEMPEST-rated or approved | 4 | Spine-leaf fabric |
| **Management switch** | 1GbE, out-of-band | 1 | IPMI/iDRAC/iLO |
| **Data diode** | Advenica SecuriCDS DD1000 or equivalent FRA-approved | 1+ | One-way data import |
| **Import scanning station** | Hardened workstation with removable media reader | 2 | Media scanning before import |
| **KVM console** | TEMPEST-rated, local only | 2 | Emergency console access |

**Hardware sourcing**: All hardware must be procured through FMV-approved supply chains with chain-of-custody documentation. Firmware must be verified against vendor-published checksums before deployment. Consider Swedish/European suppliers where available (Advenica is Swedish).

---

## 5. Platform Architecture

### 5.1 Why Kubernetes (RKE2)

The recommendation is **RKE2** (Rancher Kubernetes Engine 2) for the following reasons:

| Criteria | RKE2 | Alternatives Considered |
|---|---|---|
| **Security posture** | CIS Kubernetes Benchmark compliant by default, SELinux support, FIPS-capable build | OpenShift (heavier, Red Hat licensing), vanilla kubeadm (more manual hardening), Talos (newer, less ecosystem) |
| **Air-gap support** | First-class air-gap installation with pre-packaged images and binaries | All can be air-gapped, but RKE2 has the most streamlined offline workflow |
| **Embedded etcd** | Built-in HA etcd, no separate etcd cluster management | kubeadm requires manual etcd setup |
| **Containerd runtime** | Uses containerd directly, no Docker dependency | Aligns with modern container runtime standards |
| **Simplicity** | Single binary, systemd-managed, minimal moving parts | Critical for a 12-person team that cannot afford operational complexity |
| **FLOSS** | Apache 2.0 licensed | No licensing cost; avoids vendor lock-in |

**Alternative consideration**: If the organization already has OpenShift experience or Red Hat support contracts, OpenShift may be appropriate -- it has Common Criteria certification which could simplify FMV/MUST accreditation. However, it adds licensing cost and operational complexity.

### 5.2 Cluster Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    HEMLIG KUBERNETES PLATFORM                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   CONTROL PLANE (3 nodes)                 │   │
│  │                                                           │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │   │
│  │  │ cp-1        │ │ cp-2        │ │ cp-3        │        │   │
│  │  │ RKE2 server │ │ RKE2 server │ │ RKE2 server │        │   │
│  │  │ etcd        │ │ etcd        │ │ etcd        │        │   │
│  │  │ API server  │ │ API server  │ │ API server  │        │   │
│  │  │ scheduler   │ │ scheduler   │ │ scheduler   │        │   │
│  │  │ controller  │ │ controller  │ │ controller  │        │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   WORKER NODES (8 nodes)                  │   │
│  │                                                           │   │
│  │  ┌────────┐┌────────┐┌────────┐┌────────┐               │   │
│  │  │ wk-1   ││ wk-2   ││ wk-3   ││ wk-4   │               │   │
│  │  │ RKE2   ││ RKE2   ││ RKE2   ││ RKE2   │               │   │
│  │  │ agent  ││ agent  ││ agent  ││ agent  │               │   │
│  │  └────────┘└────────┘└────────┘└────────┘               │   │
│  │  ┌────────┐┌────────┐┌────────┐┌────────┐               │   │
│  │  │ wk-5   ││ wk-6   ││ wk-7   ││ wk-8   │               │   │
│  │  │ RKE2   ││ RKE2   ││ RKE2   ││ RKE2   │               │   │
│  │  │ agent  ││ agent  ││ agent  ││ agent  │               │   │
│  │  └────────┘└────────┘└────────┘└────────┘               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 CEPH STORAGE (5 nodes)                    │   │
│  │                                                           │   │
│  │  ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐     │   │
│  │  │ ceph-1 ││ ceph-2 ││ ceph-3 ││ ceph-4 ││ ceph-5 │     │   │
│  │  │ OSD x8 ││ OSD x8 ││ OSD x8 ││ OSD x8 ││ OSD x8 │     │   │
│  │  │ MON    ││ MON    ││ MON    ││        ││        │     │   │
│  │  │ MGR    ││ MGR    ││ MGR    ││        ││        │     │   │
│  │  └────────┘└────────┘└────────┘└────────┘└────────┘     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Operating System

**SUSE Linux Enterprise Micro (SLE Micro)** or **Rocky Linux 9** hardened to CIS Level 2 (validated against FMV/MUST requirements):

- Immutable OS preferred (SLE Micro) to reduce attack surface and simplify patching
- SELinux enforcing mode mandatory
- LUKS2 full-disk encryption on all nodes (FRA-approved if specific crypto modules are mandated)
- Minimal package set -- no GUI, no unnecessary services
- Automated compliance scanning via OpenSCAP with Swedish-adapted CIS profiles
- Kernel hardening: sysctl parameters per CIS benchmark, kernel module blacklisting

### 5.4 Container Networking: Cilium

**Cilium** is the recommended CNI for this platform:

- eBPF-based -- high performance with deep visibility
- Mandatory **default-deny** network policies across all namespaces
- Transparent encryption (WireGuard or IPsec) for all pod-to-pod traffic within the cluster
- Network policy enforcement at L3/L4/L7
- Hubble for network flow observability (critical for audit requirements)
- No dependency on iptables (more predictable, better performance)
- FLOSS (Apache 2.0)

Network policy baseline:
```yaml
# Applied to every namespace - deny all ingress/egress by default
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny-all
spec:
  endpointSelector: {}
  ingress: []
  egress: []
```

Explicit allow rules are then added per-workload based on documented data flow requirements.

### 5.5 Storage: Rook-Ceph

**Rook-Ceph** provides software-defined storage integrated with Kubernetes:

- **Block storage (RBD)**: For database workloads, persistent volumes requiring high IOPS
- **File storage (CephFS)**: For shared filesystem access across pods
- **Object storage (RGW)**: S3-compatible object storage for application data and backups
- **Encryption at rest**: Ceph OSD encryption with dm-crypt (LUKS2), keys managed internally
- **Replication**: Minimum 3x replication for all data (tolerates 2 node failures)
- **Storage classes**: Defined for different performance tiers (NVMe-only fast tier, mixed standard tier)
- **FLOSS**: Apache 2.0 / LGPL

### 5.6 Platform Services Stack

All services deployed within the air-gapped cluster:

| Service | Component | Purpose |
|---|---|---|
| **Container Registry** | Harbor | Air-gapped image registry with vulnerability scanning (Trivy), image signing, RBAC |
| **GitOps** | ArgoCD | Declarative deployment from internal Git; drift detection and reconciliation |
| **Git Server** | Gitea | Lightweight self-hosted Git for classified code and configuration |
| **Secret Management** | HashiCorp Vault (community) | Secrets storage, dynamic credentials, PKI, encryption as a service |
| **Certificate Authority** | Vault PKI + cert-manager | Internal PKI for all TLS certificates; automated rotation |
| **Identity / SSO** | Keycloak | SAML/OIDC identity provider; integrates with central directory if available |
| **Policy Engine** | Kyverno | Admission controller enforcing security policies (image sources, resource limits, labels, pod security) |
| **Monitoring** | Prometheus + Alertmanager | Metrics collection and alerting |
| **Dashboards** | Grafana | Visualization for infrastructure and application metrics |
| **Log Aggregation** | Loki + Promtail | Centralized logging with long-term retention for audit compliance |
| **Runtime Security** | Falco + Tetragon | Syscall monitoring, runtime behavior alerting |
| **Vulnerability Scanning** | Trivy (via Harbor + CronJobs) | Container image and filesystem vulnerability scanning |
| **Backup** | Velero + Rook-Ceph snapshots | Kubernetes resource and persistent volume backup |
| **DNS** | CoreDNS (built-in) | Internal cluster DNS |
| **Load Balancing** | MetalLB | Bare-metal load balancer for services exposed within the classified network |
| **IPAM/DCIM** | NetBox | Infrastructure documentation, IP address management |

---

## 6. Network Architecture

### 6.1 Network Segmentation

Four physically separated networks within the air-gapped environment:

| Network | Purpose | VLAN | Subnet (example) | Access |
|---|---|---|---|---|
| **Management** | IPMI/iDRAC, OS provisioning, Ansible | VLAN 10 | 10.10.10.0/24 | Platform engineers only |
| **Cluster** | K8s API, etcd, inter-node communication | VLAN 20 | 10.10.20.0/24 | K8s nodes only |
| **Workload** | Application pod traffic, service mesh | VLAN 30 | 10.10.30.0/16 | Cilium-managed |
| **Storage** | Ceph public + cluster networks | VLAN 40/41 | 10.10.40.0/24, 10.10.41.0/24 | Ceph + K8s nodes |

### 6.2 Spine-Leaf Fabric

- 2 spine switches (100GbE) for redundancy
- 4 leaf switches (25GbE access ports, 100GbE uplinks)
- ECMP (Equal-Cost Multi-Path) for load distribution
- All links redundant -- every node dual-homed to two leaf switches
- No spanning tree -- pure L3 fabric with BGP or OSPF
- All switch management on dedicated out-of-band management network

### 6.3 Firewall and Access Control

- Host-based firewall (nftables) on every node as defense-in-depth
- Cilium network policies for all east-west pod traffic
- No north-south traffic to/from the internet (air-gapped)
- Management network access restricted by source IP to jump hosts only
- All SSH access via hardened jump host with MFA, session recording, and individual accountability

---

## 7. Air-Gap Operations

### 7.1 Data Import Process

All data entering the classified environment follows a strict chain-of-custody:

```
Developer writes code    Unclassified review     Media preparation       Data diode import      Classified registry
on unclassified network → and approval gate    → write to approved     → one-way transfer    → Harbor scans and
                          (security officer)      removable media         into HEMLIG network    stores images
                                                  (encrypted, checksummed)
```

**Steps in detail:**

1. **Development** occurs on the unclassified network by any of the 40 engineers
2. **Code review** and security review on the unclassified side
3. **Build and package** container images, Helm charts, and artifacts on the unclassified side
4. **Generate SBOMs** (Syft) and vulnerability reports (Grype/Trivy) on the unclassified side
5. **Sign artifacts** with cosign using the organization's signing key
6. **Write to approved removable media** (encrypted USB or optical media per FMV/MUST approved media types)
7. **Chain-of-custody documentation** -- who prepared the media, what it contains, checksums, approval signatures
8. **Physical transfer** by cleared personnel to the data diode import room
9. **Import scanning station** -- malware scan, checksum verification, signature verification
10. **Data diode transfer** (Advenica SecuriCDS or equivalent) -- hardware-enforced one-way transfer into the HEMLIG network
11. **Harbor registry** on the classified side receives images, runs Trivy scan, verifies signatures
12. **Cleared engineers** deploy via ArgoCD from the classified Gitea repository

### 7.2 Data Export (Downgrade/Release)

Data leaving the HEMLIG environment is subject to **sanitization review and formal release procedures** per Säkerhetsskyddslagen:

1. Request for data export submitted with justification
2. Security officer reviews content for classification markings and sensitive information
3. Formal downgrade decision documented and signed by authorized personnel
4. Data written to approved media with chain-of-custody
5. Transfer through controlled process (no data diode -- manual only for export)
6. Full audit trail retained

### 7.3 Software Supply Chain for Air-Gap

Maintaining an air-gapped platform requires offline mirrors of all dependencies:

| Dependency | Offline Solution |
|---|---|
| OS packages (RPM/DEB) | Local mirror via Pulp or Uyuni; periodic sync via media import |
| Container images | Harbor registry; images imported via data diode |
| Helm charts | ChartMuseum or Harbor OCI; bundled with image imports |
| RKE2 binaries/upgrades | Pre-packaged tarballs from Rancher; imported via media |
| Ansible collections | Local Galaxy mirror or bundled collections |
| Ceph packages | Included in OS mirror |
| Vulnerability databases | Trivy offline DB updates imported periodically |
| Certificate revocation | Internal CRL distribution via Vault PKI |

### 7.4 Offline Kubernetes Deployment Pattern

```
# RKE2 air-gap installation (pre-staged on import)
# 1. Stage images tarball
cp rke2-images.linux-amd64.tar.zst /var/lib/rancher/rke2/agent/images/

# 2. Install RKE2 binary
INSTALL_RKE2_ARTIFACT_PATH=/opt/rke2-artifacts sh install.sh

# 3. Configure RKE2 for air-gap
cat > /etc/rancher/rke2/config.yaml <<EOF
system-default-registry: "harbor.hemlig.internal:443"
private-registry: "/etc/rancher/rke2/registries.yaml"
selinux: true
secrets-encryption: true
audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
EOF
```

---

## 8. Security Architecture

### 8.1 Encryption

| Layer | Mechanism | Notes |
|---|---|---|
| **Disk at rest** | LUKS2 (dm-crypt) on all OS disks and Ceph OSDs | FRA-approved crypto modules if mandated |
| **etcd at rest** | Kubernetes secrets encryption (AES-CBC or AES-GCM) | Encryption config applied at RKE2 level |
| **Pod-to-pod transit** | Cilium WireGuard transparent encryption | All inter-node pod traffic encrypted |
| **Service TLS** | cert-manager + Vault PKI | Automated certificate issuance and rotation |
| **Vault storage** | Vault auto-unseal with local KMS or Shamir keys | Shamir key shares held by separate cleared individuals |
| **Backup encryption** | Velero with encrypted backend (Ceph RGW with server-side encryption) | Backup keys managed in Vault |

**FRA crypto requirement**: Verify with FRA whether specific approved cryptographic modules or products are mandated for HEMLIG. If FRA requires specific HSMs or crypto accelerators, integrate them for TLS termination and disk encryption key management.

### 8.2 Identity and Access Control

- **Keycloak** as the central identity provider
- All access to Kubernetes API authenticated via OIDC tokens from Keycloak
- **RBAC** with least-privilege: namespace-scoped roles, no cluster-admin for routine operations
- **Need-to-know enforcement**: Having HEMLIG clearance does not grant access to all HEMLIG data. RBAC policies enforce project-level isolation
- **MFA required** for all authentication (hardware tokens -- YubiKey or equivalent)
- **Individual accountability**: Shared accounts prohibited. Every action traceable to a named individual
- **Service accounts**: Minimal permissions, short-lived tokens, regularly rotated

### 8.3 Admission Control and Policy Enforcement (Kyverno)

Mandatory policies enforced at admission:

| Policy | Effect |
|---|---|
| Images must come from `harbor.hemlig.internal` only | Block images from any other registry |
| Images must be signed with organization cosign key | Block unsigned images |
| No privileged containers | Block `privileged: true` |
| No host network/PID/IPC namespace | Block host namespace access |
| Read-only root filesystem required | Enforce `readOnlyRootFilesystem: true` |
| Resource limits mandatory | Block pods without CPU/memory limits |
| No `latest` tag | Require explicit image digests or semver tags |
| Mandatory labels (classification, owner, project) | Block resources without required metadata |
| Restrict volume types | Only allow PVC, configMap, secret, emptyDir |
| Automount service account token disabled by default | Reduce credential exposure |

### 8.4 Audit and Logging

**Audit everything, retain everything.**

- **Kubernetes audit logging**: All API server requests logged at `RequestResponse` level for write operations, `Metadata` level for reads
- **Falco**: Runtime syscall monitoring on all nodes -- alerts on unexpected process execution, file access, network connections
- **Tetragon**: eBPF-based security observability for process lifecycle, file access, and network events
- **Loki**: All logs aggregated centrally with long-term retention (minimum per FMV/MUST requirements -- typically 5+ years for classified systems)
- **Hubble (Cilium)**: Network flow logs for all pod-to-pod and pod-to-service communication
- **OS-level audit**: Linux auditd on all nodes capturing privileged operations, file access to sensitive paths, user authentication events
- **Immutable log storage**: Audit logs written to append-only Ceph RGW bucket with object lock (WORM) to prevent tampering
- **Log integrity**: Logs cryptographically chained (hash chain) to detect any deletion or modification

### 8.5 Supply Chain Security

- All container images built from known base images on the unclassified side
- **SBOM generation** (Syft) for every image before import
- **Vulnerability scanning** (Trivy) on both sides -- pre-import and post-import
- **Image signing** (cosign) with organizational key; Kyverno enforces signatures
- **Binary verification**: All imported binaries (RKE2, Ceph, OS packages) verified against published checksums and GPG signatures
- **Hardware supply chain**: Procurement through FMV-approved channels, tamper-evident packaging, firmware hash verification on receipt

---

## 9. Automation and GitOps

### 9.1 Infrastructure as Code

| Layer | Tool | Repository |
|---|---|---|
| **Bare-metal provisioning** | Ansible + MAAS (or Foreman) | `infra-provisioning` in classified Gitea |
| **OS hardening** | Ansible roles (CIS benchmark implementation) | `os-hardening` in classified Gitea |
| **Kubernetes bootstrap** | Ansible + RKE2 installer | `k8s-bootstrap` in classified Gitea |
| **Ceph deployment** | Rook-Ceph operator (Helm via ArgoCD) | `platform-services` in classified Gitea |
| **Platform services** | ArgoCD ApplicationSets (Helm/Kustomize) | `platform-services` in classified Gitea |
| **Application deployment** | ArgoCD Applications | `app-deployments` in classified Gitea |
| **Network configuration** | Ansible for switch configuration | `network-config` in classified Gitea |

### 9.2 GitOps Workflow

```
Classified Gitea                ArgoCD                    Kubernetes Cluster
    │                              │                            │
    │  Push to main branch         │                            │
    ├─────────────────────────────>│                            │
    │                              │  Detect drift / new commit│
    │                              ├───────────────────────────>│
    │                              │  Apply manifests           │
    │                              │  (sync)                    │
    │                              ├───────────────────────────>│
    │                              │                            │
    │                              │  Health check              │
    │                              │<───────────────────────────┤
    │                              │                            │
```

- All changes go through Git (merge request review by cleared engineers)
- ArgoCD syncs automatically on merge to main
- Drift detection alerts if cluster state diverges from Git
- Manual sync gate for critical platform components (control plane, security services)

### 9.3 Ansible Automation

Ansible is used for all infrastructure below Kubernetes:

- **Inventory**: Static inventory in Git (air-gapped -- no dynamic inventory from external sources)
- **AWX** (FLOSS upstream of Ansible Automation Platform): Web UI for job execution, scheduling, credential management, audit trail
- **Vault integration**: Ansible Vault for sensitive variables; HashiCorp Vault for dynamic credentials
- **Molecule testing**: Where possible, roles tested on unclassified side before import
- **Collections**: Bundled and imported offline -- `community.general`, `ansible.posix`, `kubernetes.core`

---

## 10. Monitoring and Observability

### 10.1 Metrics Stack

```
┌─────────────┐    ┌────────────────┐    ┌──────────┐
│ Prometheus   │───>│ Alertmanager   │───>│ On-call  │
│ (scrape all  │    │ (route alerts) │    │ (PagerDuty│
│  targets)    │    └────────────────┘    │  or local)│
└──────┬──────┘                          └──────────┘
       │
       v
┌──────────────┐
│ Grafana      │
│ (dashboards) │
└──────────────┘
```

- **Prometheus**: Scrapes all Kubernetes components, node exporters, Ceph exporters, application metrics
- **Alertmanager**: Routes alerts to on-call. In air-gapped environment, alerts go to internal paging system or physical notification systems
- **Grafana**: Dashboards for cluster health, node resources, Ceph storage, application metrics, security events
- **Thanos** (optional): Long-term metrics storage on Ceph RGW for capacity planning and historical analysis

### 10.2 Logging Stack

- **Promtail** on every node: Ships OS and container logs to Loki
- **Loki**: Centralized log aggregation with label-based querying
- **Retention**: Configured per FMV/MUST requirements (minimum 5 years for security-relevant logs)
- **Audit logs**: Separate Loki tenant for audit logs with stricter access control and immutable storage backend

### 10.3 Hardware Monitoring

- **IPMI/Redfish exporters**: Hardware health (temperature, fan speed, disk health, PSU status) exposed to Prometheus
- **Ceph health**: Ceph native health checks surfaced to Prometheus and Grafana
- **Network equipment**: SNMP monitoring of switches via SNMP exporter (if switches support it) or syslog forwarding

---

## 11. Disaster Recovery and Business Continuity

### 11.1 RTO/RPO Targets

| Component | RPO | RTO | Strategy |
|---|---|---|---|
| Kubernetes control plane | 0 (HA) | Automatic failover | 3-node etcd cluster, automatic leader election |
| Application workloads | 0 (HA) | Automatic rescheduling | Pod anti-affinity, PDB, multi-replica deployments |
| Persistent data (Ceph) | 0 (replicated) | Automatic recovery | 3x replication, self-healing on node failure |
| Platform services | 1 hour | 4 hours | Velero backup + ArgoCD redeploy |
| Full platform rebuild | 24 hours | 48 hours | Ansible automation + documented procedures |

### 11.2 Backup Strategy

- **Velero**: Kubernetes resource backup (etcd snapshots, namespace exports) to Ceph RGW
- **Ceph snapshots**: Point-in-time snapshots of RBD volumes
- **etcd snapshots**: Automated periodic snapshots stored on separate Ceph pool
- **Offline backup copies**: Periodic encrypted backup to removable media stored in a physically separate secure location (different fire compartment or building, within the same accredited facility)
- **Backup testing**: Monthly restoration tests documented and reported

### 11.3 Node Failure Tolerance

| Failure | Impact | Recovery |
|---|---|---|
| 1 control plane node | No impact (etcd quorum maintained) | Replace and rejoin automatically |
| 2 control plane nodes | **Platform degraded** -- etcd loses quorum | Emergency: restore from backup or repair second node |
| 1 worker node | Pods rescheduled to remaining 7 workers | Replace node via Ansible |
| 2 worker nodes | Pods rescheduled; capacity reduced | Priority: restore capacity |
| 1 Ceph node | No data loss (3x replication) | Ceph self-heals by rebalancing |
| 2 Ceph nodes | No data loss (3x replication, 5 nodes) | Ceph rebalances; reduced redundancy until restored |

---

## 12. Accreditation Approach

### 12.1 Accreditation Lifecycle with FMV/MUST

Following the universal accreditation pattern adapted for Swedish defense:

1. **Categorize**: HEMLIG classification for all data processed on the platform. Document data types, threat model, and risk assessment per Säkerhetsskyddslagen requirements.

2. **Select controls**: Map FMV/MUST security requirements to architectural controls. Document how each requirement is met. Cross-reference with MSB guidance where applicable.

3. **Implement**: Build the platform as described in this architecture. Every control must be demonstrably implemented and testable.

4. **Assess**: FMV/MUST (or their delegated assessors) conduct independent assessment. Provide:
   - Architecture documentation (this document)
   - Security controls mapping
   - Test results (penetration testing, vulnerability scans, compliance scans)
   - Operational procedures
   - Personnel security documentation (all operators are Säpo-cleared)

5. **Authorize**: FMV/MUST grants driftgodkännande (operational approval) for the platform at HEMLIG level.

6. **Monitor**: Continuous compliance monitoring via:
   - OpenSCAP automated scans (daily)
   - Falco/Tetragon runtime monitoring (continuous)
   - Trivy vulnerability scanning (on every image import + weekly full scan)
   - Quarterly compliance review by security officer
   - Annual accreditation review with FMV/MUST

### 12.2 Documentation Deliverables for Accreditation

| Document | Content |
|---|---|
| **Säkerhetsskyddsanalys** | Threat and risk analysis per Säkerhetsskyddslagen |
| **System Security Plan** | This architecture document + detailed control mappings |
| **Physical Security Plan** | Facility design, access control, TEMPEST measures |
| **Personnel Security Plan** | Clearance requirements, access procedures, training |
| **Operational Security Procedures** | Runbooks, incident response, change management |
| **Data Flow Documentation** | All data paths including import/export procedures |
| **Compliance Evidence** | Automated scan results, test reports, audit logs |
| **Continuity Plan** | DR/BC procedures, backup strategy, recovery testing results |

### 12.3 Automated Compliance

Design for continuous compliance from day one:

```yaml
# Example: OpenSCAP scan as a Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: openscap-compliance-scan
  namespace: security
spec:
  schedule: "0 2 * * *"  # Daily at 02:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: openscap
            image: harbor.hemlig.internal/security/openscap-scanner:latest
            command: ["oscap", "xccdf", "eval", "--profile", "cis-level2",
                     "--results", "/results/scan-$(date +%Y%m%d).xml",
                     "--report", "/results/scan-$(date +%Y%m%d).html",
                     "/content/ssg-rhel9-ds.xml"]
            volumeMounts:
            - name: results
              mountPath: /results
          volumes:
          - name: results
            persistentVolumeClaim:
              claimName: compliance-results
          restartPolicy: OnFailure
```

---

## 13. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-6)
- Facility preparation and physical security validation
- Hardware procurement through approved supply chain, receipt verification
- Network fabric installation (spine-leaf, cabling with red/black separation)
- Base OS installation and hardening on all nodes
- Ceph cluster deployment and validation

### Phase 2: Platform (Weeks 7-12)
- RKE2 cluster bootstrap (control plane + workers)
- Cilium CNI deployment with default-deny policies
- Rook-Ceph integration with Kubernetes
- Harbor registry deployment
- Vault deployment and PKI initialization
- ArgoCD and Gitea deployment

### Phase 3: Security and Compliance (Weeks 13-18)
- Kyverno policy deployment and validation
- Falco/Tetragon runtime security deployment
- OpenSCAP compliance automation
- Audit logging pipeline (Loki + immutable storage)
- Monitoring stack (Prometheus, Grafana, Alertmanager)
- Data diode integration and import/export procedure development
- Penetration testing

### Phase 4: Accreditation (Weeks 19-26)
- Documentation finalization
- FMV/MUST pre-assessment meetings
- Independent assessment
- Remediation of findings
- Driftgodkännande (operational approval)

### Phase 5: Onboarding (Weeks 27-30)
- First application workloads deployed
- Operations team training and runbook validation
- DR testing
- Handover to steady-state operations

**Total estimated timeline: 7-8 months** from hardware procurement to first classified workload.

---

## 14. Risk Register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Key person dependency (12 cleared engineers) | High | Medium | Cross-training, documentation, rotation across roles |
| FRA crypto product requirement delays procurement | Medium | Medium | Engage FRA early in Phase 1; identify approved products before hardware procurement |
| FMV/MUST accreditation findings require rearchitecture | High | Low | Engage FMV/MUST from Phase 1; iterative reviews, not big-bang assessment |
| Hardware supply chain delays (TEMPEST-rated equipment) | Medium | Medium | Order long-lead items first; identify alternative approved suppliers |
| Ceph operational complexity for small team | Medium | Medium | Extensive automation, Rook operator handles day-2, cross-training |
| Air-gap software supply chain lag (CVE patching) | Medium | High | Defined SLA for emergency patch imports; pre-staged critical updates |
| Clearance processing delays for new hires | High | Medium | Begin Säpo clearance process 6+ months before needed |

---

## 15. Architectural Decision Records (ADRs)

### ADR-001: Kubernetes distribution -- RKE2
- **Decision**: Use RKE2 over OpenShift, kubeadm, or Talos
- **Rationale**: Best air-gap support, CIS-hardened by default, FLOSS, manageable complexity for 12-person team
- **Tradeoff**: No Common Criteria certification (unlike OpenShift). Mitigated by comprehensive hardening and FMV/MUST assessment

### ADR-002: CNI -- Cilium
- **Decision**: Use Cilium over Calico or Canal
- **Rationale**: eBPF performance, transparent encryption, Hubble observability, strong network policy enforcement
- **Tradeoff**: More complex than Calico. Mitigated by team training and operational runbooks

### ADR-003: Storage -- Rook-Ceph
- **Decision**: Use Rook-Ceph over Longhorn or dedicated SAN
- **Rationale**: Unified block/file/object storage, Kubernetes-native management, FLOSS, scales with cluster
- **Tradeoff**: Operational complexity. Mitigated by Rook operator automation and dedicated storage nodes

### ADR-004: GitOps -- ArgoCD
- **Decision**: Use ArgoCD over Flux
- **Rationale**: Mature UI for operational visibility, broad adoption, strong RBAC, good air-gap support
- **Tradeoff**: Heavier than Flux. Acceptable given the need for operational UI for cleared operators

### ADR-005: Policy engine -- Kyverno
- **Decision**: Use Kyverno over OPA/Gatekeeper
- **Rationale**: Kubernetes-native policy language (no Rego), easier to write and review policies, mutation support
- **Tradeoff**: Less expressive than Rego for complex logic. Acceptable for our policy requirements

### ADR-006: Data diode -- Advenica SecuriCDS
- **Decision**: Use Advenica (Swedish manufacturer) for data diode
- **Rationale**: Swedish company with FMV/FRA relationships, hardware-enforced one-way transfer, established in Swedish defense sector
- **Tradeoff**: Vendor dependency for a critical component. Mitigated by also designing manual media import as fallback

---

## 16. Cost Estimate (Annual, Rough Order of Magnitude)

| Category | Estimated Annual Cost (SEK) | Notes |
|---|---|---|
| Hardware (amortized over 5 years) | 3,000,000 - 5,000,000 | Servers, switches, data diode, TEMPEST measures |
| Facility (rack space, power, cooling in accredited DC) | 1,500,000 - 3,000,000 | Depends on existing facility vs. new build |
| Software licensing | 500,000 - 1,000,000 | Primarily FLOSS; costs for SUSE/Red Hat support, possible Advenica support |
| Personnel (12 cleared engineers, allocated portion) | 12,000,000 - 18,000,000 | Largest cost; security-cleared engineers command premium |
| Accreditation and compliance | 500,000 - 1,000,000 | FMV/MUST assessment, penetration testing, documentation |
| **Total estimated annual cost** | **17,500,000 - 28,000,000** | ~1.75M - 2.8M EUR |

The FLOSS-first approach saves significantly on licensing compared to a VMware + commercial Kubernetes + proprietary storage stack, which could add 3-5 MSEK annually in licensing alone.

---

## Appendix A: Technology Stack Summary

| Layer | Technology | License |
|---|---|---|
| Operating System | SLE Micro or Rocky Linux 9 | SUSE subscription / FLOSS |
| Container Runtime | containerd | Apache 2.0 |
| Kubernetes | RKE2 | Apache 2.0 |
| CNI | Cilium | Apache 2.0 |
| Storage | Rook-Ceph | Apache 2.0 / LGPL |
| Container Registry | Harbor | Apache 2.0 |
| GitOps | ArgoCD | Apache 2.0 |
| Git | Gitea | MIT |
| Secrets | HashiCorp Vault (community) | BSL (evaluate; use community edition or consider OpenBao as FLOSS fork) |
| PKI | cert-manager + Vault | Apache 2.0 |
| Identity | Keycloak | Apache 2.0 |
| Policy | Kyverno | Apache 2.0 |
| Monitoring | Prometheus + Grafana + Loki | Apache 2.0 / AGPL (Grafana) |
| Runtime Security | Falco + Tetragon | Apache 2.0 |
| Vulnerability Scanning | Trivy | Apache 2.0 |
| Backup | Velero | Apache 2.0 |
| Load Balancing | MetalLB | Apache 2.0 |
| Automation | Ansible + AWX | GPL / Apache 2.0 |
| IPAM/DCIM | NetBox | Apache 2.0 |
| Data Diode | Advenica SecuriCDS | Proprietary (Swedish manufacturer) |

## Appendix B: Glossary of Swedish Terms

| Swedish | English |
|---|---|
| Säkerhetsskyddslagen | Protective Security Act |
| Säkerhetsskydd | Protective security |
| Försvarsindustri | Defense industry |
| FMV (Försvarets materielverk) | Defense Materiel Administration |
| MUST (Militära underrättelse- och säkerhetstjänsten) | Military Intelligence and Security Service |
| Säpo (Säkerhetspolisen) | Swedish Security Service |
| MSB (Myndigheten för samhällsskydd och beredskap) | Swedish Civil Contingencies Agency |
| FRA (Försvarets radioanstalt) | National Defence Radio Establishment |
| HEMLIG | Secret (classification level) |
| KVALIFICERAT HEMLIG | Top Secret (classification level) |
| BEGRÄNSAT HEMLIG | Restricted (classification level) |
| Driftgodkännande | Operational approval / Authority to Operate |
| Säkerhetsskyddsanalys | Protective security analysis |
| Säkerhetsskyddat utrymme | Security-protected area |
