# The 15 shipped components — what each is, why it exists

Authoritative list: `version.yaml:293-311` (`builtImages:`). 12 Wasm plugins in `plugins/`
+ 3 services in `components/` = 15. Three MORE entries in that same `builtImages:` block
(`mockLlm`, `mockOidc`, `mockPeRuntime`) are test-only doubles, explicitly excluded from
promotion by `version.yaml:317-320` (`testImages:`) — never count them as shipped, they
never reach prod (`dev.mockUpstream`/`dev.mockOidc` are false there).

LOC counts below were measured directly (`find <dir> -name '*.go' -not -name
'*_test.go' | xargs wc -l`) — re-measure if this file feels stale, code changes faster
than docs.

## The 12 Wasm plugins

Priorities and the request/response contract for each live in
`references/request-path.md`. This file covers **why each exists** — specifically, which
Higress built-in was evaluated first and why it failed, where that reasoning is recorded.

### `model-allowlist` — 127 LOC, `plugins/model-allowlist/`

Smallest plugin, and the clearest example of "we wrote this only because nothing else
worked." `docs/ARCHITECTURE.md:649-652` records the full rejection list: **no Higress
2.2.2 built-in correlates a request-header group with a per-group model allow-list in one
rule** — `model-mapper` only maps model names, it doesn't authorize; `ai-proxy` passes
unlisted models through unchanged; `request-validation` would need one Ingress route per
group (doesn't scale with tenant count); `ai-security-guard` is Alibaba-Cloud-coupled
(phones out, violates the local-only guardrail requirement).

### `prompt-guard` — regex prompt-injection classifier, `plugins/prompt-guard/`

`docs/ARCHITECTURE.md:705,718-720`: `ai-data-masking` masks data PATTERNS (emails, keys)
but is not an injection classifier, and the only built-in injection guard
(`ai-security-guard`) is the same cloud-coupled plugin rejected above. So a small
self-contained regex guard, global baseline + per-project overlay
(`consumer_patterns{}`, rendered by the control-plane reconcile).

### `ai-cache` — the largest plugin, `plugins/ai-cache/`

A **fork**, not written from scratch — `docs/ARCHITECTURE.md:2375-2376` names it
explicitly: "forked Wasm exact-match + semantic cache… Apache-2.0 fork." Forked because
tenant isolation (`x-mse-consumer` → `org.project` scoping every Redis key and Qdrant
filter) wasn't in the upstream version. Two-tier design (Redis exact-match, Qdrant
semantic) and the full request/response contract are covered in `references/
request-path.md` and are worth reading directly from `plugins/ai-cache/main.go` +
`plugins/ai-cache/core.go` — this is the one plugin genuinely too large to fully
summarize here without losing the read/write asymmetry that matters.

### `key-injector` — provider key + model swap, `plugins/key-injector/`

Solves a problem no built-in addresses: the client authenticates with OUR key
(`sk-…`), but the real provider needs ITS OWN credential, per-consumer, per-model.
`routes{}` map keyed on consumer, containing per-model `{credential, upstream}` pairs.

### `tool-result-compressor`, `semantic-guard`, `auto-router`, `prompt-decorator`,
### `prompt-template`, `mcp-tenant-guard`, `ai-usage-reporter`, `ai-prompt-log`

Each solves a specific problem no built-in plugin covers — token cost from replayed
tool-call history, embedding-based (as opposed to regex) injection detection, resolving
`model:"auto"`, enforced per-project system prompts, named prompt templates, cross-MCP
tenant isolation, push-based usage accounting (see the ledger rationale in
`references/platform-ops.md`), and opt-in content logging respectively. Full behavior for
each is in `references/request-path.md`; source is `plugins/<name>/main.go`.

## The 3 shipped services (`components/`)

### `control-plane` — the largest artifact in the repo by far

`components/control-plane/`, Go, standard library only (no `client-go` — deliberately
dependency-free, same policy as the migration runner). **64 non-test `.go` files**, and
**`reconcile.go` alone is `1528` lines** (`components/control-plane/reconcile.go` — wc
-l confirms), the single largest file in the repository.

Five distinct jobs in one binary:

1. **Schema owner** — applies **48 numbered embedded migrations**
   (`components/control-plane/migrations/`, latest is `0051_pe_conversations`), then
   grants a least-privilege `opsta_app` role and serves the read API as that role.
   Forward-only, additive — see this repo's CLAUDE.md prod-lifecycle rule.
