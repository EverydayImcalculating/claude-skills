# claude-skills

Personal Claude Code skills/agents, each packaged as its own installable plugin so
people can grab just the one they want instead of the whole set.

## Install

Add the marketplace once, per machine:

```
/plugin marketplace add EverydayImcalculating/claude-skills
```

Then install only what you want:

```
/plugin install aigw
/plugin install cluster
/plugin install daily-update
/plugin install opstella-specialist
```

(Or, before it's pushed to GitHub, add the marketplace by local path:
`/plugin marketplace add /Users/fsma/Opsta/claude-skills`.)

## Plugins

| Plugin | Type | What it's for |
|---|---|---|
| [`aigw`](plugins/aigw/skills/aigw/SKILL.md) | skill | Explains the Opsta AI Gateway (`ai-gateway` repo) and the Higress gateway it runs on — request/response plugin chain, components, GitOps ops, debugging 401/403/429. Repo-specific: its citations only resolve inside that repo's working tree. |
| [`cluster`](plugins/cluster/skills/cluster/SKILL.md) | skill | Opsta infrastructure-provisioning consultant — Proxmox/vCenter VM provisioning, RKE2 install, Rancher, Istio Gateway API ingress, Harbor pull-through cache. |
| [`daily-update`](plugins/daily-update/skills/daily-update/SKILL.md) | skill | Summarizes a Claude Code session into a Thai-language team daily-update block and appends it to a dated log. |
| [`opstella-specialist`](plugins/opstella-specialist/agents/opstella-specialist.md) | agent | Opstella IDP Day-2 ops specialist — ArgoCD/GitLab/Harbor/Vault/Keycloak/OneChart troubleshooting, roles, Deltron cluster topology. Ships with a trimmed knowledge base ([`knowledge/opstella/`](plugins/opstella-specialist/knowledge/opstella/)) — deliberately **without** `07-runbook.md`, a live incident log specific to one private environment. |

## Adding a new plugin

```
mkdir -p plugins/<name>/skills/<name>
# write plugins/<name>/skills/<name>/SKILL.md (+ references/ if needed)
```

Then:
1. Write `plugins/<name>/.claude-plugin/plugin.json` (name, version, description, author,
   license, `"skills": "./skills/"` — copy an existing plugin's file as a template).
2. Add an entry to `.claude-plugin/marketplace.json`'s `plugins` array with
   `"source": "./plugins/<name>"`.

Each plugin installs independently — `/plugin install <name>` only pulls that one in.

## Updating a skill from its source repo

Skills that describe a specific codebase (like `aigw`) are edited in place there and
copied here when ready to publish:

```
cp -R /path/to/source-repo/.claude/skills/<name> plugins/<name>/skills/<name>
```

There is currently no symlink/sync automation — copy-in-and-commit is a manual step.
