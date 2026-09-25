# Datagoat agent skills

Five skills that teach an agent to use [Datagoat](https://datagoat.io), one for each journey, and a
Claude Code plugin that installs them with the Datagoat MCP server.

## Install in Claude Code

```
/plugin marketplace add dmilstein-match/datagoat-skills
/plugin install datagoat@datagoat-skills
```

Then run `/mcp` and sign in to Datagoat. Your account is made at first sign-in; the free samples need
no card.

## The skills

| Skill | Journey | Use it when |
|---|---|---|
| [datagoat-first-run](plugins/datagoat/skills/datagoat-first-run/SKILL.md) | Try it | someone wants to see what Datagoat does, on the free samples |
| [datagoat-ask](plugins/datagoat/skills/datagoat-ask/SKILL.md) | Ask your data | answering a yes/no, level, best-option or ranking question from past outcomes |
| [datagoat-product](plugins/datagoat/skills/datagoat-product/SKILL.md) | Ship a product, Run it | scoring customers' cases on a schedule, and keeping the models healthy |
| [datagoat-gate](plugins/datagoat/skills/datagoat-gate/SKILL.md) | Ship a product | an agent will act on an answer and must know when a person decides |
| [datagoat-prove](plugins/datagoat/skills/datagoat-prove/SKILL.md) | Prove it | showing that answers were right, or that acting on them worked |

Every skill keeps one rule: never state a chance, level, reason, range or direction the answer does
not contain. Each has evals in its `evals/evals.json`.

## Other agents

Each `SKILL.md` stands alone: put it wherever your agent reads skills, and connect the MCP server
`https://api.datagoat.io/mcp` (or use the REST API with a key). The same files are served at
`https://datagoat.io/skills/<name>/SKILL.md`.

## Contributing

These files are published from Datagoat's main repository. Issues and pull requests are welcome; a
change is ported there and comes back with the next release. Apache-2.0.
