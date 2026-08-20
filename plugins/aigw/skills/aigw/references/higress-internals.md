# Higress internals

Primary source for this file is `docs/HIGRESS-INTERNALS.md`, a living doc that was
verified against the live cluster on 2026-08-11 (its own §9 records exactly what was
checked and what was corrected). Read `SKILL.md`'s source-of-truth ladder first — several
facts below are corrections to an earlier, wrong understanding, and this file states the
corrected version, not the original.

## What Higress actually is

Not written from scratch — an assembly of three existing projects:

| Layer | Origin | What Higress does with it |
|---|---|---|
| Envoy | CNCF L7 proxy | Unmodified. The thing that moves bytes |
| Istio `pilot`/`discovery` | Istio's control plane | Reused as-is as the xDS config server |
| Higress controller (`higress-core`) | Alibaba's own code | Translates Ingress/Gateway-API/Higress CRDs into Istio's internal config model |

`docs/HIGRESS-INTERNALS.md:26-38`. No service mesh, no sidecars — Istio-without-the-mesh,
aimed at north-south (ingress) traffic. Apache-2.0 (`docs/HIGRESS-INTERNALS.md:38`).

## Control plane vs data plane

Two Deployments, `higress-controller` and `higress-gateway`, in `higress-system`
(`docs/HIGRESS-INTERNALS.md:105-114`).

- **`higress-gateway` = Envoy.** Every client request enters here. Separate Deployment
  because it's the only thing on the hot path — scales and fails independently.
- **`higress-controller` = 2 containers, verified live:**
  `higress-controller-866b675ff5-lms2k   higress-core,discovery   Running`
  (`docs/HIGRESS-INTERNALS.md:142-158`). `higress-core` watches K8s objects and translates
  them; `discovery` is Istio pilot, converting that into xDS resources and serving them
  over gRPC. Two containers, not one, because pilot is upstream Istio taken as-is —
  Higress didn't fork it, so it tracks Istio releases independently.

**If the controller dies, the gateway keeps serving on its last-known xDS state.** You
lose the ability to CHANGE routing, not to serve it — `docs/HIGRESS-INTERNALS.md:120-122`.

## xDS — why this isn't a reload-based gateway

xDS is the gRPC protocol Envoy uses to fetch config from a control plane instead of
reading a file. Sub-protocols named for their payload: LDS (listeners), RDS (routes),
CDS (clusters), EDS (endpoints), ECDS (extension config = Wasm plugins) —
`docs/HIGRESS-INTERNALS.md:62-68`.

Key property: **push-based and hot.** The control plane pushes a delta over a persistent
gRPC stream and Envoy reprograms itself in-process — no reload, no dropped connections, no
pod restart. This is the single biggest behavioral difference from ingress-nginx (which
rewrites `nginx.conf` and signals a reload). It's also what makes a runtime controller
(our own `control-plane`, see `references/components.md`) rewriting `WasmPlugin` config
every ~30 seconds a viable design rather than a disruptive one.

## Envoy's four core objects

`docs/HIGRESS-INTERNALS.md:53-60`, mapped to the nginx concept a DevOps reader likely
already has:

| Envoy term | Means | nginx analogy |
|---|---|---|
| Listener | a socket bound to a port + the filter chain to run | `server { listen 443; }` |
| Route | (host, path, header) → a cluster | `location` block |
| Cluster | a named group of backend endpoints + LB policy | `upstream` block |
| Endpoint | one actual IP:port inside a cluster | a `server` line in `upstream` |

## The listener reality check — ~20 CR listeners, 2 real ones

**Corrected fact — was wrong in an earlier draft of the repo's own docs, see
`docs/HIGRESS-INTERNALS.md:168-179` and `:692`.** Our `Gateway` CR
(`charts/opsta-ai-gateway/templates/gateway.yaml`) declares one HTTP + one HTTPS listener
for each of 8 hostnames — `api`, `grafana`, `console`, `auth`, `mcp`, `backup`,
`seaweedfs` (labeled `swf` in the template), `argocd`, `vault`, `higress-console` — 9
hostnames × 2 = 18-20 named listener entries depending on which optional ones are
enabled (`gateway.yaml:17-222`, confirmed by grepping `hostname:` occurrences in that
file).

Envoy itself has exactly **2 physical listeners**: `0.0.0.0_80` and `0.0.0.0_443`
(`docs/HIGRESS-INTERNALS.md:259-291`). A listener is a socket on a port, and there are
only two ports. The named Gateway-API listeners collapse into **filter chains selected
by TLS SNI** inside those two:

```
listener 0.0.0.0_443
├── filter_chain  server_names=[api-<env>.<domain>]        → AI plugin stack attaches HERE
├── filter_chain  server_names=[console-<env>.<domain>]    → no AI filters
├── filter_chain  server_names=[grafana-<env>.<domain>]
├── … one per hostname …
└── (one chain with NO server_names, and — critically — NO default_filter_chain)
```

