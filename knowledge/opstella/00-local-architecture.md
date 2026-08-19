# Local Infrastructure Context — Opstella Knowledge

> Source: Architecture/ docs + DEV/SIT install trackers (`kube-cluster/deltron/{opstella-installation,sit-opstella-installation}/CLAUDE.md`); Orion re-inspected 2026-05-19
> Updated: 2026-07-03
> Coverage: current per-env topology (DEV/SIT/CUSTOMER) + Orion live; detailed hardware tables are the pre-rebuild baseline (prior art), see note in Cluster Topology

## Scope & Boundary

**This file describes INTERNAL Opsta Opstella infrastructure only:**
- Deltron — one Dell R6615 ESXi host now runs **per-Opstella-env stacks** (DEV live, SIT installing, CUSTOMER provisioning), not a single shared instance
- Orion GKE (secondary, cloud-hosted)

**Customer-site Opstella installs do NOT use this architecture.** They use ephemeral infrastructure at `$HOME/opstella-installation/` per customer and are documented in `08-install-index.md` (read by the `opstella-install` agent).

## Summary
Deltron = one Dell PowerEdge R6615 ESXi host running per-env RKE2 stacks. Use this file for "our" setup — cluster contexts, K8s versions, node sizes, ingress/storage classes, TLS secrets, where things run — rather than Opstella product concepts in general.

> **Live vs baseline vs future — read before quoting specifics:**
> - **Live/current** = the per-env installs. Contexts `deltron-{env}-{role}`, domains `{env}.deltron.opstella.in.th`, realm `{env}-opstella`, internal DNS `cluster.local`. Authoritative per-cluster detail (API endpoints, kubeconfigs, namespaces, defects) → the DEV/SIT tracker CLAUDE.md files.
> - **Pre-rebuild baseline** = the detailed 4-cluster hardware tables below (contexts `rke2cis-deltron-*`, domain `*.deltron.opstella.in.th`, DNS `.svc.kluster.local`). Prior art — the DEV env was rebuilt from it. Do **not** quote its contexts/DNS as live.
> - **Future** = Design D greenfield (8 clusters, `.opstella.deltron.local`) → `Architecture/DESIGN.md` + `SPEC.md`. Not built.

---

## Environment Topology (current)

One R6615 host, per-Opstella-env RKE2 stacks. **Opstella env** {DEV,SIT,CUSTOMER} = full platform instance ≠ **app-tier** {dev,sit,pre-prod,prod} (see Environment Mapping below).

| Env | State | Clusters (context `deltron-{env}-{role}`) | API endpoint | Base domain / realm |
|---|---|---|---|---|
| **DEV** | Live (install done 2026-06-18) | `deltron-dev-devsecops` | `10.21.110.2:6443` | `dev.deltron.opstella.in.th` / `dev-opstella` |
| | | `deltron-dev-non-prod-wk` | `10.21.111.10:6443` | |
| | | `deltron-dev-prod-wk` | `10.21.112.10:6443` | |
| **SIT** | Installing (from 2026-06-30) | `deltron-sit-devsecops` (mgmt) | `10.21.120.2:6443` | `sit.deltron.opstella.in.th` / `sit-opstella` |
| | | `deltron-sit-wk` (combined nonprod+prod) | `10.21.121.2:6443` | workloads: `*.app.sit.deltron.opstella.in.th` |
| **CUSTOMER** | Provisioning (Terraform on bastion) | — | — | → `Architecture/CUSTOMER-SESSION-CONTEXT.md` |
| **Shared** | — | `deltron-share-observability` | — | — |

- Internal DNS: `cluster.local`. Kubeconfigs: `source shell-values/kubernetes/<cluster>_cluster.vars.sh` (long-lived opstella-admin SA) or Rancher — exact paths in each tracker's **Cluster Map**.
- SIT diverges from DEV: **combined** workload cluster (nonprod+prod co-tenant, tier isolation by namespace+ResourceQuota+PSA), and a **two-zone** DNS split (`*.sit` → mgmt LB, `*.app.sit` → workload LB).
- ingress-nginx namespace on DEV = `ingress-controller` (not `ingress-nginx`), all clusters.
- Authoritative live detail per env → `opstella-installation/CLAUDE.md` (DEV) and `sit-opstella-installation/CLAUDE.md` (SIT), plus their companion docs/defect trails.

---