2. **The write API** — orgs, projects, groups, users, consumers, API keys, budgets,
   limits, providers, guardrails, prompts, agents. Tiered authz
   (platform_admin/org_admin/member), console-only header-trust, ClusterIP-only.
3. **The reconcile loop** — `reconcile.go` projects Postgres onto live Kubernetes
   objects roughly every 30 seconds: `WasmPlugin.matchRules` for the tenant-aware
   plugins, dynamic `ai-proxy` blocks (`egress.go:649-655` `aiProxyPluginSpec` + `aiproxy.go:179`
   `aiProxyMatchRulesFor`), dynamic `egress-*`
   HTTPRoutes, `mcp-server` lifecycle (`reconcile.go:979`, `applyMcpServerPlugin`),
   and `request-block` (the AI kill-switch, `reconcile.go:1206`). The FIRST reconcile
   runs synchronously at startup and gates `/readyz`
   (`components/control-plane/main.go:547`), so `helm --wait` and the smoke suite never
   observe a half-configured gateway.
4. **Usage ingest** — receives pushes from `ai-usage-reporter`, buffers in memory,
   flushes to `usage_ledger` on a timer. See `references/platform-ops.md` for why this
   is push-based rather than a Prometheus scrape.
5. **A second personality** — the same binary run with `--mode=lgtm-auth` becomes the
   observability auth proxy. See `references/platform-ops.md`.

### `console` — Next.js 16 / React 19 / TypeScript, `components/console/`

`components/console/package.json:19-38` confirms the stack (`next: 16.2.7`,
`react: 19.2.0`, `typescript: 5.9.3`). `output: standalone` build, deployed behind its
own `oauth2-proxy`. **Never sees a token** — identity arrives as
`x-dev-user`/`x-dev-group` headers injected by `oauth2-proxy`. All writes proxy through
to the control-plane on org-scoped routes. Two audiences in one app: the member portal
(self-service keys, usage, budget) and the admin surface
(`/admin/projects` — the canonical consumer-management page).

### `budget-controller` — 347 LOC CronJob, `components/budget-controller/main.go`

The file's own header comment (`main.go:1-11`) states the design directly: `ai-quota`
is token-denominated and default-**denies** a consumer with no balance, but the org
requirement is a USD budget. This CronJob reads each consumer's month-to-date usage
from Mimir, prices it per-model, converts the remaining USD budget to a token-equivalent
balance, and pushes it into `ai-quota`'s admin API
(`/v1/chat/completions/quota/refresh`). `main.go:11` notes the interplay explicitly:
because `ai-quota` also decrements per-request between CronJob runs, enforcement is
necessarily an approximation bounded by the run interval, not instantaneous-exact.

## The 3 test-only doubles — never shipped

From `version.yaml:317-320`, built locally for e2e, skipped by the dev CI matrix,
`promote-uat`, and `release`:

| Double | Stands in for |
|---|---|
| `mock-llm` (`components/mock-llm/`) | An OpenAI-style upstream — echoes the last user message so masking/rewriting is observable |
| `mock-oidc` (`components/mock-oidc/`) | Google — discovery, `/authorize`, `/token`, JWKS, Google-shaped `hd`/`groups` claims |
| `mock-pe-runtime` (`components/mock-pe-runtime/`) | The `opsta-ai-pe` agent runtime's SSE stream |

The principle behind all three: `task test` should never depend on a real provider or a
real IdP's uptime.

## The 9 BUILT-IN plugins — mirrored, configured, never rewritten

Distinct from the 15 above: these are **not our artifacts**. Authoritative list is
`version.yaml:329-341` (`aiPlugins:`), duplicated for the chart at
`charts/opsta-ai-gateway/values.yaml:648`. Nine, not eight — `ai-data-masking` belongs in
this list and is easy to forget because it ships disabled.

**How they get here.** Mirrored as OCI artifacts from the Alibaba registry
(`higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins`, `version.yaml:330`) into our own
registry — `task images` runs
`oras cp <source>/<name>:<tag> <registry.pull>/ai-gateway-plugins/<name>:<tag>`
(`version.yaml:324-326`), and the `WasmPlugin` CRs reference the **mirror**, so nothing
reaches cn-hangzhou at runtime. Supply-chain decision, not a convenience one.

**Tag `2.0.0`** (`version.yaml:331`) — and read the comment above it
(`version.yaml:327-328`): that's the tag the Higress console catalog pins, **NOT** the
plugins' internal `1.0.0` VERSION files. Anyone browsing upstream by VERSION file will
pick the wrong source.

