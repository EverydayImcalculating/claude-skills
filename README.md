# claude-skills

Personal Claude Code skills, packaged as one plugin so they can be reused across
projects and shared with others.

## Install

From any Claude Code session:

```
/plugin marketplace add EverydayImcalculating/claude-skills
/plugin install claude-skills
```

(Or, before it's pushed to GitHub, add it by local path:
`/plugin marketplace add /Users/fsma/Opsta/claude-skills`.)

## Skills

| Skill | What it's for |
|---|---|
| [`aigw`](skills/aigw/SKILL.md) | Explains the Opsta AI Gateway (`ai-gateway` repo) and the Higress gateway it runs on — request/response plugin chain, components, GitOps ops, debugging 401/403/429. Repo-specific: its citations only resolve inside that repo's working tree. |

## Adding a skill

```
mkdir -p skills/<name>
# write skills/<name>/SKILL.md (+ references/ if needed)
```

Any dir under `skills/` with a `SKILL.md` is picked up automatically — no other file
needs editing.

## Updating a skill from its source repo

Skills that describe a specific codebase (like `aigw`) are edited in place there and
copied here when ready to publish:

```
cp -R /path/to/source-repo/.claude/skills/<name> skills/<name>
```

There is currently no symlink/sync automation — copy-in-and-commit is a manual step.
