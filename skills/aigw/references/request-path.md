# The request and response path, plugin by plugin

Read `SKILL.md`'s source-of-truth ladder before trusting anything here. Priorities and
line numbers in this file are cited from `charts/opsta-ai-gateway/templates/wasmplugins.yaml`
and `charts/opsta-ai-gateway/templates/mcp.yaml` as of this file's last edit — re-check
those files if a number here looks surprising, since they can drift after this reference
does.

## How to read this file

Every Wasm plugin gets called by Envoy on the way **in** (request phase:
`onHttpRequestHeaders` / `onHttpRequestBody` or the plugin's equivalently-named function)
and, for the plugins that register it, on the way **out** (response phase:
`onHttpResponseHeaders` / `onHttpResponseBody` / streaming variants). Most plugins act on
only one side. `ai-cache` and the telemetry plugins act on both.

For each plugin: what it reads from platform state, what it writes, and the `file:line`
where that behavior lives in source.

## Request phase, in priority order (AUTHN phase — higher priority runs first)

### `key-auth` — priority 1000, built-in
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:17-58`

Reads the `Authorization` header against a static `consumers[]` list rendered into this
plugin's own `WasmPlugin.spec.matchRules[0].config` (one entry per API key). On match,
writes `x-mse-consumer: <org>.<project>.<user>` onto the request. Everything after this
point in the chain reads that header, never the raw key. 401 with body
`No Key Authentication information found` if no match.

This is a Higress built-in (mirrored, not vendored source) — its internal lookup
implementation is not in this repo to cite. What IS verifiable: `consumers[]` in the CR is
a flat JSON array (see the shape at `wasmplugins.yaml:17-58`), but that describes the
*config format*, not the runtime lookup structure — proxy-wasm convention (confirmed by
every one of OUR plugins below) is to parse config once at `onPluginStart` into a Go map,
not to rescan the array per request.

### `ai-data-masking` — priority 991, built-in
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:143-160`

**OFF by default** — `charts/opsta-ai-gateway/values.yaml:357` (`enabled: false`). See
`SKILL.md`'s source-of-truth ladder for the full explanation (upstream truncates streaming
responses containing `reasoning_content` + `tool_calls`). If an operator has explicitly
enabled it: masks/denies PII patterns in both request and response bodies via local
grok+regex, no external call.

### `mcp-tenant-guard` — priority 990, ours
`charts/opsta-ai-gateway/templates/mcp.yaml:18-36`, source `plugins/mcp-tenant-guard/main.go:19-21`

`onHeaders` reads `x-mse-consumer` and the `:path` pseudo-header
(`plugins/mcp-tenant-guard/main.go:19-21`). Only acts on MCP tool-call paths; enforces
that the consumer's project matches the MCP server being reached, otherwise 403. A
no-op for any non-MCP request such as a plain `/v1/chat/completions` call.

### `prompt-template` — priority 980, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:321-335`, source `plugins/prompt-template/main.go:103-113`

`onRequestBody` (`plugins/prompt-template/main.go:103`) reads `x-mse-consumer`
(`main.go:113`) and checks the body for a template reference. If present, expands it into
real `messages[]` content. Runs BEFORE the guards (970, 960) so they scan real text, not a
template name.

### `prompt-guard` — priority 970, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:177-194`

Reads `x-mse-consumer`, looks up that consumer's entry in `consumer_patterns{}` (a
per-project overlay rendered by the control-plane reconcile), and combines it with the
global baseline `patterns[]` from `values.yaml`. Runs a case-insensitive regex match over
`messages[].content`. 403 with header `x-aigw-blocked-by: prompt-guard` and body
`code: prompt_guard_denied` on a hit; otherwise passes through unchanged.

### `tool-result-compressor` — priority 965, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:707-721`, source `plugins/tool-result-compressor/main.go:157,173-180`

`onRequestBody` (`main.go:173`) reads `x-mse-consumer` (`main.go:178`) and checks the
consumer's **project** key against `config.tenants{}` (`main.go:157,180` — the map is
keyed by project, not full consumer). If the project opted in, rewrites ONLY
`role:"tool"` message content — lossless `json.Compact` normally, or truncated with an
elision marker in `aggressive` mode. Everything else in the body (user/system/assistant
content, `tool_call_id`) is untouched. A no-op if the project never opted in, or if the
body has no `role:"tool"` messages yet.