| Built-in | Rendered by | Job | Source of its config |
|---|---|---|---|
| `key-auth` | chart | API key → `x-mse-consumer` | `wasmplugins.yaml`, `consumers[]` from reconcile |
| `ai-statistics` | chart | usage → Prometheus token metrics | `wasmplugins.yaml` |
| `ai-quota` | chart | 403 over USD-derived token balance | `wasmplugins.yaml`; balance from `budget-controller` |
| `ai-token-ratelimit` | chart | 429 over tokens/minute | `wasmplugins.yaml`; bindings from reconcile |
| `model-router` | chart | body `model` → `x-higress-llm-model` | `wasmplugins.yaml` |
| `ai-data-masking` | chart | PII mask/deny, request+response | `values.yaml:346-360` — **`enabled: false`** |
| `ai-proxy` | **control-plane** | native provider protocol translation | `egress.go` `aiProxyPluginSpec` + `aiproxy.go` `aiProxyMatchRulesFor` |
| `mcp-server` | **control-plane** | MCP proxy; exists only while ≥1 server registered | `reconcile.go` `applyMcpServerPlugin` |
| `request-block` | **control-plane** | AI kill-switch (503) from suspended scopes | `reconcile.go` `reconcileKillSwitch` |

The last three have **no static priority to cite** — they aren't in `wasmplugins.yaml` at
all, and their lifecycle (created AND deleted) is owned by our control-plane, not ArgoCD.
That's the sharpest form of Reveal #5: for these, the reconcile loop owns the whole object,
not just `.spec.matchRules`.

**`ai-data-masking` is off by default and this is deliberate** —
`charts/opsta-ai-gateway/values.yaml:347-357`. The upstream 2.0.0 plugin truncates any
streaming response containing `reasoning_content` + `tool_calls`: it buffers the body to
mask PII, chokes on the reasoning→tool-call transition, and drops both the `tool_calls` and
the final `finish_reason` — breaking tool-calling for every streaming agentic client
(opencode, Cursor, …). It has no config to skip response/streaming masking and can't run
request-only, so it's all-or-nothing. Off until upstream ships a streaming-safe version.
See the ladder in `SKILL.md` — this is the single most common place a docs-only answer
goes wrong.

### Reading a built-in's internals

None of these nine have source in this repo — only config. When a question needs the
implementation ("how does key-auth actually look up a key", "what does ai-proxy send to
Bedrock", "what's the full schema ai-quota accepts"), go upstream:

```
github.com/higress-group/higress → plugins/wasm-go/extensions/<name>/
```

pinned to the mirrored tag **`2.0.0`**, not `main`. Ladder rung 5 — read the rules in
`SKILL.md` under "Upstream Higress source" before citing it. In short: label it "upstream
source, not verified on our cluster", never present an upstream default as our behavior
(we override throughout `values.yaml`), and if you can't fetch it, say so rather than
reconstructing it.

**Where upstream and ours meet: `ai-cache`.** It is in `plugins/` — ours, one of the 15 —
but it's a **fork** of the upstream built-in, base and patch recorded in
`plugins/ai-cache/UPSTREAM.md` and attributed at `plugins/ai-cache/go.mod:1` to
`github.com/alibaba/higress/plugins/wasm-go/extensions/ai-cache` (Apache-2.0). Note the
repo's own attribution uses the `alibaba/higress` coordinate — quote that when citing the
fork base. `UPSTREAM.md` states the reason to fork: stock `ai-cache` has a flat static
`cacheKeyPrefix` and a static `collectionID` baked into all Qdrant URLs, with no
per-consumer dimension and no search filter, so a front-wrapper could namespace the Redis
key but never isolate the semantic vector store — tenant isolation had to move inside the
plugin. Reveal #4 again, in its most expensive form: the constraint cost us a fork.

This is the highest-value upstream use in general — **diff, don't describe**. "Upstream
defaults X, we set Y, here's why" is an answer; "upstream does X" alone is a guess about
our cluster.

## What was deliberately NOT built here

Worth stating alongside the inventory, because each is its own small decision:

- The nine built-ins above — mirrored and configured, not rewritten. See that section for
  why each was kept rather than replaced (and `model-allowlist` / `prompt-guard` above for
  the two cases where a built-in was evaluated and rejected).
- Every Deployment/Service/Ingress for THIS repo's own workloads is meant to come from
  opsta's OneChart, not a hand-rolled template — see this repo's CLAUDE.md rule on
  OneChart. Existing bespoke chart templates for `control-plane`/`console` predate that
  rule and are migration debt, not the target pattern for anything new.
