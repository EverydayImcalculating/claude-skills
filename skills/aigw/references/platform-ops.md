# Platform operations — deploy, secrets, state, identity, observability, backup, release

This is the "how does it run as a system" reference — the `ops` mode in `SKILL.md`. Every
section ends by naming the command a user could run to confirm the claim; this skill
never runs it (see `SKILL.md` — "What this skill will never do").

## GitOps bootstrap and sync waves

`task up` runs an imperative bootstrap first, because ArgoCD can't install the cluster it
will later live in — then everything after is pull-based:

```
k3d cluster → Gateway API CRDs → redis-operator (for ArgoCD's own Redis CR)
→ vault-operator → OpenBao, KV seeded from secrets.yaml
→ ArgoCD → applies bootstrap/root-app.yaml (ONE Application, hand-applied)
→ root-app.yaml renders platform/ — everything else is ArgoCD from here
```

`docs/ARCHITECTURE.md:12-22` (the GitOps M2 cutover narrative). `platform/values.yaml` is
a Helm chart whose only output is more ArgoCD `Application` objects, one per `helmApps`
entry — each carries a `wave:` that ArgoCD uses as its
`argocd.argoproj.io/sync-wave` ordering annotation (apply wave N, wait Healthy, then N+1).

**Live wave assignments**, cited directly from `platform/values.yaml` (bare `values.yaml:N`
in the table below means THAT file — the chart's own `charts/opsta-ai-gateway/values.yaml`
is always written out in full):

