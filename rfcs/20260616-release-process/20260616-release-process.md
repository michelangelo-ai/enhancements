# RFC-20260616: Coordinated Release Process

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-06-16

---

## Problem statement

Michelangelo publishes 7 independently versioned artifacts (Python SDK, Go containers, UI container, npm packages, Helm chart) with no coordinated release process. This creates three classes of problems:

1. **Version fragmentation.** Every component tracks its own version independently: Python SDK at 0.1.1, npm core at 0.2.4, Helm chart at 0.1.0/0.2.1, UI at 0.0.0. Users have no way to know which versions are compatible with each other.

2. **Missing publication channels.** The Python wheel is built in CI but never published to PyPI — users cannot `pip install michelangelo`. The Helm chart has no OCI registry push — users cannot `helm install` from a registry. Two of the project's most important installation paths are broken.

3. **No release automation.** There is no changelog generation (CHANGELOG.md is an empty stub), no release candidate process, no nightly builds, no release checklist, and no quality gates beyond CI tests. The tag format (`v0.1.0.beta.2`) doesn't follow SemVer or PEP 440, breaking downstream tooling.

## Motivation

Michelangelo is an open-source ML platform competing with Kubeflow, MLflow, Ray, and Flyte — all of which have professional, automated release processes. The absence of coordinated releases is the second-largest barrier to external adoption (after installation complexity, addressed by the Helm chart RFC).

Specific impact:

- **Users** cannot install the project through standard package managers (`pip`, `helm`, `npm`).
- **Contributors** have no changelog to understand what changed between versions.
- **Integrators** have no nightly builds to test against, and no compatibility matrix to verify their environment.
- **Maintainers** have no structured process for cutting releases, leading to ad-hoc, error-prone releases.

The Helm chart RFC (RFC-20260427) assumed a release pipeline would exist to publish charts to an OCI registry. This RFC delivers that pipeline.

## Goals

- All 7 artifacts share a unified **Major.Minor** version, starting at **v0.3.0**.
- `pip install michelangelo` installs the Python SDK from PyPI.
- `helm install michelangelo oci://ghcr.io/uber/michelangelo/charts/michelangelo` installs from the OCI registry.
- A single `git tag v0.3.0` on a release branch triggers publication of all artifacts.
- Nightly builds publish daily at 02:00 UTC with automatic integration testing validating that all 7 artifacts work together.
- `git-cliff` generates changelogs from conventional commits; GitHub Releases are auto-populated.
- A version-bump script updates all component files with a single command.
- A release checklist issue template guides maintainers through the release process.

## Non-goals

- **Long-term support (LTS) branches.** During v0.x, only the latest minor release branch is maintained. LTS can be revisited at v1.0.
- **Automated cherry-pick tooling.** Patch releases require manual cherry-picks from `main` to the release branch.
- **PyPI organization account setup.** This RFC covers the CI pipeline; PyPI org provisioning is a prerequisite handled separately.
- **Signing artifacts.** Sigstore/cosign signing is desirable but out of scope for v0.3.0.

## High-level architecture

### Release flow

```
main ─────●─────●─────●─────●─────●─────●─────●──── (development)
                      │                 │
                      ├─ release/v0.3 ──●── v0.3.0-rc.1
                      │                 │
                      │                 ●── v0.3.0 (final)
                      │                 │
                      │                 ●── v0.3.1 (cherry-pick patch)
                      │
                      ├─ release/v0.4 ──●── v0.4.0
```

### Artifact publication matrix

```
git tag v0.3.0
    │
    ├── release.yaml ──┬── Python wheel ──→ PyPI (poetry publish)
    │                  ├── Python wheel ──→ GitHub Release asset
    │                  ├── Helm chart ────→ ghcr.io OCI (helm push)
    │                  └── Go binaries ──→ GitHub Release assets (Phase 3)
    │
    ├── dev-release.yml ── Go containers ──→ ghcr.io
    │
    ├── ui-release.yml ─── UI container ───→ ghcr.io
    │
    ├── npm-publish.yml ─┬─ @michelangelo/core ──→ npm
    │                    └─ @michelangelo/rpc ───→ npm
    │
    └── changelog.yml ──── Release notes ──→ GitHub Release body
```

### Nightly flow

```
02:00 UTC daily (cron)
    │
    ├── nightly.yml ────── check main changed ──→ build all artifacts
    │                      with -nightly.YYYYMMDD suffix
    │
    └── nightly-integration.yml ── pull nightly artifacts
                                   ├── install Python SDK from PyPI (--pre)
                                   ├── deploy Helm chart to test cluster
                                   ├── pull Go + UI containers
                                   └── run integration test suite
                                       ├── PASS → no action
                                       └── FAIL → open GitHub Issue
```

### Version format

| Artifact | Stable | RC | Nightly |
|---|---|---|---|
| Git tag | `v0.3.0` | `v0.3.0-rc.1` | `v0.3.0-nightly.20260616` |
| Python (PEP 440) | `0.3.0` | `0.3.0rc1` | `0.3.0.dev20260616` |
| Containers / npm / Helm | `0.3.0` | `0.3.0-rc.1` | `0.3.0-nightly.20260616` |

## APIs and CRDs

No new APIs or CRDs. This RFC adds CI/CD infrastructure only.

Changes to existing files:

| File | Change |
|---|---|
| `.github/workflows/release.yaml` | Add PyPI publish step, Helm OCI push job, Go binary cross-compile (Phase 3) |
| `.github/workflows/dev-release.yml` | Add `tags: ['v*']` trigger |
| `.github/workflows/ui-release.yml` | Add `tags: ['v*']` trigger |
| `.github/workflows/nightly.yml` | **New** — scheduled daily build |
| `.github/workflows/nightly-integration.yml` | **New** — post-nightly integration validation |
| `.github/workflows/changelog.yml` | **New** — git-cliff on tag push |
| `.github/workflows/cve-scan.yml` | **New** — container image scanning (Phase 3) |
| `.github/workflows/cleanup-nightlies.yml` | **New** — prune artifacts >14 days (Phase 3) |
| `.github/workflows/compat-matrix.yml` | **New** — multi-version CI matrix (Phase 3) |
| `.github/ISSUE_TEMPLATE/release-checklist.md` | **New** — structured release checklist |
| `cliff.toml` | **New** — git-cliff configuration |
| `CONTRIBUTING.md` | Add conventional commit guide |
| `scripts/version-bump.sh` | **New** — update all component versions |

## Alternatives considered

### Alternative A: Single monolithic release workflow

Package all artifact builds into a single `release.yaml` workflow.

**Pros:** Single file to maintain. Clear sequential ordering.
**Cons:** A failure in npm publishing blocks Helm chart publication. No reuse of existing workflows. Much harder to debug individual artifact failures.
**Why not chosen:** The existing repo already has separate workflows per artifact type (`dev-release.yml`, `ui-release.yml`, `npm-publish.yml`). Extending them with tag triggers is less disruptive and preserves independent failure isolation.

### Alternative B: Release Please (Google)

Use [Release Please](https://github.com/googleapis/release-please) for automated versioning and changelog.

**Pros:** Mature, widely used. Handles version bumps via PR automation. Supports monorepos.
**Cons:** Opinionated about PR-based releases (creates release PRs automatically). Doesn't natively support PEP 440 pre-release formats. Would require significant configuration for 7 different artifact types with different version file formats.
**Why not chosen:** git-cliff provides changelog generation without imposing a release workflow. The explicit branch + tag model gives maintainers more control over the release cadence, which is important during v0.x when the process is still being refined.

### Alternative C: Start at v0.2.0

Use v0.2.0 as the first coordinated release version.

**Pros:** Follows the original proposal. Lower version number.
**Cons:** `@michelangelo/core` is already at npm version 0.2.4. Publishing 0.2.0 would be a version downgrade, which npm rejects.
**Why not chosen:** v0.3.0 is the lowest version that avoids conflicts with all existing published artifacts.

## Open questions

- [ ] Who provisions the `PYPI_TOKEN` secret on the GitHub org? (Required for Task 003)
- [ ] Should nightly Python packages include a runtime `DeprecationWarning`? (Proposed in Task 011 — may annoy CI users who intentionally pin nightlies)
- [ ] Should we adopt Release Drafter alongside git-cliff for PR-label-based notes? (Task 018 — evaluation)
- [ ] What Kubernetes versions should the compatibility matrix target? (Proposed: 1.28, 1.29, 1.30)

## Rollout strategy

The implementation is phased to deliver value incrementally:

### Phase 1: Foundation (target: v0.3.0-rc.1)

**Tracking:** [michelangelo-ai/michelangelo#1338](https://github.com/michelangelo-ai/michelangelo/issues/1338)

1. Adopt conventional commits — update `CONTRIBUTING.md`
2. Add `cliff.toml` — configure git-cliff
3. Add PyPI publishing — modify `release.yaml`, set up `PYPI_TOKEN` secret
4. Add Helm OCI publishing — add `helm push` job
5. Align all version numbers to 0.3.0
6. Fix tag format — use `vX.Y.Z` going forward

**Rollback:** All changes are additive. Removing the new workflow steps restores prior behavior.

### Phase 2: Automation (target: v0.3.0)

**Tracking:** [michelangelo-ai/michelangelo#1339](https://github.com/michelangelo-ai/michelangelo/issues/1339)

7. Create `nightly.yml` — daily scheduled builds
8. Create `changelog.yml` — auto-generated release notes
9. Add release checklist issue template
10. Add tag-based triggers to `dev-release.yml` and `ui-release.yml`
11. Add nightly runtime warning to Python SDK
12. Add version-bump script
13. Add nightly integration test workflow — validate all artifacts work together

**Rollback:** Disable cron schedules to stop nightlies. Remove tag triggers to revert to branch-only builds.

### Phase 3: Polish (target: v0.4.0)

**Tracking:** [michelangelo-ai/michelangelo#1340](https://github.com/michelangelo-ai/michelangelo/issues/1340)

14. Add Go binary cross-compilation
15. Add CVE scanning on container images
16. Add nightly retention cleanup (14-day window)
17. Add compatibility testing matrix
18. Evaluate Release Drafter

**Rollback:** Each item is an independent workflow. Disable or delete individually.

### Migration path

- **Existing users:** No breaking changes. The first coordinated release (v0.3.0) supersedes all prior independent versions.
- **Tag format:** `v0.1.0.beta.2` style tags are deprecated. New tags use `vX.Y.Z`. Old tags are not deleted.
- **Version numbers:** Component versions jump to 0.3.0. This is a forwards-only change.

## References

- [RFC-20260427: Michelangelo Helm Chart](../20260427-michelangelo-helmchart/20260427-michelangelo-helmchart.md) — Helm chart RFC (depends on OCI publishing from this RFC)
- [git-cliff documentation](https://git-cliff.org/) — changelog generator
- [PEP 440](https://peps.python.org/pep-0440/) — Python version identification
- [OCI Artifacts spec](https://github.com/opencontainers/artifacts) — Helm OCI registry standard