## Hardware

| Component | Spec |
|---|---|
| Host | Dell PowerEdge R6615 |
| CPU | 48 physical cores — AMD EPYC 9454P 48-Core |
| RAM | 255.58 GB |
| NVMe SSD | 319 GB |
| Storage | 8.73 TB HDD |

**Usable ceiling (with overcommit):** ~96 vCPU / ~230 GB RAM / ~9 TB

---

## Cluster Topology — Pre-Rebuild Baseline (prior art)

> **The tables in this section describe the pre-rebuild 4-cluster host** (single Opstella instance, contexts `rke2cis-deltron-*`, DNS `.svc.kluster.local`). The DEV env was rebuilt from this layout — hardware sizing is still broadly representative, but **contexts, domains, and DNS are superseded** by the per-env facts in Environment Topology above. Do not quote `rke2cis-deltron-*` as a live context. Live per-env specs live in the DEV/SIT trackers.

### DevSecOps Cluster
Hosts all CI/CD, container registry, secret management, and Opstella platform tooling.

**Access & Platform:**
- kubectl context: `rke2cis-deltron-devsecops`
- Kubernetes version: v1.32.5+rke2r1
- IngressClass: `nginx`
- StorageClasses: `external-nfs` (default), `longhorn`, `longhorn-static`

**Hardware:**

| Role | vCPU | RAM | Storage |
|---|---|---|---|
| 1 Master | 2 | 4 GB | 30 GB |
| 3 Workers | 8 each | 24 GB each | 500 GB each |
| 1 NFS | 1 | 2 GB | 200 GB |
| 1 HAProxy | 1 | 2 GB | 10 GB |

### Non-prod Cluster
Workload target for Opstella-deployed applications in non-production environments (dev/sit/pre-prod).

**Access & Platform:**
- kubectl context: `rke2cis-deltron-nonprod`
- Kubernetes version: v1.32.5+rke2r1
- IngressClass: `nginx`
- StorageClasses: `external-nfs` (default)

**Hardware:**

| Role | vCPU | RAM | Storage |
|---|---|---|---|
| 1 Master | 2 | 4 GB | 30 GB |
| 3 Workers | 4 each | 8 GB each | 60 GB each |
| 1 NFS | 1 | 2 GB | 100 GB |
| 1 HAProxy | 1 | 2 GB | 10 GB |

### Prod Cluster
Workload target for Opstella-deployed applications in production environment.

**Access & Platform:**
- kubectl context: `rke2cis-deltron-prod`
- Kubernetes version: v1.32.5+rke2r1
- IngressClass: `nginx`
- StorageClasses: `external-nfs` (default)

**Hardware:**

| Role | vCPU | RAM | Storage |
|---|---|---|---|
| 1 Master | 2 | 4 GB | 30 GB |
| 3 Workers | 4 each | 8 GB each | 60 GB each |
| 1 NFS | 1 | 2 GB | 200 GB |
| 1 HAProxy | 1 | 2 GB | 10 GB |

### Observability Cluster
Hosts the shared observability stack (Grafana, Mimir, Loki, Tempo, Alloy agents).

**Access & Platform:**
- kubectl context: `rke2cis-deltron-observability`
- Kubernetes version: v1.32.5+rke2r1
- IngressClass: `nginx`
- StorageClasses: `external-nfs`, `longhorn` (default), `longhorn-static`

**Hardware:**

| Role | vCPU | RAM | Storage |
|---|---|---|---|
| 1 Master | 2 | 4 GB | 30 GB |
| 3 Workers | 4 each | 8 GB each | 60 GB each |
| 1 NFS | 1 | 2 GB | 1024 GB |
| 1 HAProxy | 1 | 2 GB | 10 GB |

### Bastion
Jump host for secure access to all clusters.

| vCPU | RAM | Storage |
|---|---|---|
| 2 | 4 GB | 50 GB |

### Orion GKE Cluster (Secondary — Cloud)
Live Opstella reference environment on GCP. Heavily multi-tenant (~140 namespaces, ~10 platform teams).
Inspected 2026-05-19 — nodes restored and healthy.