`docs/HIGRESS-INTERNALS.md:262-291`. `:80` listeners carry exactly one rule per chain:
`RequestRedirect 301 → https`. HTTP never reaches application logic.

**Consequence worth internalizing:** an unknown hostname matches no chain →
**the TCP connection is dropped**, not answered with a 404. Symptom is `curl: (35) recv
failure` or a connection reset, not an HTTP status. A 404 means TLS succeeded and route
matching failed — different layer, different investigation
(`docs/HIGRESS-INTERNALS.md:266-267`, and see Reveal #1 in `SKILL.md`).

## The filter chain — where policy actually lives

Within a chain, a request walks an ordered list of HTTP filters before the terminal
router filter. Any filter may read, mutate, buffer, or short-circuit (return a response
immediately, stopping the chain — the router filter and everything after never runs)
(`docs/HIGRESS-INTERNALS.md:70-73`).

**Live-verified ordering** (`docs/HIGRESS-INTERNALS.md:293-334`, captured from
`config_dump` on 2026-08-11) — note that native Envoy filters bracket and separate the
Wasm ones, which is what "phase" concretely means:

```
envoy.filters.http.custom_response / compressor / cors     ← native, always first
─── AUTHN-phase Wasm, descending priority ───
  (see SKILL.md's plugin table for the full live-verified order)
─── envoy.filters.http.rbac / local_ratelimit ───           ← THIS is the phase boundary
─── default-phase Wasm ───
  (ai-statistics, ai-quota, ai-token-ratelimit, …)
─── envoy.filters.http.fault / router ───                   ← router proxies to the cluster
```

`AUTHN` literally, physically means "runs before the native `rbac`/`local_ratelimit`
filters" — not an abstract label. A `WasmPlugin`'s default-phase status reads as
`UNSPECIFIED_PHASE` in `kubectl -o yaml`, never `DEFAULT`
(`docs/HIGRESS-INTERNALS.md:338-340`).

**`*.internal` filters are Higress's own shipped defaults**, distinct from ours — e.g.
`key-auth`(1000, ours-configured) vs `key-auth.internal`(310, Higress's own unconfigured
default). Both exist in the same live chain simultaneously
(`docs/HIGRESS-INTERNALS.md:303,318,341-344`).

## proxy-wasm and the Wasm plugin delivery model

Envoy can load a WebAssembly module as an HTTP filter. **proxy-wasm** is the ABI between
Envoy and the module: callbacks Envoy invokes (`onHttpRequestHeaders`,
`onHttpRequestBody`, `onHttpResponseHeaders`, `onHttpResponseBody`,
`onHttpStreamingResponseBody`), and host functions the module calls back into (get/set
header, get/set body, `sendLocalResponse`, `httpCall`/cluster dispatch) —
`docs/HIGRESS-INTERNALS.md:75-79`. This repo's own plugins are Go compiled to
`GOOS=wasip1` (`plugins/`).

Why Wasm over native filters or Lua: sandboxed (a plugin fault can't take down Envoy —
though see the caveat below), hot-loadable via ECDS (no Envoy rebuild or restart), and
distributable as an **OCI artifact** — pushed to a registry with `oras`, referenced as
`oci://registry/plugin:tag`, and **pulled by Envoy itself at runtime**, not by the
kubelet. This is why a `WasmPlugin`'s `imagePullSecret` is a separate single-string field
from a pod's normal `imagePullSecrets` — pod-level pull creds don't cover a fetch Envoy
performs itself (`docs/HIGRESS-INTERNALS.md:220-222`).

**The sandbox is not absolute.** A masking rule with `type=replace` + `restore=false`
runs an unguarded fancy-regex path that can panic and abort the whole Wasm VM (a 503) if
a pattern hits a backtracking limit — see the `%{HOSTNAME}` grok note in
`charts/opsta-ai-gateway/values.yaml` near line 348.

## The WasmPlugin CRD fields that matter

`docs/HIGRESS-INTERNALS.md:211-222`:

- **`url`** — `oci://…`, fetched by the gateway itself (see above)
- **`phase`** — coarse chain position; `AUTHN` runs before the default phase. Semantic,
  not numeric
- **`priority`** — fine ordering within a phase. **Higher runs first** — counterintuitive,
  it's not a "nice" value
- **`matchRules`** — selects traffic by `domain`, `ingress`, `service`, `routeType`.
  **There is no consumer selector**, and `route` is not a valid matchRule field — this
  single limitation is Reveal #4 in `SKILL.md` and shapes most of the custom plugin
  design in this repo

## McpBridge — why a plugin can't just use Service DNS

The chain of reasoning, in order (`docs/HIGRESS-INTERNALS.md` §5d, `:557-597`):

```
plugin needs Redis (or Qdrant, or Ollama, or our own control-plane)
  → the proxy-wasm sandbox has NO syscalls — no socket(), no connect(), no resolver
    → the plugin must ask ENVOY to make the call on its behalf
      → Envoy addresses backends by CLUSTER NAME, not by DNS name
        → clusters only exist for services Higress has REGISTERED
          → an McpBridge `dns` registry entry does that registration
```

Live-verified: `charts/opsta-ai-gateway/templates/redis.yaml:18-52` registers all four
backends this platform's plugins need — `redis` (`:19`), `control-plane` (`:38`),
`qdrant` (`:45`), `ollama` (`:49`) — each yielding an Envoy cluster like
`outbound|6379||redis.dns`, which `ai-token-ratelimit`/`ai-quota` address as
`service_name: redis.dns` (`wasmplugins.yaml:502,692`).

**Result: two addressing schemes for the same backend**, and this is not
redundancy — two callers with genuinely different capabilities:

| Caller | Address | Why |
|---|---|---|
| A Wasm plugin (`ai-quota`, `ai-cache`, `semantic-guard`, …) | `<name>.dns` (McpBridge cluster) | sandbox has no socket |
| A normal pod (our `control-plane`, console's `oauth2-proxy`) | plain Kubernetes Service DNS | normal syscalls, normal DNS |

The `control-plane` McpBridge entry is the sharpest illustration of the rule: **a plugin
calling our OWN API still can't use Service DNS**, because the sandbox constraint applies
regardless of whose service it is. Generalize it: any time a Wasm plugin must reach
something, that something needs an `McpBridge` entry.

## Other Higress CRDs, briefly

- **`Http2Rpc`** — HTTP↔Dubbo/gRPC transcoding. Not used in this repo (an
  Alibaba-ecosystem feature with no matching need here).
- `enableIstioAPI: true` (in `platform-values/higress.yaml`) gives ONLY the `EnvoyFilter`
  CRD — the raw "patch generated Envoy config directly" escape hatch — NOT
  `VirtualService`/`DestinationRule`, which an earlier draft of this repo's own docs
  incorrectly claimed (`docs/HIGRESS-INTERNALS.md:168-179`). Treat `EnvoyFilter` as a last
  resort — it matches on generated config *structure*, so it breaks silently across
  version bumps.

## Why Higress specifically (the rejected alternatives)

`docs/HIGRESS-INTERNALS.md:224-236`:

| Option | Why not |
|---|---|
| ingress-nginx | Config-file + reload model, no Wasm, no AI/LLM awareness |
| Istio ingress gateway | Same Envoy, but drags in the whole mesh; no AI plugin catalog |
| Kong / APISIX | Kong's LLM features are enterprise-gated; APISIX's plugin story is Lua-first |
| Envoy Gateway | Clean Gateway-API implementation, but no AI plugin catalog to build on |
| LiteLLM | Genuinely AI-native, but enterprise-licensed governance — forbidden by this repo's license rules |

Higress wins on the intersection: Apache-2.0, Envoy-grade data plane, Gateway-API
native, AND a prebuilt AI plugin catalog. The catalog is the deciding factor.

## Reading upstream source when this repo has none

Everything above is repo- and cluster-verified. When a question goes past what this repo
contains — the internals of a built-in plugin, the proxy-wasm host functions, the
controller's CRD→Istio translation — the source is upstream
`github.com/higress-group/higress`, plus the two SDKs our plugins genuinely import:

```
github.com/higress-group/proxy-wasm-go-sdk   the proxy-wasm ABI binding
github.com/higress-group/wasm-go             the `wrapper` layer (ParseConfig,
                                             ProcessRequestHeaders, NewClusterClient)
```

Both appear as real `require` entries in every `plugins/*/go.mod`, so their pinned
pseudo-versions are checkable in-repo — that is the honest way to answer "which SDK
revision are we on".

Upstream is **ladder rung 5**, below everything in this file. Rules that matter, in full
under "Upstream Higress source" in `SKILL.md`:

- Pin to our tested matrix — chart `2.2.3`, AI plugin OCI tag `2.0.0` (`version.yaml`).
  Never `main`.
- Never present an upstream default as our behavior. We override defaults throughout
  `charts/opsta-ai-gateway/values.yaml`, and `ai-cache` is a patched fork
  (`plugins/ai-cache/UPSTREAM.md`).
- Label it: "upstream source, not verified on our cluster."
- The repo's own fork attribution says `github.com/alibaba/higress`
  (`plugins/ai-cache/go.mod:1`) — quote that when citing a fork base rather than
  substituting the `higress-group` coordinate.
