# Debugging reasoning — which gate blocked a request, and how to check

This skill never runs a cluster command itself (see `SKILL.md` — "What this skill will
never do"). This reference's job is to reason about a symptom accurately and hand over
the exact command the user should run, quoted from `docs/HIGRESS-INTERNALS.md` §8, which
was verified working against `ai-gateway-dev` on 2026-08-11.

## Step 1 — identify which gate rejected the request

`docs/ARCHITECTURE.md:178-193` — this table is the single most useful thing in this
reference file. Our custom plugins set an explicit `x-aigw-blocked-by` header; the
mirrored built-ins can't be modified to add that header, so they're identified by
status + exact body text instead:

| Gate | Status | How to identify | Line ref |
|---|---|---|---|
| `key-auth` (built-in) | 401 | body `No Key Authentication information found` | `ARCHITECTURE.md:187` |
| `ai-token-ratelimit` (built-in) | 429 | body `{"error":{"type":"rate_limit_exceeded"}}` | `ARCHITECTURE.md:188` |
| `ai-quota` (built-in) | 403 | body `Request denied by ai quota check, No quota left` | `ARCHITECTURE.md:189` |
| `ai-data-masking` (built-in, off by default) | 403 | the configured `deny_message` | `ARCHITECTURE.md:190` |
| `model-allowlist` (ours) | 403 | header `x-aigw-blocked-by: model-allowlist`, body `code: model_allowlist_denied` | `ARCHITECTURE.md:191` |
| `prompt-guard` (ours) | 403 | header `x-aigw-blocked-by: prompt-guard`, body `code: prompt_guard_denied` | `ARCHITECTURE.md:192` |
| an unknown prompt template (ours) | 404 | header `x-aigw-blocked-by: …` | `ARCHITECTURE.md:759` |

If the report is "connection reset" or a curl-level failure rather than an HTTP status,
you are NOT in this table at all — go to "Connection reset vs 404" below first, it's a
different, earlier layer.

## Step 2 — if the answer isn't in the table, reason about layer, not plugin

Every plugin's request-side and response-side reads/writes are in
`references/request-path.md`. Before assuming a specific plugin, check WHICH phase the
symptom is consistent with:

- **Rejected before the provider was ever called** → an AUTHN-phase plugin
  short-circuited. Walk the priority-ordered table in `SKILL.md` top to bottom — the
  first plugin whose job matches the symptom is usually the culprit (a 403 about a model
  is `model-allowlist`, not `prompt-guard`).
- **Response looks wrong or truncated, but a 200 came back** → likely
  `ai-data-masking` if enabled (streaming truncation bug, see the ladder in `SKILL.md`),
  or `ai-cache` serving a stale cached answer (check the tenant's cache TTL).
- **Request succeeded but usage/budget numbers look wrong** → the response-phase
  plugins (`ai-usage-reporter`, `ai-statistics`, `ai-quota`, `ai-token-ratelimit`) —
  check whether the request was actually a cache HIT, since a hit skips every one of
  them (see the "cache-hit shortcut" section of `references/request-path.md`).
- **A specific project's guardrail/cache/routing behaves differently from another
  project's** → almost certainly a `tenants{}`/`consumer_patterns{}`/`consumer_groups{}`
  entry difference for that consumer's project, not a bug — see anti-pattern #7 in
  `SKILL.md`.

## Connection reset vs 404 — a different layer entirely

If the symptom is a TCP-level failure (`curl: (35) recv failure`, connection reset, no
HTTP status at all) rather than a 4xx body, the request never got past TLS/SNI matching
— it is NOT a plugin problem. See `references/higress-internals.md` ("The listener
reality check") and Reveal #1 in `SKILL.md`: Envoy's `:443` listener has no
`default_filter_chain`, so an unrecognized SNI hostname drops the connection outright.
Check the hostname against this environment's dash-separator convention
(`api-ai-gateway-dev.opsta.in.th`, not `api.<baseDomain>`) before suspecting anything
else — a wrong hostname is the single most common cause of this exact symptom.

## Config-not-taking-effect

Before assuming a plugin bug, establish WHICH of the four config-change paths applies —
see the decision table at the end of `references/platform-ops.md`. A tenant-data change
with no corresponding Postgres row is correctly doing nothing; a chart-value change that
was never pushed through `task gitops:push` is sitting invisible in the working tree.
Only once the right path is confirmed does it make sense to suspect the reconcile loop
or ArgoCD sync itself.

## config_dump forensics — commands for the user to run

All of the following are quoted verbatim from `docs/HIGRESS-INTERNALS.md` §8 (all
confirmed working against `ai-gateway-dev` on 2026-08-11). This skill hands these over;
it does not execute them. Context note repeated here because it matters for every one of
these: the working context is `ai-gateway-dev`, NOT `k3d-opsta-ai-gateway-dev` (that
kubeconfig entry exists but the connection refuses); always pass `--context` explicitly.

**Every declared plugin with its phase + priority** — the fastest sanity check of the
policy chain as CONFIGURED (expect `UNSPECIFIED_PHASE` for default-phase plugins, not
`DEFAULT`):

```bash
kubectl --context ai-gateway-dev -n higress-system get wasmplugin \
  -o custom-columns='NAME:.metadata.name,PHASE:.spec.phase,PRIORITY:.spec.priority'
```

**Fetch the Envoy config dump once** (~570 KB — save it, don't re-fetch per question,
Envoy admin port is `15000`):

```bash
kubectl --context ai-gateway-dev -n higress-system exec deploy/higress-gateway -- \
  curl -s localhost:15000/config_dump > /tmp/dump.json
```

**Which Wasm modules Envoy actually ACCEPTED (ECDS)** — compare this against the
`wasmplugin` list above; a plugin present in the CR list but MISSING here means Envoy
rejected the module (this is how you'd catch, for example, the documented
high-WasmPlugin-churn config-freeze failure mode below):

```bash
jq -r '.configs[] | select(.["@type"]|test("EcdsConfigDump")) | .ecds_filters[].ecds_filter.name' /tmp/dump.json
```

**The real ordered HTTP filter chain for the `api` host**, native filters included — the
single most useful command in the whole reference set, because it shows what's actually
running, not what was merely declared:

```bash
jq -r '.configs[] | select(.["@type"]|test("ListenersConfigDump")) | .dynamic_listeners[] | select(.name=="0.0.0.0_443") | .active_state.listener.filter_chains[] | select(((.filter_chain_match.server_names // [""])[0]) | startswith("api-")) | .filters[] | select(.name|test("http_connection_manager")) | .typed_config.http_filters[].name' /tmp/dump.json
```

**Envoy clusters**, including the McpBridge-derived `*.dns` ones (useful when suspecting
a plugin can't reach Redis/Qdrant/Ollama/control-plane — confirm the cluster exists
before suspecting the plugin logic):

```bash
jq -r '.configs[] | select(.["@type"]|test("ClustersConfigDump")) | .dynamic_active_clusters[].cluster.name' /tmp/dump.json
```

**The CRDs themselves, and the git-vs-database route split** (see Reveal #5 in
`SKILL.md` — routes with no source in git are correctly control-plane-owned, not a bug):

```bash
kubectl --context ai-gateway-dev -n higress-system get gateway,httproute,wasmplugin,mcpbridge
```

**Controller/pilot logs**, when a CRD change doesn't seem to take effect at all:

```bash
kubectl --context ai-gateway-dev -n higress-system logs deploy/higress-controller -c higress-core --tail=100
```

## Known failure modes — recorded, not hypothesized

- **Gateway config-freeze under high `WasmPlugin` churn** — `ai-proxy` wasm fails to
  instantiate when many `WasmPlugin` writes happen in quick succession, and Envoy
  rejects the listener entirely. Documented reproducing at BOTH the pre- and
  post-memory-bump resource limits with zero OOM restarts recorded — meaning this is
  NOT a memory problem, despite looking like one. If someone reports a channel-wide
  outage that started after a burst of admin activity (many consumers/keys/guardrails
  added quickly), this is the first thing to suspect, and the ECDS-vs-CR comparison
  command above is how to confirm it.
- **413 on real agent payloads** — body-inspecting plugins must buffer the WHOLE
  request/response body, and the chart deliberately raises
  `connectionBufferLimits` to 10 MiB (both directions) specifically because the 32 KiB
  Envoy default was 413-ing real agent conversation histories. If a 413 shows up after a
  values change, check whether `gateway.maxRequestBytes` and the buffer limits drifted
  out of lock-step — they're required to move together.
- **`ai-data-masking` streaming truncation** — see the source-of-truth ladder in
  `SKILL.md`. If someone has explicitly enabled this plugin and reports missing
  `tool_calls` or a cut-off `finish_reason` on streaming responses, this is the known,
  documented cause, not a new bug.