| Wave | Line ref | Contains |
|---|---|---|
| -1 | `values.yaml:19-22` | `gateway-api-crds` |
| 0 | `values.yaml:25-47,88-92` | `external-secrets`, `cloudnative-pg`, `higress`, `cert-manager` |
| 1 | `values.yaml:99-103` | `barman-plugin` (needs cert-manager's webhook from wave 0) |
| 3 | `values.yaml:228-233` | `opsta-pe-agent` (condition: `peRuntime.enabled`) |
| 10 | `values.yaml:61-82,108-111` | `qdrant`, `ollama`, `seaweedfs` (all condition-gated), `opsta-ai-gateway-infra` |
| 15 | `values.yaml:119-123` | `keycloak` (+ its own CNPG DB) |
| 20 | `values.yaml:140-143` | `opsta-ai-gateway-app` (control-plane, console, oauth2-proxy, WasmPlugins, routes) |
| 25 | `values.yaml:160-206` | `loki`, `tempo`, `grafana`, `alloy`, `mimir`, `velero` |
| 26 | `values.yaml:210-214` | `velero-ui` (needs the server from wave 25) |
| 30 | `values.yaml:129-133` | `vault-oidc-config` (condition: `vaultUi.enabled`) |

**Wave 30 is a deadlock fix, not an arbitrary late number.** OpenBao's `auth/oidc/config`
write eagerly fetches Keycloak's discovery document at write time, but bank-vaults'
`externalConfig` converges sequentially, abort-on-first-error, during the operator's own
bootstrap — long before Keycloak (wave 15) exists. Doing that write inline would hang the
whole bootstrap. `docs/ARCHITECTURE.md:297-307` covers the full mechanism, including that
`vault-oidc-config` runs as an ArgoCD `Sync` hook Job (idempotent `bao write`, so it
re-applies safely on every sync).

To confirm live wave state yourself: `kubectl -n argocd get application -o
custom-columns='NAME:.metadata.name,SYNC-WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave,HEALTH:.status.health.status'`.

## Where git lives

Dev pulls from an **in-cluster git mirror** (`git://gitserver.gitops.svc:9418/repo.git`,
branch `dev-mirror`) — `task gitops:push` archives the local `HEAD`, renders it, and
bundles it in. Fully offline, no GitHub deploy key needed. Prod/customer uses a real
online repo instead. `docs/ARCHITECTURE.md:38-50` — `repoURL` in `config.yaml` chooses
*where*, the render mechanism (`scripts/render-platform-values.sh`) is identical either
way.

## The secrets chain

```
secrets.yaml (git-ignored)
  → seeded declaratively into OpenBao KV at bootstrap (bank-vaults startupSecrets)
  → an ESO ClusterSecretStore named "vault" (k8s-auth)
  → one ExternalSecret per row, producing a k8s Secret with the SAME NAME the
    chart already referenced before this refactor
  → consumed by Deployments/CRs/WasmPlugins exactly as before
```

**The refactor changed only the producer of each Secret, never the consumer** —
`docs/specs/2026-06-27-argocd-gitops-refactor-plan.md:163-176` has the full mapping
table. Confirmed rows: `secret/gateway/redis` → `redis-secret`
(`plan.md:168`, consumed by the Redis CR + `ai-quota`/`ai-cache`);
`secret/gateway/keycloak` → `keycloak-admin` (`plan.md:170`); `secret/gateway/postgres`
→ `opsta-pg-app`/`opsta-pg-appuser` (`plan.md:175`).

**Bootstrap-tier secrets never flow through ESO** (chicken-and-egg — ESO and ArgoCD need
them just to start): ArgoCD's own admin password, the git deploy key, OpenBao's own
`adminPassword`.

To confirm: `kubectl -n opsta-ai-gateway get externalsecret` (shows sync status per row)
or `kubectl -n vault-secrets-operator get clustersecretstore vault -o yaml`.

## State stores

| Store | Runs as | Used by |
|---|---|---|
| Postgres `opsta-pg` | CNPG `Cluster`, `opsta-ai-gateway` ns | control-plane's runtime source of truth |
| Postgres `keycloak-pg` | CNPG `Cluster`, `opsta-keycloak` ns | Keycloak's own isolated DB |
| Redis | Opstree `Redis` CR (standalone) or `RedisReplication`+`RedisSentinel` (HA) | see the logical-DB table below |
| Qdrant | official chart, `qdrant.higress-system:6333` | vectors for `ai-cache`, `semantic-guard`, `auto-router` |
| Ollama | official chart, `ollama.higress-system:11434` | `bge-m3` embeddings for the same three plugins |
| SeaweedFS | official chart, all-in-one, `seaweedfs-all-in-one:8333` | S3 backup target |

**Redis logical DB assignment — this is a place where the docs are incomplete, worth
flagging rather than papering over.** `docs/ARCHITECTURE.md:542-549` lists only DBs 0-2
(0 = `ai-quota`+`ai-token-ratelimit`, 1 = console `oauth2-proxy` sessions, 2 = the
control-plane API rate limiter). Two more assignments exist directly in
`charts/opsta-ai-gateway/values.yaml` that ARCHITECTURE.md's table predates: **DB 3**
(`charts/opsta-ai-gateway/values.yaml:676`, `semantic.redisDatabase: 3`) for `ai-cache`'s exact-match store, and
**DB 4** (`charts/opsta-ai-gateway/values.yaml:773`, `mcp.redisDatabase: 4`) for the MCP gateway feature. If
asked "how many Redis DBs are in use", the accurate answer is 5 (0-4), citing
`values.yaml` directly, not the 3 in `ARCHITECTURE.md`'s prose table.

**Infra/app release split** (`docs/ARCHITECTURE.md:209-236`) — the product chart renders
as two releases off the SAME chart, selected by `deployMode`: `tier=infra` (the Redis CR
and both CNPG Clusters ONLY) and `tier=app` (everything else). Why: a routine app deploy
used to re-apply the stateful CRs, waking the operators to roll the Redis/Postgres pods —
a brief outage during which `ai-quota` failed CLOSED ("No quota left" for everyone). Now
`task deploy` syncs `--selector tier=app` only, so a routine deploy never touches the
stores.

To confirm: `kubectl -n higress-system get redis,statefulset -l app.kubernetes.io/managed-by=redis-operator`.

## Identity

Keycloak is the in-cluster identity **broker** — fronting local DB users and/or brokered
AD-LDAP/Google/Entra/OIDC/SAML, with per-scheme group mapping
(`docs/ARCHITECTURE.md:108-127`). Console (via its own `oauth2-proxy`) and Grafana
(native OIDC) both authenticate against the realm.

`oauth2-proxy` runs in **reverse-proxy mode**, not as a Higress plugin
(`docs/ARCHITECTURE.md:993-999`): Higress's own `oidc` plugin has no Google provider and
can't enforce a hosted domain or map groups; `ext-auth`/forward_auth returns a bare 401
instead of redirecting a browser to the IdP — neither fits interactive human login.

**The identity seam** (`docs/ARCHITECTURE.md:1000-1007`): `oauth2-proxy` injects
`x-dev-user`/`x-dev-group` onto the upstream — the SAME headers the pre-SSO dev path used.
So the `{user, group}` tuple that budgets and limits enforce on never changed; only its
*source* did (a dev header vs an SSO claim).

**Sessions** — when Redis is on, sessions live in Redis DB 1 with a 5-minute
`COOKIE_REFRESH`, so disabling a Keycloak user logs them out within that window. Without
Redis, sessions are encrypted cookies, not server-side revocable
(`docs/ARCHITECTURE.md:1019-1025`).

**Two auth worlds, never mixed:** humans use SSO for dashboards/console; machines use
`sk-…` API keys via `key-auth` on `/v1` (`docs/ARCHITECTURE.md:1026-1028`).

To confirm: `kubectl -n opsta-keycloak get pods`, or check a specific user's group
mapping via the Keycloak Admin REST API (see `docs/HIGRESS-RESEARCH.md` §8-adjacent
commands in `docs/DEV-ENVIRONMENT.md`).

## Observability

```
higress-gateway :15020/stats/prometheus  ──scrape 15s──┐
gateway access logs                       ──tail──────►│ Grafana Alloy
                                                         ├─ remote_write ──► Mimir
                                                         └─ loki.write ────► Loki
                                                          (Tempo present, idle)
                                          all three ──datasource──► Grafana
```

`docs/ARCHITECTURE.md:878-940`. Mimir runs **standalone all-in-one**
(`-target=all`, single pod, filesystem) from this repo's own
`oci://ghcr.io/opsta/mimir-standalone` chart — `mimir-distributed` is explicitly NOT
usable for standalone: with filesystem storage its separate microservice pods get
separate RWO volumes, so the store-gateway never sees the ingester's shipped blocks and
historical queries return empty. This was proven at runtime, not caught by
render/kubeconform — worth remembering if someone proposes "just use the distributed
chart" for a single-node setup.

**Tenancy** = `X-Scope-OrgID`. `docs/ARCHITECTURE.md:914-925`: the control-plane's
`--mode=lgtm-auth` personality builds a token→tenant map from per-org
`lgtm-tenant-<org>` Secrets and FORCES `X-Scope-OrgID` from the presented credential — a
per-org token is pinned to its own tenant (cross-tenant read = 403). Isolation is
enforced on the READ side, where it matters, rather than issuing every Alloy stream a
per-tenant token.

**Grafana is platform-operator-only, deliberately.** `docs/ARCHITECTURE.md:929-939`: OSS
Grafana has no datasource-level permissions and grafana-operator v5 has no `GrafanaOrg`
CRD, so per-org Grafana isn't achievable declaratively. Org users read tenant-scoped
telemetry through the console instead; Grafana login is restricted to the platform admin
group.

To confirm: `kubectl -n opsta-observability get pods`, or query Mimir directly with a
known per-org token to verify tenant pinning.

## Backup and disaster recovery

Two layers, one in-cluster S3 store (SeaweedFS), gated on `backup.enabled`
(`docs/ARCHITECTURE.md:941-981`):

- **Layer 1 — Postgres PITR.** CNPG's Barman Cloud Plugin does continuous WAL archiving
  plus a 12h base backup per DB, to `s3://cnpg-backups/<db>/`. RPO ≈ WAL lag
  (seconds-minutes). The in-tree `barmanObjectStore` field is deprecated since CNPG
  1.26 — the plugin (sidecar, `isWALArchiver: true`) is the supported path.
- **Layer 2 — cluster objects.** Velero, daily 02:00, 7-day TTL, all namespaces. **CNPG
  PGDATA PVCs are excluded by design** — that's Layer 1's job; excluding them stops
  double-storage and keeps Velero restores fast.
- **Operating it** — `otwld/velero-ui`, authenticating natively against Keycloak (no
  `oauth2-proxy` in path, which would force a double login), with a Casbin policy
  gating *manage* to `opsta-admins`/`backup-admins`.

**Honest limit, stated in the doc itself:** single-site, in-cluster store — this protects
against data loss, not site loss. Losing SeaweedFS loses the backups too. Restore-based,
not warm-standby. A restore drill on 2026-06-28 measured Velero namespace round-trip
~15s and CNPG PITR into a fresh cluster ~70s.

To confirm: `kubectl -n cnpg-system get scheduledbackup`, `kubectl -n velero get
backup`.

## Release pipeline — build-once, promote by retag

```
dev push  → ci.yml (lint/license/scan/unit/chart) → build+push :dev and :sha-<short>
merge to main → promote-uat.yml: crane copy :dev → :uat        (NO rebuild)
tag vX.Y.Z    → release.yml: crane copy :uat → :vX.Y.Z, mirror AI plugins, package+push chart
manual trigger → production.yml (gated by GitHub "production" Environment, required reviewer)
              → task prod:deploy → helmfile -e prod --selector prodDeploy=true sync
              → control-plane starts → embedded migrations run forward-only
```

`docs/ARCHITECTURE.md:1887-1922`. The exact digest tested on `dev` is what reaches
production — `:uat` and `:vX.Y.Z` are extra tags on the SAME image, never a rebuild.
`prodDeploy=true` labels the product chart AND the LGTM releases, so an observability
change reaches prod through this same gate. The stable operators
(higress/cert-manager/redis-operator/cnpg/keycloak) are deliberately UNLABELLED — synced
only on a deliberate full sync, keeping routine releases fast and low-blast-radius.

**Prod is never `task reset`.** `docs/ARCHITECTURE.md:1972-1980`:
`task reset`/`helmfile destroy` are dev-only and wired into no prod path; destroying prod
is a manual `k3d cluster delete` an operator types by hand; migrations are forward-only
by rule.

To confirm: `gh run list --workflow=production.yml` (if the user has `gh` access and
wants to see recent prod deploys — still not something this skill runs itself).

## The config-change decision table

The question "how does my change reach the cluster" has exactly four answers, and
picking the wrong one is the single most common real mistake:

| Change kind | Path | What actually moves it |
|---|---|---|
| Tenant data (consumer, key, budget, allow-list, guardrail pattern, provider) | console/API → Postgres → reconcile (≤30s) → live `WasmPlugin.matchRules` | Nothing to deploy — it just happens |
| Product config (chart values, plugin baseline, a new resource) | git commit → render → in-cluster mirror or online repo → ArgoCD syncs by wave | `task gitops:push` then `task deploy:config` |
| Our own image code (a plugin or component) | build → push `:dev` → rollout restart | `task deploy` |
| A stateful store CR (Redis/Postgres) | deliberate, isolated from routine deploys | `task deploy:infra` |

If a question is "I changed X, why hasn't it taken effect", the FIRST thing to establish
is which of these four rows X belongs to — a tenant-data change that never got a git
commit is doing exactly what it should (nothing to deploy), while a chart-value change
that was never pushed through `gitops:push` will sit invisible in the working tree
forever.