**Access & Platform:**
- kubectl context: `gke_orion` (full: `gke_opstella-dev_asia-southeast1-a_internal-orion-opstella-omni`)
- GCP project: `opstella-dev` | Zone: `asia-southeast1-a`
- Kubernetes version: v1.35.3-gke.1389000
- Container runtime: containerd 2.1.5 | OS: Container-Optimized OS (Google), kernel 6.12.68+
- IngressClass: `nginx`
- StorageClass in use: `standard-hdd` (GCP Persistent Disk, HDD tier, RWO only — used for all PVCs)
- Available StorageClasses: `standard-rwo` (default), `premium-rwo`, `standard` (RWX) — but `standard-hdd` is what Opstella Helm charts actually provision
- Wildcard domain: `*.orion.opstella.in.th`
- TLS secret: `wildcard-orion-opstella-in-th-tls`
- Single ingress LB IP: `35.185.177.127`

**Node Pool (spot-pool — all 6 nodes are GKE spot/preemptible):**

| Nodes | vCPU | RAM | Allocatable CPU | Allocatable RAM | Pool label |
|---|---|---|---|---|---|
| 6 spot nodes | 4 vCPU each | ~16 GB each | 3920m per node | ~13.3 GB per node | `cloud.google.com/gke-spot=true` |
| **Cluster total** | **24 vCPU** | **~96 GB** | **23520m** | **~80 GB** | |

> **Memory pressure observed (2026-05-19):** 2 of 6 nodes at 81% and 57% RAM. Spot nodes can be evicted by GKE at any time — no persistent node guarantee.

**Full Helm stack (by category):**

| Category | Key Releases | Namespaces |
|---|---|---|
| DevSecOps tools | ArgoCD v3.4.2 (**FAILED** Helm status), Harbor 2.13.1, Vault 1.20.1, SonarQube 2025.1.3, DefectDojo 2.47.1, ESO v0.10.5, Headlamp, Reloader | `devsecops-system` |
| Identity | Keycloak 24.0.5 (HA × 2 replicas + PostgreSQL) | `opstella-identity-system` |
| Observability | Grafana 11.2.0, Loki 3.5.3, Mimir 2.17.0, Tempo 2.8.2, Alloy agents | `observability-system`, `observability-agents`, `observability-agents-dev` |
| Opstella platform | opstella-core, opstella-ui, opstella-clear-session, 12× workers, ok8s-integration | `opstella-platform`, `opstella-platform-dev` |
| Source control | GitLab v17.11.6 (Helm chart, in-cluster) + PostgreSQL + Gitaly | `gitlab` |
| Supporting | Dapr 1.15.9, MinIO 50 Gi, Redis operator, cert-manager v1.19.0, kubernetes-replicator | `dapr-system`, `supporting-services` |
| CI runners | GitHub Actions Runner Scale Set controller + 2 runner sets (gcloud + gcloud-custom) | `arc-systems`, `arc-runners` |

**Top memory consumers (RSS, idle load):**

| Workload | Namespace | RAM RSS |
|---|---|---|
| SonarQube | devsecops-system | **1945 Mi** |
| GitLab Sidekiq | gitlab | 1312 Mi |
| Grafana Alloy metrics (×2) | observability-agents | ~1600 Mi |
| Keycloak HA (×2 pods) | opstella-identity-system | ~1010 Mi |
| Mimir ingesters (×2) | observability-system | ~890 Mi |
| Opstella core | opstella-platform-dev | 437 Mi |
| ArgoCD app controller | devsecops-system | 404 Mi |

**Standard app namespace ResourceQuota (Opstella-provisioned):**
```
limits.cpu:    5 (5000m)
limits.memory: 5000Mi
```
Actual usage is typically 0–200 Mi per namespace (most app namespaces idle).

**Storage PVC highlights:**
- MinIO: 50 Gi (`supporting-services`)
- Vault HA: 3 × 10 Gi = 30 Gi
- GitLab (Gitaly + Minio + PG): 28 Gi
- Per-tool PostgreSQL: 8–10 Gi each
- All PVCs use `standard-hdd` storage class

**Identity:** Keycloak runs in-cluster (HA × 2) at `opstella-identity-system`. This differs from early docs that said "no Keycloak pod in Orion." The external reference at `https://idp.orion.opstella.in.th/realms/internal-dev` is served from this in-cluster Keycloak.

**Note:** Orion has no NFS or HAProxy — GKE managed LoadBalancer + GCP Persistent Disks. No MetalLB. No Jenkins (uses GitHub Actions runners instead).

