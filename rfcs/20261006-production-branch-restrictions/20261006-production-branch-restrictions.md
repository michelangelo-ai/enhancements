# RFC-20261006-production-branch-restrictions: Configurable Production-Branch Restrictions

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-10-06
- **Internal ERD:** N/A

---

## Problem statement

Michelangelo's `api.EnvironmentLabel` (`michelangelo/environment`) is a free-string Kubernetes
label with no relationship to the git branch a `PipelineRun`, `TriggerRun`, or `Model`'s source
commit came from. A user can label a run `production` from any branch, fork, or pull request with
no guard rail -- neither the API server nor the UI enforce or even surface a restriction today.

This is already tracked publicly: [michelangelo-ai/michelangelo#2155](https://github.com/michelangelo-ai/michelangelo/issues/2155)
("Run Trigger / Run Pipeline dialogs: should Production / auto-switch be restricted by source
branch?"), and the codebase carries explicit TODO comments linking to that issue
(`triggerrun/apihook/apihook.go`). Meanwhile, `pipeline/controller.go` already hardcodes
`if branch == "main" || branch == "master"` for a structurally related but separate purpose
(gating `Pipeline.Status.LatestRevision` advancement), showing that the "what counts as a
production branch" question already exists natively but has never been generalized, made
configurable, or connected to the environment-label mechanism.

## Motivation

- **Correctness risk today:** A user can trigger a `production`-labeled run from a feature branch,
  a fork, or any arbitrary ref. There is no way for an operator to prevent this, meaning production
  environment runs can carry untested, unreviewed code with no platform-level guard rail.
- **Duplication and drift:** Production-branch checks have been independently reimplemented in
  multiple places and have drifted out of sync with each other (e.g. one call site recognizing
  only one branch name where another recognizes two). A single, centralized, config-driven
  predicate eliminates this class of drift.
- **Operator demand:** Different deployments use different branch naming conventions. Some use
  `main`, others `master`, others both. The restriction mechanism must be configurable, not
  hardcoded to any single convention.
- **UI gap:** Both Run-creation dialogs (`run-trigger-form.tsx`, `create-pipeline-run-form.tsx`)
  hardcode a fixed two-item environment option list (`development` / `production`) with no
  branch-awareness and no operator configurability -- a gap #2155 explicitly calls out.

## Goals

- Operators can configure, via a single `values.yaml` key, a per-environment allow-list of git
  branch names (e.g. `{production: [main, master]}`).
- The API server enforces the restriction at `PipelineRun` and `TriggerRun` creation (and update)
  time, returning a clear `FailedPrecondition` error when a request violates it.
- The UI's Run Trigger and Run Pipeline dialogs share one source of truth for the restriction and
  disable restricted environment options with an explanatory caption, rather than hiding them or
  deferring to a post-submit error.
- Both the API server and the UI read the same operator-configured value, set once in `values.yaml`
  and fanned out at `helm template`/`helm upgrade` time -- no new runtime cross-service RPC.
- The feature is fully backward compatible: an operator who sets no value sees exactly today's
  behavior (every environment unrestricted, from any branch).
- Operators can optionally configure the ordered list of environment names offered as radio
  options in the two Run-creation dialogs, replacing today's hardcoded `[development, production]`
  pair.

## Non-goals

- **Per-project branch restrictions.** This RFC scopes the restriction to the whole deployment (one
  Helm-configured map for all projects). Per-project overrides are left to a follow-up if real
  demand emerges.
- **Glob or regex branch patterns.** Only exact branch-name matching is supported in this pass.
  GitHub Actions' richer pattern spectrum is acknowledged but deferred.
- **Environment-name validation / enum enforcement.** `api.EnvironmentLabel` remains a fully
  unconstrained free string. This RFC does not introduce an allow-list of legal environment values,
  only a branch restriction for environment values an operator explicitly opts in.
- **`Model` apihook enforcement.** `Model` has no `Spec.Commit`/branch concept of its own -- its
  environment label is either defaulted or inherited from its source `PipelineRun`, which was
  already validated at creation time.
- **A general per-environment config object.** See "Alternatives considered" below.

## High-level architecture

The design introduces three coordinated pieces, all anchored in the existing codebase's own
patterns rather than new infrastructure.

### 1. Shared predicate: `api.IsBranchAllowedForEnvironment`

A new, pure, exported Go function in `go/api/environment.go` (alongside the existing
`EnvironmentLabel` and `UnspecifiedEnvironment` constants it is a natural sibling of):

```go
func IsBranchAllowedForEnvironment(environment, branch string, restrictedBranches map[string][]string) bool
```

The function takes the operator's configured `restrictedBranches` map (keyed by environment name,
not hardcoded to `"production"`) and returns `true` if the branch is allowed for that environment.
An environment name absent from the map, or present with an empty/nil slice, is always
unrestricted -- the map's keys never need to be exhaustive, and absence is the "don't restrict
this one" signal. This is the single predicate every `EnvironmentLabel`-related enforcement call
site must use, centralizing the check that previously drifted across multiple ad-hoc
reimplementations.

The map is keyed by environment name (rather than a flat `[]string` hardcoded to the literal
string `"production"`) because `EnvironmentLabel` is itself an operator-defined free string with
no enum anywhere in `go/api`. Hardcoding `"production"` as the one restrictable value would be
inconsistent with that, and would silently give zero protection to an operator whose
production-equivalent environment is named `prod` or `release`.

**OSS precedent:** GitHub Actions' "deployment branch and tag policies" model exactly this shape
-- a named environment paired with an explicit allow-list of branch patterns, evaluated as a
single yes/no gate before the protected action proceeds. GitHub Actions environments are
themselves operator-named (not drawn from a fixed enum), which is itself precedent for not
hardcoding one privileged environment name in the branch-restriction predicate.

### 2. Config schema: evolving the existing `apihandler.Config`

A new `RestrictedBranches map[string][]string` field is added to the existing
`go/api/handler/config.go`'s `Config` struct -- the same struct that already holds
`PipelineRunDefaultEnvironment`, loaded by the same `go.uber.org/config` `Populate` call, sourced
from the same `apiserver.pipelineRunDefaults` Helm value block, and injected via the same Fx
wiring into all existing apihooks. No new config type, Helm block, or Fx provider is introduced.

The `map[string][]string` type is safe to populate via `go.uber.org/config`'s `Populate` -- the
repo already has a working, tested precedent for `map[string]<T>`-typed fields loaded through the
identical mechanism (`IngesterConfig`'s `ConcurrentReconcilesMap map[string]int` and
`RequeuePeriodMap map[string]time.Duration`).

### 3. UI config surface: Helm-template fan-out (no runtime RPC)

The UI already has a standalone, Helm-templated, operator-configurable settings mechanism:
`ui-configmap.yaml` renders a `config.json` containing today just `{"apiBaseUrl": "..."}`, served
as a static file by the UI's nginx container and fetched once at startup by `getRuntimeConfig()`.
This design extends that existing mechanism with two new fields:

- `restrictedBranches`: rendered from the *same* `values.yaml` key the apiserver reads
  (`.Values.apiserver.pipelineRunDefaults.restrictedBranches`), using a cross-section reference
  -- the exact pattern this template already uses when computing `apiBaseUrl`'s fallback from
  `.Values.envoy.ingress.hosts`.
- `environmentLabels`: an ordered `string[]` of environment names to offer as radio options in the
  two Run-creation dialogs (display-only, no validation semantics). Defaults to
  `["development", "production"]` when unset, matching today's hardcoded pair exactly.

One operator-set value in `values.yaml`, fanned out into two independently-loaded,
service-local configs at `helm upgrade`/render time. No new RPC, no runtime API-to-UI network
call, no cross-service availability coupling.

### 4. API-side enforcement

The two existing `EnvironmentLabel`-defaulting apihooks (`pipelinerun/apihook` and
`triggerrun/apihook`) each gain a `checkRestrictedBranch` call after the pipeline/revision is
resolved and the branch is available. On violation, the call returns a gRPC `FailedPrecondition`
error with a message naming the restricted environment, the configured allowed branches, and the
actual source branch.

`BeforeUpdate` gets the same check (a client could otherwise bypass the create-time check with a
follow-up Update that sets `EnvironmentLabel` directly). When a restricted environment is
requested but no branch data can be resolved (e.g. the owning Pipeline is not found), the check
**fails closed** -- rejects the create rather than silently granting a restricted environment.
This mirrors GitHub Actions' and GitLab's own posture: an unauthorized job fails rather than being
silently skipped.

### 5. UI dialog changes

Both dialogs named in #2155 gain branch-awareness through one shared React context provider
(`EnvironmentPolicyProvider` / `useEnvironmentPolicy`) and one shared utility function
(`isBranchAllowedForEnvironment` in `environment-utils.ts`, mirroring the Go predicate's logic
exactly). The hardcoded two-entry options arrays are replaced by a mapping over the
operator-configured `environmentLabels` list, with each option's `disabled` state driven by
`isBranchAllowedForEnvironment`. Restricted options show a caption explaining which branches are
allowed and which branch the pipeline is currently on -- visible, not hidden, so the restriction
and its reason are always discoverable.

The provider and hook follow the existing dependency-injection pattern (`CoreApp`'s `dependencies`
prop) already used for `service`, `theme`, `user`, and `navigationBar` -- `packages/core` stays
free of any direct dependency on `@michelangelo-ai/rpc`.

## APIs and CRDs

**No new CRDs are introduced.** `PipelineRun`, `TriggerRun`, and `Model` CRDs are unchanged --
this feature is a config + validation-logic addition, not a schema addition to any existing
resource.

**No new proto messages or RPCs are introduced.** An approach that would have required a new
`PlatformConfigService` RPC was considered and rejected (see "Alternatives considered" below).

The only new API-surface artifacts are:

| Artifact | Language | Location |
|---|---|---|
| `IsBranchAllowedForEnvironment` predicate | Go | `go/api/environment.go` |
| `Config.RestrictedBranches` field | Go | `go/api/handler/config.go` |
| `RuntimeConfig.restrictedBranches` field | TypeScript | `javascript/packages/rpc/types.ts` |
| `RuntimeConfig.environmentLabels` field | TypeScript | `javascript/packages/rpc/types.ts` |

### Helm `values.yaml` keys

Two new keys, both under the existing `apiserver.pipelineRunDefaults` block (alongside the
existing `environment` key established by the [PipelineRun environment label defaulting RFC](../20260806-pipelinerun-environment-label-defaulting/20260806-pipelinerun-environment-label-defaulting.md)):

| Key | Type | Default | Description |
|---|---|---|---|
| `apiserver.pipelineRunDefaults.restrictedBranches` | `map[string][]string` | `{}` | Per-environment allow-lists of git branch names, keyed by the `michelangelo/environment` label value the restriction applies to. An environment name absent from this map is unrestricted. Read by both the apiserver's `Config` and the UI's `config.json`. |
| `apiserver.pipelineRunDefaults.environmentLabels` | `[]string` | `["development", "production"]` | Ordered list of environment names offered as radio options in the two Run-creation dialogs. Display-only. Read only by the UI's `config.json`. |

No separate `ui.restrictedBranches` or `ui.environmentLabels` key is introduced -- each value is
set exactly once, preventing operator drift between the two services.

## Alternatives considered

### Alternative A: Generalized per-environment config object

**Considered:** Replace the flat `PipelineRunDefaultEnvironment` string with a map to a richer
struct (`map[string]EnvironmentConfig{ DefaultForNewRuns bool, RestrictedBranches []string, ... }`)
-- generalizing "environment" into a first-class configurable concept with arbitrary
per-environment properties, closer to how GitHub Actions or GitLab model a named "Environment"
object.

**Pros:** Cleaner long-term model if multiple per-environment settings accumulate. Closer to the
external precedent.

**Cons:** Nothing in #2155 or the current operator requests asks for a general per-environment
config object with multiple properties. Building one for hypothetical future per-environment
settings that don't exist yet would be scope creep an OSS reviewer would reasonably push back on.

**Why not chosen:** A minimal, additive change (one new field on the existing `Config` struct) is
more mergeable than a speculative general per-environment-config framework nobody asked for. The
`Config` struct is additive-by-field, so a fuller generalization remains available later,
non-destructively, if real demand materializes.

### Alternative B: CRD-based modeling

**Considered:** Add a new `PlatformConfig` CRD kind, generated via the same `tools/grpc-svc-gen.sh`
pipeline every other resource uses, so `GetPlatformConfig` fits the existing pattern including
List/Update/Delete.

**Pros:** Fits the repo's existing CRUD-resource pattern exactly.

**Cons:** The CRUD generator's pattern assumes a persisted, user-managed, ownerRef-participating
Kubernetes object with real create/update/delete/list semantics. `restrictedBranches` is a
singleton, read-only-to-clients, Helm-ConfigMap-sourced value with none of that. Modeling it as a
CRD would mean either a fake "resource" nobody is meant to `kubectl create`/`delete` (confusing
and a maintenance burden), or building real CRD storage and then having to keep it in sync with
the Helm value anyway.

**Why not chosen:** Do not force a singleton, Helm-sourced config value through machinery built
for persisted, user-managed resources.

### Alternative C: UI-only enforcement (no API-side check)

**Considered:** Since the user-visible symptom in #2155 is entirely UI-side, only fix the two
dialogs and skip API-side enforcement.

**Pros:** Smallest possible diff; no Go changes at all.

**Cons:** Every enforcement-oriented external precedent surveyed (GitHub Actions, GitLab) enforces
*before the protected action executes*, not merely via a UI hint, specifically because a UI-only
guard is trivially bypassed by any direct API/gRPC/CLI caller. A UI-only fix would still allow
unrestricted-by-branch `production` runs via the API -- exactly the gap #2155 describes, just one
layer removed.

**Why not chosen:** The server-side check is the actual guarantee; the UI disablement is a UX
improvement layered on top of it, not a substitute for it.

## Open questions

- [ ] Should glob/regex branch patterns be supported in `restrictedBranches` values (e.g.
  `release-*`), or is exact-match sufficient for the initial release? GitHub Actions supports
  glob patterns as a first-class feature; this RFC starts with exact-match only but the predicate
  could be extended non-destructively.
- [ ] Should per-project overrides (e.g. project A restricts `production` to `main`, project B to
  `release`) be supported in a follow-up? The current design scopes the restriction to the whole
  deployment. Both `Config` and `RuntimeConfig` can grow a project-keyed variant alongside the
  flat environment-keyed map non-destructively if real multi-project demand emerges.
- [ ] What is the right naming convention for the new `CoreApp` dependency entry
  (`environmentPolicy` vs. `runtimeConfig` vs. another name)? The existing entries
  (`error`/`service`/`theme`/`navigationBar`/`user`) are named after the domain, not the feature,
  so `environmentPolicy` follows that convention, but maintainer preference may differ.

## Rollout strategy

- **Phase 1 (this RFC's scope):** Land the shared Go predicate, the `Config` field, the apihook
  enforcement call sites, the Helm template changes, and the UI dialog updates. Ship behind the
  existing `values.yaml` opt-in: `restrictedBranches` defaults to `{}` (no restriction), so the
  feature is inert until an operator explicitly configures it.
- **Migration path:** There is no existing version of this feature to migrate from -- this is a
  green-field addition. An operator upgrading to a release containing this feature takes no action
  and sees no behavior change. An operator who was previously working around this gap with a
  forked patch to one or more apihooks can delete that patch and instead set
  `apiserver.pipelineRunDefaults.restrictedBranches` in their Helm values.
- **Backward compatibility:** The zero value of every new field is defined to reproduce today's
  behavior exactly. `IsBranchAllowedForEnvironment(env, branch, nil)` returns `true` for every
  `(env, branch)` pair. The UI renders exactly today's two radio options with no disabled state
  when `restrictedBranches` is unset. No existing test, CRD, RPC signature, or stored object is
  changed.
- **Rollback:** All changes are additive. Reverting the code changes restores pre-feature behavior
  with no data migration needed. An operator who has already set `restrictedBranches` in their
  `values.yaml` simply has that key ignored after rollback (Helm values for absent template
  references are silently unused).

## References

- [michelangelo-ai/michelangelo#2155](https://github.com/michelangelo-ai/michelangelo/issues/2155) --
  "Run Trigger / Run Pipeline dialogs: should Production / auto-switch be restricted by source
  branch?" -- the public issue this RFC proposes to close.
- [RFC-20260806-pipelinerun-environment-label-defaulting](../20260806-pipelinerun-environment-label-defaulting/20260806-pipelinerun-environment-label-defaulting.md) --
  related RFC by the same author that established the `apiserver.pipelineRunDefaults.environment`
  Helm value and the `EnvironmentLabel` / `UnspecifiedEnvironment` constants this RFC builds on.
  Both RFCs touch the `environment` label concept; that RFC covers defaulting/propagation, this
  one covers branch-based restriction.
- GitHub Actions -- Deployment branch and tag policies:
  https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment
- GitLab -- Protected environments:
  https://docs.gitlab.com/ee/ci/environments/protected_environments.html
