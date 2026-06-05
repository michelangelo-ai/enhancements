# Enhancement Design Team

Team name: `enhancement-design`  
Config: `.claude/design-team/team-config.json`

## Setup — confirm repo paths before launching

`team-config.json` uses two path placeholders that must be substituted before use:

| Placeholder | Default | Description |
|---|---|---|
| `{{enhancements-repo}}` | current directory (this repo) | Absolute path to your `enhancements` clone |
| `{{michelangelo-repo}}` | `../michelangelo` relative to enhancements | Absolute path to your `michelangelo` clone |

When creating the team, confirm both paths with the user:

```
enhancements-repo: <current working directory>   # e.g. /home/you/michelangelo-ai/enhancements
michelangelo-repo: <enhancements-repo>/../michelangelo  # e.g. /home/you/michelangelo-ai/michelangelo
```

Then substitute the placeholders in a local copy of `team-config.json` before spawning agents. Do **not** commit the substituted copy — it contains user-specific paths.

## Members

| Role | Agent name | Responsibility |
|---|---|---|
| **Team Lead** | (you / Claude Code) | Coordinates, synthesizes RFC, owns final output |
| **Architect** | `architect` | Maps system architecture, designs component interfaces and overall system interaction |
| **Researcher** | `researcher` | Identifies and researches relevant OSS solutions and ecosystem patterns |
| **Developer Advocate** | `dev-advocate` | Sets OSS community DX quality bar, reviews contributor experience |
| **Coder** | `coder` | Reviews implementation quality, defines OSS-grade coding standards for new features |

## Workflow

1. Architect + Researcher + Dev Advocate + Coder explore in parallel (Tasks 1–4)
2. Team Lead synthesizes findings + drafts RFC (Task 5, blocked on 1–4)
3. All agents review RFC draft
4. Final RFC committed to `rfcs/`

## Findings files (written by agents)

- `{{enhancements-repo}}/architect-findings.md` — architecture map, APIs, CRDs, data flows
- `{{enhancements-repo}}/researcher-findings.md` — OSS ecosystem comparison, design lessons
- `{{enhancements-repo}}/dev-advocate-findings.md` — DX audit, quality rubric, RFC DX checklist
- `{{enhancements-repo}}/coder-findings.md` — implementation patterns, RFC implementation checklist

## RFC output location

`rfcs/NNNN-<feature>.md` following the template in `README.md`