> **Orion as Deltron redesign reference:** see `orion/ANALYSIS.md` for cross-reference of Orion resource consumption against Deltron Design D sizing assumptions. Key finding: SonarQube alone consumes ~2 GB RSS — validates that DevSecOps 3×4/8 GB workers are marginal for sustained load.

---

## Cluster Roles

Per-env stacks share the same role pattern (`{env}-devsecops` = control plane; `{env}-*-wk` = workload; observability shared):

| Role | Type | Purpose |
|---|---|---|
| **DevSecOps** (per env) | RKE2 on-prem | Opstella platform core, ArgoCD, Harbor, Vault, Keycloak, GitLab, SonarQube — the "control plane" of the IDP |
| **Non-prod / Workload** | RKE2 on-prem | Workload cluster. DEV splits nonprod (dev/sit/pre-prod) + prod; SIT/CUSTOMER use one combined cluster for all tiers |
| **Prod** (DEV only) | RKE2 on-prem | DEV's dedicated `prod` app-tier workload cluster |
| **Observability** (`deltron-share-observability`) | RKE2 on-prem | Shared observability (Grafana, Mimir, Loki, Tempo) serving all envs; tenant-separated by `X-Scope-OrgID` |
| **Orion** | GKE cloud | Secondary Opstella instance on GCP, multi-tenant (~140 ns, ~10 teams); spot node pool |
| **Bastion** | VM only | Admin SSH jump host — not a cluster |

---

## Opstella Environment Mapping

Two key terms that must not be conflated:

- **Opstella env** — a complete Opstella platform instance. Three exist: `DEV`, `SIT`, `CUSTOMER`
- **App-deployment tier** — a downstream target inside an Opstella instance. Four exist: `dev`, `sit`, `pre-prod`, `prod`

Example: `sit-app-prod` = the SIT Opstella instance's `prod` app tier — not "SIT and prod combined".

Live per-env cluster→tier mapping:
- **DEV** (3 clusters): `deltron-dev-non-prod-wk` hosts non-prod app tiers (`dev`, `sit`, `pre-prod`); `deltron-dev-prod-wk` hosts `prod`.
- **SIT** (2 clusters): `deltron-sit-wk` is **combined** — runs ALL tiers (nonprod + prod), isolated by namespace + ResourceQuota + PSA + NetworkPolicy, not by separate clusters.

---

## Network Layout

| Component | Role |
|---|---|
| HAProxy (per cluster) | Load balancer for cluster API server and ingress traffic |
| NFS (per cluster) | Persistent volume storage for workloads in that cluster |
| Bastion | SSH jump host; only external access point to the internal network |

Public ingress domain (live): **per-env** `{env}.deltron.opstella.in.th` (SIT workloads also `*.app.sit.deltron.opstella.in.th`). The flat `*.deltron.opstella.in.th` was the pre-rebuild baseline.
Cluster-internal DNS suffix (live): `cluster.local` (baseline used `.svc.kluster.local`).

---

## TLS & Certificates

### Deltron (RKE2 clusters)

**Live (per-env):** each env has `wildcard-{env}.deltron.opstella.in.th-tls` for the base domain; SIT workloads add `wildcard-app-sit.deltron.opstella.in.th-tls` for `*.app.sit...`. cert-manager per env; exact ClusterIssuer/renewal per env → the DEV/SIT trackers (note: DEV trackers flag ClusterIssuer failures as escalate-only, environment-specific).

**Pre-rebuild baseline (prior art):**

| Item | Value |
|---|---|
| ClusterIssuer | `opsta-issuer` (cert-manager, all 4 clusters) |
| Primary Wildcard Secret | `wildcard-deltron-opstella-in-th-tls` |
| Wildcard Domain | `*.deltron.opstella.in.th` |
| Per-tier certs (workload clusters) | `wildcard-{dev,sit,uat}-deltron-opstella-in-th-tls` (nonprod); `wildcard-{prd,pre}-deltron-opstella-in-th-tls` (prod) |

### Orion (GKE)

| Item | Value |
|---|---|
| Wildcard Secret | `wildcard-orion-opstella-in-th-tls` |
| Wildcard Domain | `*.orion.opstella.in.th` |

---

## Key Namespaces per Cluster

> Namespace layout below is the baseline/representative pattern (old cluster names). The per-env clusters follow the same roles; live per-env namespace lists → the DEV/SIT trackers.