### `semantic-guard` — priority 960, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:571-585`, source `plugins/semantic-guard/main.go:34-59`

Two-step, both gated on the tenant map:
- `onHeaders` (`main.go:34`) reads `x-mse-consumer`, derives the project, checks
  `c.tenants{}` (`main.go:40`) — pauses the request if opted in, else passes through.
- `onBody` (`main.go:47`) embeds the user's message via an Ollama cluster call
  (`wrapper.NewClusterClient` at `main.go:58`), then searches Qdrant
  (`main.go:59`) for tenant-scoped deny/allow vectors. Deny hit above threshold, or an
  allow-list configured with no allow hit, → 403. Errors anywhere → fail open (pass).

### `prompt-decorator` — priority 950, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:283-297`, source `plugins/prompt-decorator/main.go:128-133`

`onRequestBody` (`main.go:128`) reads `x-mse-consumer` (`main.go:133`) and, if the
project has an enforced system prompt configured, injects it. Runs AFTER the guards
(970, 960) deliberately — so they scan the user's original text, not our injected system
prompt, which could otherwise hide an injection underneath a mandatory project prompt.

### `auto-router` — priority 940, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:618-632`, source `plugins/auto-router/main.go:122,183-225`

`onRequestHeaders` (`main.go:183`) reads `x-mse-consumer` (`main.go:187`), checks the
project against `config.tenants{}` (`main.go:192`, populated per-tenant at
`main.go:122`) — no-op if not opted in. If the body's `model` field is literally
`"auto"`, `onRequestBody` (`main.go:200`) embeds the prompt (`main.go:224-225`,
`wrapper.NewClusterClient` to the embedding + vector clusters) and searches for the
nearest configured route; above the tenant's threshold, REWRITES `body.model` to that
route's concrete model before `model-router` (880) or `model-allowlist` (800) ever run —
so an auto-resolved model still gets authorized normally, no allow-list bypass.

### `model-router` — priority 880, built-in
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:112-131`

Reads `body.model`, writes `x-higress-llm-model` header. This header — not the JSON
body field — is what routing (the terminal router filter, see
`references/higress-internals.md`) actually matches on downstream.

### `model-allowlist` — priority 800, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:355-372`

Reads `x-mse-consumer` → looks up `consumer_groups{}` for the consumer's group name →
looks up `group_models{}` for that group's allowed model list → checks `x-higress-llm-model`
against it (set two steps earlier by `model-router`). 403 if not allowed, before any
token accounting happens (800 runs before the cache at 500 and before the provider is
ever reached).

### `ai-cache` — priority 500, ours (fork)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:515-532`, source `plugins/ai-cache/main.go`, `plugins/ai-cache/core.go`

The largest plugin, and the only one with a symmetric request+response contract. Full
detail (both phases, the exact-match/semantic two-tier design, fail-open behavior) is
worth its own read at `plugins/ai-cache/main.go:66-108` (request phase: tenant gate,
build cache key, `CheckCacheForKey`) and `plugins/ai-cache/main.go:212-236` plus
`plugins/ai-cache/core.go:239-274` (response phase: accumulate the final chunk,
`cacheResponse` writes Redis, `uploadEmbeddingAndAnswer` writes Qdrant). On a HIT, the
request phase SHORT-CIRCUITS here — `proxywasm.SendHttpResponseWithDetail`
(`plugins/ai-cache/core.go:78`) sends the cached answer directly; the router filter never
runs, the provider is never called, and every plugin below this line
(`key-injector` through `ai-token-ratelimit`) is skipped entirely for this request.

