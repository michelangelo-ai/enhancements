# Enhancement Design Team

Team name: `enhancement-design`  
Config: `.claude/team-config.json`

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

- `.claude/architect-findings.md` — architecture map, APIs, CRDs, data flows
- `.claude/researcher-findings.md` — OSS ecosystem comparison, design lessons
- `.claude/dev-advocate-findings.md` — DX audit, quality rubric, RFC DX checklist
- `.claude/coder-findings.md` — implementation patterns, RFC implementation checklist

## RFC output location

`rfcs/NNNN-<feature>.md` following the template in `README.md`