### DevSecOps Cluster
- System: `cert-manager`, `longhorn-system`, `cilium-secrets`, `kube-system`
- Opstella Core: `opstella-system`, `opstella-system-stg`, `opstella-v4-system`
- Identity: `opstella-identity-system` (Keycloak prod, release `keycloak-cloudpirates`), `opstella-identity-system-stg`
- DevOps Tools: `devsecops-system` (ArgoCD, SonarQube, DefectDojo, Vault, Headlamp agents), `registry-system` (Harbor)
- Observability: `observability-system`, `observability-agents`
- CI/CD: `opstella-shared-runner` (GitLab runners), `arc-systems`, `arc-runners` (GitHub Actions runners)
- Supporting: `dapr-system`, `cnpg-system`, `apps-supporting-services`, `log-agent`, `vault`, `ot-operators`

### Non-prod Cluster
- System: `cert-manager`, `cilium-secrets`, `kube-system`
- Opstella: `opstella-system`, `opstella-user`
- Observability: `observability-system`, `observability-agents`
- DevOps: `devsecops-system` (agent namespaces)
- Supporting: `datastore-system`, `log-agent`, `vault`
- Workload namespaces: per-app deployment namespaces (dell-*, dst-*, juk-*, etc.)

### Prod Cluster
- System: `cert-manager`, `cilium-secrets`, `kube-system`
- Opstella: `opstella-system`, `opstella-user`
- Observability: `observability-system`, `observability-agents`
- DevOps: `devsecops-system` (agent namespaces)
- Supporting: `log-agent`, `vault`
- Workload namespaces: per-app deployment namespaces (dell-*, dst-*, juk-*, etc., prod-suffixed)

### Observability Cluster
- System: `cert-manager`, `longhorn-system`, `cilium-secrets`, `kube-system`
- Monitoring: `observability-system` (Grafana, Mimir, Loki, Tempo), `observability-agents`
- Supporting: `seaweed-system` (SeaweedFS for log/metric storage), `apps-supporting-services`, `devsecops-system`
- Opstella: `opstella-system`

---

## External Integration Points

| Service | Address | Type | Details |
|---|---|---|---|
| GitLab (Omnibus) | `ubuntu@10.21.1.9` | External VM | Ruby `gitlab.rb`; TLS certs at `/etc/gitlab/ssl/`; NOT in-cluster |
| Keycloak (Deltron) | `https://idp.deltron.opstella.in.th` | In-cluster (DevSecOps) | Prod realm: `opstella-identity-system`, release `keycloak-cloudpirates` |
| Keycloak (Deltron Staging) | `https://idp-stg.deltron.opstella.in.th` | In-cluster (DevSecOps) | Staging realm: `opstella-identity-system-stg` |
| Keycloak (Orion, in-cluster) | `https://idp.orion.opstella.in.th/realms/internal-dev` | In-cluster (Orion) | Keycloak 24.0.5 HA×2 in `opstella-identity-system` on Orion GKE — NOT Deltron's Keycloak |

---

## Key Operational Notes

- **Single-master clusters** — this is demo/dev infrastructure; no HA control planes
- **Access path:** Bastion → HAProxy → cluster API / worker nodes
- **DevSecOps cluster workers carry the heaviest load** (8 vCPU / 24 GB RAM each vs 4 vCPU / 8 GB RAM on workload clusters) — reflect this when sizing new tools on the DevSecOps cluster
- **Observability NFS is oversized** (1 TB) relative to others — intentional for log/metric retention

---

## Maintenance

- **Re-sync this file** when an env's install/provisioning state changes, new per-env clusters come online, or K8s is upgraded. Live source of truth = the DEV/SIT tracker CLAUDE.md files (this file summarizes; trackers hold verified specs).
- **When SIT install completes / CUSTOMER provisioning finishes:** promote its row in Environment Topology and fill verified specs from its tracker's Cluster Map.
- **Pre-rebuild baseline tables:** historical prior art — do not extend; if fully decommissioned, condense to a one-line pointer.
- **Orion cluster:** re-inspected 2026-05-19. Next verify after GKE upgrade or major tool changes. See `08-orion-gke.md` for GKE-specific patterns.
- **Future rebuild (Design D):** `Architecture/DESIGN.md` + `SPEC.md` — target-state, not built.
- **Docs-sourced files (01–06):** re-fetch quarterly or after Opstella version upgrades
