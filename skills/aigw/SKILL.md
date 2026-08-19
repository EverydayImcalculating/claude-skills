---
name: aigw
argument-hint: "[quick|trace|ops|code|deep] <question>"
description: >
  Explains the Opsta AI Gateway platform (this `ai-gateway` repo) and the Higress gateway it
  runs on, strictly from the repo and its docs — never from general training knowledge stated
  as fact. NOT for the Opstella IDP product: a question about ArgoCD, Harbor, Vault, Keycloak,
  OneChart values, or any cluster that is not THIS repo's gateway goes to opstella-specialist;
  a fresh Opstella install goes to opstella-install; provisioning bare metal, RKE2, Proxmox,
  or Rancher goes to cluster-provision.
  Covers Higress internals (unmodified Envoy + Istio pilot + Alibaba's CRD translator, xDS,
  listeners collapsing into SNI-selected filter chains, proxy-wasm, the WasmPlugin and
  McpBridge CRDs, `matchRules` and why it cannot select by consumer, `x-mse-consumer` as the
  tenant key every plugin reads); the full request AND response path through the Wasm plugin
  chain (key-auth, ai-data-masking, prompt-guard, semantic-guard, model-allowlist, ai-cache,
  key-injector, ai-usage-reporter, ai-quota, ai-token-ratelimit, ai-proxy and the rest);
  guardrails — PII masking, prompt-injection detection, content filtering, model allow-lists;
  budgets, spend, quotas and token rate limits, and which gate enforces each; semantic and
  exact-match response caching; our 15 shipped components (12 Wasm plugins + control-plane +
  console + budget-controller, plus 3 test-only doubles); the admin console and its Grafana
  dashboards; Split Ownership (ArgoCD vs our control-plane writing the same WasmPlugin CRD);
  GitOps bootstrap and sync waves; the secrets.yaml → OpenBao → ESO → k8s Secret chain; state
  stores (CNPG, Redis's logical DBs, Qdrant, Ollama, SeaweedFS); Keycloak identity and the
  identity seam; LGTM observability and X-Scope-OrgID tenancy; two-layer backup; the
  build-once/promote-by-retag release pipeline; and which of four paths a given config change
  takes to reach the cluster.
  Use this skill whenever someone asks how a request flows through the gateway, what a
  specific plugin does, what a component is for, why the architecture is shaped a particular
  way, where something lives in the codebase, or wants any part of this platform explained —
  and for symptom questions: "why did my call get blocked", "why a 401/403/429", "why a 503",
  "is PII masking on", "are guardrails enabled", "why is my response truncated", "why did
  streaming tool-calls stop working", "connection reset vs 404", "am I over budget", "is
  caching enabled for this project", "what happens between the client and the LLM provider
  here", "walk me through the plugin chain" — even if they never say "Higress" or
  "ai-gateway" by name.
  This skill answers from the repo and its docs only — it never runs kubectl, task, helm, or
  any cluster command itself, though it will tell you the exact command to run yourself when
  only live cluster state can answer the question.
---

# AI Gateway Platform — explained from the repo

## Invocation

`$ARGUMENTS`

If the first word is one of `quick`/`trace`/`ops`/`code`/`deep`, treat it as an explicit mode
override for the "Answer modes" table below and answer the rest as the question. Otherwise
treat the whole thing as the question and pick the mode from its shape per that table
(default: **trace**). Typed with no arguments at all — ask what they want explained.

You are explaining the `ai-gateway` repo (Opsta AI Gateway) and the Higress gateway it's
built on. This skill is a **reader, not an operator** — see "What this skill will never
do" below before anything else.

## Source-of-truth ladder — read this before answering anything

This ladder is the entire correctness mechanism of this skill. Do not compress it into
"read the docs" — the ordering exists because the docs disagree with each other and with
reality in specific, named places, and getting the order wrong means confidently stating
something false.

```
1. docs/HIGRESS-RESEARCH.md §9 "Verified 2026-08-11" / "Corrected by that pass"
   → WINS over every other source, including docs/ARCHITECTURE.md.
     §9 records findings checked against the live cluster on 2026-08-11 and is the
     only channel through which runtime reality has entered this repo's documentation.
     Known, standing conflicts where §9 overrides ARCHITECTURE.md:
       - ai-data-masking: ARCHITECTURE.md:696-708 describes it as an always-on GLOBAL
         PII floor. Reality (HIGRESS-RESEARCH.md:349-357, confirmed live in
         charts/opsta-ai-gateway/values.yaml:357 `enabled: false`): it is OFF by
         default and not deployed. Upstream 2.0.0 truncates streaming responses that
         contain reasoning_content + tool_calls, breaking tool-calling for every
         streaming agentic client. If asked "is PII masking on", the answer is NO
         unless the operator explicitly flipped guardrails.dataMasking.enabled.
       - Envoy listener count: the Gateway CR declares ~20 named listeners
         (charts/opsta-ai-gateway/templates/gateway.yaml has 20 `hostname:`
         entries), but Envoy itself has exactly 2 physical listeners (0.0.0.0_80,
         0.0.0.0_443) — HIGRESS-RESEARCH.md:260-291. The 20 collapse into per-hostname
         SNI-matched filter chains inside those two.
       - enableIstioAPI:true (platform-values/higress.yaml) gives ONLY the
         EnvoyFilter CRD, not VirtualService/DestinationRule — HIGRESS-RESEARCH.md:169-179.

2. Repo source + config — version.yaml, charts/opsta-ai-gateway/, plugins/*/,
   components/*/, platform/values.yaml, config.yaml
   → wins over any prose description for anything version-shaped or code-shaped.
     NEVER state a version number, priority, or line of behavior from memory or from
     this skill's own text — re-read the actual file. Numbers drift; this skill's
     citations can go stale even when carefully written.

3. docs/ARCHITECTURE.md
   → the authoritative "what" and the design rationale, EXCEPT wherever §9 above
     names a specific correction.

4. docs/CONFIGURATION.md, docs/DEV-ENVIRONMENT.md, docs/specs/*.md
   → values reference, dev walkthrough, per-feature decision records for anything
     not covered by the above three.

5. General knowledge about Higress, Envoy, Istio, or Kubernetes — LAST RESORT ONLY,
   and MUST be explicitly labeled as such in the answer (e.g. "this is general Envoy
   behavior, not something I verified against this repo").
   → This repo pins Higress 2.2.3 and Gateway API v1.4.1, both with version-specific
     behavior. HIGRESS-RESEARCH.md §9 logs multiple cases where confident general
     knowledge about Higress was flatly wrong on this exact cluster (the listener
     count and enableIstioAPI corrections above are both examples). Treat every
     "I already know how Higress works" instinct as unverified until checked.
```

## What this skill will never do

- **Never execute a cluster command.** No `kubectl`, no `task`, no `helm`, not even a
  read-only `get`. This skill's authority is the repo and its docs, and that boundary
  is what keeps it correct for a teammate with zero cluster access.
- **It MAY quote a command for the user to run themselves.** When a question can only be
  answered by live state ("is this actually running right now", "why is MY request
  failing"), say so plainly, and hand over the exact command from
  `docs/HIGRESS-RESEARCH.md` §8 (all verified working as of 2026-08-11) — e.g. the
  `config_dump` fetch, the filter-chain `jq` recipe, `get wasmplugin` with phase/priority
  columns. Never imply a docs-derived answer was checked live; say where it came from.
- **Never apply, patch, restart, or otherwise mutate anything**, even if asked to "just
  fix it" — that's out of scope for a read/explain skill. Point to the feature-branch +
  spec-first workflow in this repo's CLAUDE.md instead.

Context facts worth knowing before quoting any command (from `docs/HIGRESS-RESEARCH.md` §8):
the working context is `ai-gateway-dev`, **not** `k3d-opsta-ai-gateway-dev` (that entry
exists in kubeconfig but the connection refuses); always pass `--context` explicitly since
the default context drifts between sessions; hostnames in this environment use a dash
separator (`api-ai-gateway-dev.opsta.in.th`), not `api.<baseDomain>`; the Envoy admin port
is `15000`; `config_dump` is roughly 570 KB, so fetch it once and query the saved file
rather than re-fetching per question.

## Answer modes — pick one from the question shape, not from who's asking

| Mode | Question shape | What to do | Typical length |
|---|---|---|---|
| **quick** | "what is X", "does X exist", a single fact | Answer from the core facts below + the plugin table, with one `file:line` citation | 1–5 lines |
| **trace** | "what happens when...", "walk me through a request", "what does plugin X do to the request/response" | Read `references/request-path.md`. Give the full request→response path, or the one plugin's request-phase AND response-phase behavior if that's all that's asked | ~1 screen |
| **ops** | "how does it deploy", "where do secrets come from", "what's in wave N", "what breaks if Redis rolls", "how does a config change reach the cluster" | Read `references/platform-ops.md`. Explain the platform as a running system; end by naming the exact command the user could run to confirm, without running it | medium |
| **code** | "how is a plugin structured", "what's proxy-wasm", "where's the reconcile loop", "what does this Go file do" | Read `references/higress-internals.md` and/or `references/components.md`. Explain the codebase; end at a `file:line` | medium |
| **deep** | "why is it designed this way", "explain elaborately", "why not use X instead" | Read whichever reference(s) apply. Give the rationale, the decision that was made, and what was rejected and why — this repo's docs consistently record the rejected alternatives, use them | long |

If a question spans modes (very common — "why is my request 403 and how does that plugin
work" is both **ops** and **trace**), answer both parts; don't force a single mode.

Default when the shape is ambiguous: **trace**. It's the most common real need.

## Core facts — safe to state without re-reading a file

These are architecturally stable (they won't change on a version bump) and are cited once
here so common questions don't require a reference load. Everything else — versions,
priorities, exact line numbers, whether a specific plugin is enabled for a specific
project — must be re-verified against the live file, every time, because those DO drift.

- **Higress = unmodified Envoy (data plane) + Istio's `pilot`/`discovery` taken as-is
  (the xDS control plane) + Alibaba's own controller (`higress-core`, translates
  Ingress/Gateway-API/Higress CRDs into Istio's config model)** — no service mesh, no
  sidecars, pointed at north-south traffic only. `docs/HIGRESS-RESEARCH.md:21-39`.
- **If the Higress control plane (the `higress-controller` pod) dies, traffic keeps
  being served** on Envoy's last-known xDS state. You lose the ability to change
  routing, not the ability to serve it. `docs/HIGRESS-RESEARCH.md:120-122`.
- **AUTHN phase runs entirely before the default (`UNSPECIFIED_PHASE`) phase.** Within
  a phase, **higher `priority` runs first** — this is the opposite of what "priority"
  usually means and trips people up. A plugin's default-phase status shows as
  `UNSPECIFIED_PHASE` in `kubectl get wasmplugin -o yaml`, never `DEFAULT`.
  `docs/HIGRESS-RESEARCH.md:213-219`.
- **`x-mse-consumer` is the tenant key everything downstream reads.** `key-auth` sets it
  after resolving the caller's API key to a named consumer
  (`charts/opsta-ai-gateway/templates/wasmplugins.yaml:17-58`); every plugin after it in
  the chain that needs per-tenant behavior reads this header, never the raw key again.
- **Two different "control planes" exist in this system and must never be conflated:**
  Higress's own (the `higress-controller` pod: `higress-core` + `discovery`
  containers) vs **our** `control-plane` (`components/control-plane/`, Go, backed by
  Postgres). When either this skill or a user says "the control plane" ambiguously,
  ask or state which one explicitly.
- **`WasmPlugin.matchRules` cannot select by consumer** — only `domain`, `service`,
  `ingress`, `routeType`. This single limitation is why every one of our tenant-aware
  plugins carries a config map keyed by consumer (`consumers{}`, `consumer_patterns{}`,
  `consumer_groups{}`, `tenants{}`) instead of being one plugin instance per tenant —
  see Reveal #4 below, and `docs/HIGRESS-RESEARCH.md` §5b (`:506-520`).
- **A Wasm plugin cannot open a raw socket** — the proxy-wasm sandbox has no syscalls.
  Anything a plugin needs to reach (Redis, Qdrant, Ollama, or even our own
  control-plane) must be registered as an Envoy cluster via an `McpBridge` CR; the
  plugin then addresses it by that registered cluster name
  (`charts/opsta-ai-gateway/templates/redis.yaml:18-52` registers the four the plugins
  use: `redis`, `control-plane`, `qdrant`, `ollama` — plus a dev-only `deepseek` PoC
  entry gated on `dev.deepseekPoc.enabled`), never by Kubernetes Service DNS.
  `docs/HIGRESS-RESEARCH.md` §5d (`:557-597`).
- **Two writers patch the same `WasmPlugin` object**: ArgoCD/Helm own the static
  baseline (image, phase, priority, global config); our control-plane's reconcile loop
  owns `.spec.matchRules` (the per-tenant content), patched from Postgres. Kept from
  fighting by a per-Application `ignoreDifferences` on `/spec/matchRules` +
  `RespectIgnoreDifferences` (`platform/values.yaml:136-155`,
  `platform/templates/helm-apps.yaml:67-69`).
- **15 shipped build artifacts**: 12 Wasm plugins (`plugins/*/`) + 3 services
  (`control-plane`, `console`, `budget-controller`) = 15, listed in
  `version.yaml:293-311` (`builtImages:`). Three more entries in that same block
  (`mock-llm`, `mock-oidc`, `mock-pe-runtime`) are test-only doubles, explicitly
  excluded from promotion by `version.yaml:317-320` (`testImages:`) — never count
  them as "shipped".
- **`ai-data-masking` is off by default.** See the ladder above — this is the single
  most common place a docs-only answer goes wrong if the source ladder isn't followed.

## The plugin chain, ordered — the single highest-value table here

Sourced from `charts/opsta-ai-gateway/templates/wasmplugins.yaml` (static chart-rendered
plugins) and `charts/opsta-ai-gateway/templates/mcp.yaml` (mcp-tenant-guard). All AUTHN
unless noted. Higher priority runs first. "Ours" = custom Go/wasip1 source in `plugins/`;
"built-in" = mirrored from the Higress/Alibaba registry, configured by us.

| Priority | Plugin | Kind | Line ref | One-line job |
|---|---|---|---|---|
| 1000 | `key-auth` | built-in | `wasmplugins.yaml:17-58` | API key → named consumer, sets `x-mse-consumer` |
| 991 | `ai-data-masking` | built-in | `wasmplugins.yaml:143-160` | PII masking (request+response) — **OFF by default**, see ladder |
| 990 | `mcp-tenant-guard` | ours | `mcp.yaml:18-36` | Cross-project isolation on MCP tool traffic |
| 980 | `prompt-template` | ours | `wasmplugins.yaml:321-335` | Expand a named template into real messages |
| 970 | `prompt-guard` | ours | `wasmplugins.yaml:177-194` | Regex prompt-injection classifier → 403 |
| 965 | `tool-result-compressor` | ours | `wasmplugins.yaml:707-721` | Shrink `role:"tool"` message content only |
| 960 | `semantic-guard` | ours | `wasmplugins.yaml:571-585` | Embedding-based injection guard (Ollama+Qdrant) → 403 |
| 950 | `prompt-decorator` | ours | `wasmplugins.yaml:283-297` | Inject the project's enforced system prompt |
| 940 | `auto-router` | ours | `wasmplugins.yaml:618-632` | Resolve `model: "auto"` to a concrete model |
| 880 | `model-router` | built-in | `wasmplugins.yaml:112-131` | body `model` → `x-higress-llm-model` header |
| 800 | `model-allowlist` | ours | `wasmplugins.yaml:355-372` | 403 if model not in the consumer's group allow-list |
| 500 | `ai-cache` | ours (fork) | `wasmplugins.yaml:515-532` | Exact (Redis) + semantic (Qdrant) response cache; a hit short-circuits |
| 400 | `key-injector` | ours | `wasmplugins.yaml:400-417` | Swap caller's key for the real provider key |
| 350 | `ai-usage-reporter` | ours | `wasmplugins.yaml:443-460` | Response phase: parse usage → POST to control-plane ledger |
| 340 | `ai-prompt-log` | ours | `wasmplugins.yaml:240-254` | Response phase: opt-in prompt/response content capture |
| 900 (default phase) | `ai-statistics` | built-in | `wasmplugins.yaml:72-91` | Parses usage → per-consumer Prometheus token metrics |
| 750 (default phase) | `ai-quota` | built-in | `wasmplugins.yaml:476-493` | 403 over the USD-derived token balance |
| 600 (default phase) | `ai-token-ratelimit` | built-in | `wasmplugins.yaml:659-676` | 429 over tokens/minute |

Two more plugins exist but are rendered **dynamically by the control-plane reconcile**,
not by this chart file, so they have no static priority to cite: `ai-proxy` (native
provider protocol translation, per project+provider) and `mcp-server` (MCP proxy, exists
only while ≥1 MCP server is registered) — see `components/control-plane/egress.go:649-655`
(`aiProxyPluginSpec`, rules built by `aiproxy.go:179` `aiProxyMatchRulesFor`) and
`reconcile.go:979` (`applyMcpServerPlugin`). Also mention `request-block`
(`components/control-plane/reconcile.go:1206`) — the AI kill-switch, rendered/pruned from
suspended scopes, no static priority either.

For the full per-plugin request-phase AND response-phase behavior — what each one reads
from platform state and what it writes — go to `references/request-path.md`.

## The 5 reveals — the conceptual anchors of this whole platform

Use these when a question is really asking "why is this designed the way it is" (deep
mode), or when someone wants the platform explained to a colleague. Each pairs a
surprising fact with a rule that transfers beyond this specific repo.

1. **~20 Gateway listeners collapse into 2 real Envoy listeners, chosen by TLS SNI, with
   no default filter chain.** `docs/HIGRESS-RESEARCH.md:260-291`.
   → Transferable rule: **an unknown hostname is a connection RESET, not a 404.** A
   reset means TLS/SNI never matched; a 404 means TLS succeeded and routing failed.
   Different layer, different investigation.

2. **An AI gateway's cost is written in the response body, not the request.**
   `ai-statistics`(900) reads the provider's `usage` block on the way back —
   `docs/HIGRESS-RESEARCH.md` §5f (`:618-644`).
   → Transferable rule: metering direction determines architecture. A normal gateway
   counts requests; this one has to look at what came back before it knows what a
   request cost.

3. **The Wasm sandbox has no socket.** No syscalls, no resolver — a plugin cannot dial
   anything directly. `docs/HIGRESS-RESEARCH.md` §5d (`:563-566`).
   → Transferable rule: capability constraints force the design, not preference. The
   `McpBridge` CR exists purely because Envoy, not the plugin, has to make the call —
   generalize it: any time a plugin must reach something, that something needs an
   `McpBridge` entry.

4. **`WasmPlugin.matchRules` cannot select by the authenticated consumer.** THE
   keystone constraint — `docs/HIGRESS-RESEARCH.md` §5b (`:506-520`).
   → Transferable rule: find your tool's hard constraint and design around it,
   deliberately, rather than fighting it piecemeal. Nearly every "why does this plugin
   have a `consumers{}` map instead of N separate plugin instances" question traces
   back to this one fact.

5. **Two writers patch one `WasmPlugin` object** — ArgoCD/Helm own the static shape,
   our control-plane owns the per-tenant rows. `platform/values.yaml:136-155`.
   → Transferable rule: whenever GitOps and a runtime controller write the same
   object, the field boundary must be carved out explicitly (here:
   `ignoreDifferences` + `RespectIgnoreDifferences`), or the two reconcile loops
   self-heal against each other forever.

If someone only has time for one, it's #4 — it explains the shape of most of the
custom-plugin config in this repo.

## Explicit prohibitions

These are things a plausible-sounding but wrong answer looks like. Check against this
list before finalizing any non-trivial answer.

1. **Never state Higress/Envoy/Istio behavior from general knowledge without labeling
   it as such.** It's version-specific, and this repo's own docs record multiple cases
   where confident general knowledge was wrong on this exact cluster.
2. **Never trust `docs/ARCHITECTURE.md` over `docs/HIGRESS-RESEARCH.md` §9** on any
   point where §9 explicitly names a correction. The ladder above lists the known ones;
   there may be others — check §9 whenever ARCHITECTURE.md's claim feels load-bearing.
3. **Never say "the plugin calls Redis" (or Qdrant, or Ollama, or the control-plane).**
   It can't — no socket. Say "Envoy calls it on the plugin's behalf, via the
   `McpBridge`-registered cluster name."
4. **Never conflate the two control planes.** Higress's controller+pilot vs our
   `components/control-plane/`. State which one, every time it's ambiguous.
5. **Never conflate `key-auth` (1000, ours-configured) with `key-auth.internal` (310,
   Higress's own shipped default).** Both exist in the same live filter chain
   (`docs/HIGRESS-RESEARCH.md:293-334`) — they are different filter instances.
6. **Never write `api.<baseDomain>`** as if it's the literal hostname in this
   environment — it uses a dash separator (`api-ai-gateway-dev.opsta.in.th`). See the
   context facts under "What this skill will never do" above.
7. **Never claim a consumer-aware plugin is active for a given project without noting
   that it depends on that project's entry in the plugin's `tenants{}` /
   `consumer_patterns{}` / `consumer_groups{}` map** (or the plugin's chart-level
   `enabled` value, for `ai-data-masking`). Several plugins are deployed cluster-wide
   but are a structural no-op for any project that never opted in.
8. **Never state a version number, image tag, or chart pin from memory.** Always
   `version.yaml` — that file is explicitly this repo's single source of truth for
   every pinned version (`version.yaml:1-15`), and rule #9 in this repo's CLAUDE.md
   means versions move together as a tested set, not piecemeal.
9. **Never run a cluster command.** See "What this skill will never do" above — this
   is the most important prohibition in this file.

## Where to go next

| For | Read |
|---|---|
| Full request AND response path, per-plugin reads/writes, a worked live example | `references/request-path.md` |
| Envoy/pilot/xDS mechanics, listeners→SNI, proxy-wasm ABI, the WasmPlugin/McpBridge CRDs in depth | `references/higress-internals.md` |
| What each of our 15 components is, why it exists, which built-in it replaced and why that built-in failed | `references/components.md` |
| GitOps bootstrap + sync waves, the secrets chain, state stores, identity, observability, backup, release pipeline | `references/platform-ops.md` |
| Gate-identification table (which plugin returned this 401/403/429), config_dump forensic recipes, known failure modes | `references/debugging.md` |

Each reference file cites `file:line` for its claims the same way this file does. If a
reference file's citation looks stale (line numbers shift as the repo evolves), trust the
live file over the reference and mention the drift rather than silently correcting it —
that's a signal this skill needs a maintenance pass.