### `key-injector` — priority 400, ours
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:400-417`

Reads `x-mse-consumer`, looks up `routes{<consumer>}{<model>}` for the real provider
credential and upstream name, and replaces the caller's key in the outbound
`Authorization` header with it. Also injects `stream_options.include_usage=true` on
streaming requests, without which OpenAI-compatible providers omit the `usage` block
from SSE responses — breaking `ai-usage-reporter` and `ai-statistics` downstream. This is
the last request-phase plugin before the router filter.

## The router filter — not a plugin

Between `key-injector` (400) and the response-phase plugins, the AUTHN phase boundary
passes (native `rbac`/`local_ratelimit` filters,
`docs/HIGRESS-INTERNALS.md:319-321`), and then Envoy's own native
**router filter** (`envoy.filters.http.router`) matches the request against an
`HTTPRoute`, selects a cluster, and proxies to the upstream provider. This is not
configurable via any `WasmPlugin` CR — see `references/higress-internals.md` for the
mechanics.

## Response phase — as the reply streams back (only reached if nothing short-circuited)

### `ai-usage-reporter` — priority 350, ours (response phase)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:443-460`, source `plugins/ai-usage-reporter/main.go:64-65,135,160,198`

`onRequestHeaders` (`main.go:64`) captures `x-mse-consumer` (`main.go:65`) into request
context for later. On the response side, `onRespBody`/`onStreamBody`
(`main.go:160`, `:198`) parse the `usage` block from the response — reassembling it
across SSE chunks when streaming — normalize into `{input, cache_read, cache_write,
output}` tiers, and fire-and-forget POST to the control-plane via a cluster client
(`main.go:135`, `NewClusterClient` — not Service DNS, same sandbox constraint as
everything else).

### `ai-prompt-log` — priority 340, ours (response phase)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:240-254`, source `plugins/ai-prompt-log/main.go:118-119,139,233`

`onRequestBody` (`main.go:118`) reads `x-mse-consumer` (`main.go:119`) and checks
opt-in. `onStreamBody` (`main.go:139`) accumulates the model's output text; on
completion, POSTs `{consumer, model, userText, systemText, outputText}` to the
control-plane (`main.go:233`). The only plugin in this chain that stores prompt or
response **content** — every other telemetry plugin stores counts only. Off unless a
project explicitly opted in.

### `ai-statistics` — priority 900, built-in (default phase)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:72-91`

Parses the same `usage` block independently, writes a Prometheus metric per
`{consumer, model}` pair. This is a SEPARATE counter from the control-plane ledger
`ai-usage-reporter` populates — Grafana reads this one; the console reads the ledger.

### `ai-quota` — priority 750, built-in (default phase)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:476-493`

Decrements the consumer's USD-derived token balance in Redis (`redis.serviceName` at
`wasmplugins.yaml:502`) by this request's cost. The balance itself is written by the
`budget-controller` CronJob, not by this plugin — see `references/components.md`. 403
with body `Request denied by ai quota check, No quota left` on the NEXT request once
exhausted.

### `ai-token-ratelimit` — priority 600, built-in (default phase)
`charts/opsta-ai-gateway/templates/wasmplugins.yaml:659-676`

Decrements a tokens-per-minute Redis bucket (`wasmplugins.yaml:692`). 429 with body
`{"error":{"type":"rate_limit_exceeded"}}` when exceeded.

## The cache-hit shortcut, stated once clearly

If `ai-cache` (500) hits on the request side, **none** of the following ever run for
that request: `key-injector`, the router filter, the real provider call,
`ai-usage-reporter`, `ai-prompt-log`, `ai-statistics`, `ai-quota`, `ai-token-ratelimit`.
The saved tokens are real but were never spent, so they're reported separately via
`plugins/ai-cache/core.go`'s `reportCacheHit` (POST to `/api/cache-hits`, a non-billed
savings ledger) rather than through the normal usage path — and quota/rate-limit
balances are untouched by a hit.

## Sentence version of the whole chain

Authenticate the caller into a consumer → mask PII (if enabled) → check MCP tenant
isolation → expand any prompt template → screen for injection (regex, then semantic) →
compress old tool results → inject the enforced project prompt (after the guards scan
the real user text) → resolve `model: "auto"` if requested → route on the resolved model
→ authorize that model against the consumer's group → check the cache (a hit ends the
whole story here) → swap in the real provider key → **(the provider is called)** → parse
what it cost from the response → optionally log the content → meter it against
Prometheus, the USD budget, and the per-minute rate limit.
