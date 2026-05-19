# Michelangelo Enhancements

This repository tracks architectural RFCs (Requests for Comments) for [Michelangelo](https://github.com/michelangelo-ai), Uber's open-source ML platform. It is the public counterpart to internal Engineering Review Documents (ERDs) and serves as the primary venue for community-visible design discussions.

## What belongs here

- Architecture proposals for new platform features
- Design changes that affect public APIs, CRDs, or operator contracts
- Proposals from external contributors and design partners
- Decisions that benefit from broad community input before implementation

Uber-internal concerns (privacy review, observability standards, internal service integrations) are tracked in the corresponding internal ERD and linked from the RFC.

## Repository structure

```
/docs
  /architecture       # Stable architecture reference docs
    README.md
/rfcs
  0001-<feature>.md   # Accepted or in-review RFCs
  0002-<feature>.md
```

## RFC lifecycle

```
Idea → Draft PR → Community Review → Accepted / Withdrawn → Implementation
```

1. **Draft** — open a PR adding `rfcs/NNNN-<short-name>.md`
2. **Review** — use GitHub PR comments for architecture discussion; link the PR in GitHub Discussions for broader input
3. **Accepted** — PR is merged; implementation PRs link back to the RFC
4. **Withdrawn** — PR is closed with a summary of why

RFCs do not require unanimous approval, but significant objections from maintainers or affected parties should be resolved before merging.

## Writing an RFC

Copy the [RFC template](docs/architecture/README.md) and fill in:

| Section | Guidance |
|---|---|
| **Problem statement** | What breaks or is missing today? |
| **Motivation** | Why solve it now? Who is affected? |
| **Goals / non-goals** | Explicit scope boundary |
| **High-level architecture** | Diagrams welcome (Lucidchart exports as PNG/SVG) |
| **APIs and CRDs** | Public interface changes |
| **Alternatives considered** | At least one alternative with tradeoffs |
| **Open questions** | Unresolved decisions |
| **Rollout strategy** | Phasing, feature flags, migration path |

Keep RFCs focused on architecture. Full implementation details, internal operational playbooks, and large code blocks belong in the implementation PRs.

## Contribution scenarios

**Uber → open source** — internal ERD is created first for organizational alignment, then a public RFC PR is opened in this repo. Both Uber team members and external design partners review the GitHub PR simultaneously.

**Open source only / external contribution** — RFC is opened directly here by an external contributor; Michelangelo maintainers review alongside any design partners.

**External contributor-driven feature** — external contributor opens the RFC and drives the design; Michelangelo team reviews and advises.

## Relationship to internal ERDs

Internal ERDs (tracked in uPlan) remain the place for:
- Organizational approvals
- Privacy and compliance sign-off
- Internal service integration notes (observability, on-call, etc.)

Public RFCs are the place for:
- Architecture transparency
- Community collaboration
- Versioned design history

Each internal ERD should link to its corresponding public RFC PR, and vice versa.

## Prior art

This process is inspired by:
- [Kubernetes Enhancement Proposals (KEPs)](https://github.com/kubernetes/enhancements)
- [Ray Enhancement Proposals (REPs)](https://github.com/ray-project/ray/tree/master/doc/source/ray-contribute/enhance_reps)
- [Temporal architecture docs](https://github.com/temporalio/temporal/tree/main/docs/architecture)

## Questions?

Open a [GitHub Discussion](../../discussions) for process questions or early-stage ideas that aren't ready for a formal RFC yet.
