# RFC-20260806-pipelinerun-environment-label-defaulting: Default and Propagate the PipelineRun Environment Label

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-08-06
- **Internal ERD:** N/A

---

## Problem statement

`PipelineRun` objects carry a `pipelinerun.michelangelo/environment` label (defined in
`go/worker/workflows/trigger/cron_trigger_workflows.go:48`) that downstream consumers already
rely on for metrics/attribution (`go/components/pipelinerun/controller.go:633-642`'s
`getEnvironment()`, which falls back to `"unknown"` when the label is absent) and for
schedule-drift detection (`go/components/triggerrun/schedule_input.go:58-72`). Today, however:

- `PipelineRun`s created directly through the API (not via a scheduled `TriggerRun`) never get
  this label set at all — callers must remember to set it themselves, and most don't, hence the
  `"unknown"` fallback in production usage.
- The one place propagation *does* happen — `generatePipelineRunRequest` in
  `cron_trigger_workflows.go:299-313`, when a scheduled `TriggerRun` fires a new `PipelineRun` —
  hardcodes `"production"` as its fallback default, inline, with no shared source of truth.
- The label key itself is defined as two independent string literals
  (`EnvironmentLabel` in `cron_trigger_workflows.go:48` and `scheduleInputEnvironmentLabel` in
  `schedule_input.go:13`), which can silently drift.
- Nothing documents this label's meaning, default value, or override mechanism today.

## Motivation

Any user or dashboard that filters, alerts on, or attributes cost by this label is working with
incomplete data whenever a `PipelineRun` is created directly through the API — a large fraction of
runs simply have no value and fall back to `"unknown"`. As more of the platform (cost reporting,
environment-scoped policy, promotion workflows) comes to depend on knowing a run's environment,
this gap becomes a correctness problem, not just a cosmetic one. Solving it now, before more
features are built on top of an inconsistently-labeled resource, avoids compounding the problem.

## Goals

- `PipelineRun`s created without an explicit environment label receive a sensible, configurable
  default at creation time, regardless of whether they were created directly via the API or fired
  by a scheduled `TriggerRun`.
- When a new `PipelineRun` is regenerated from a parent resource that already carries the label
  (e.g. a scheduled re-fire), the value is propagated forward rather than re-derived.
- The environment-label key and default value are defined in exactly one place, imported by both
  `go/api` and `go/worker`.
- The change is visible: any time a default is injected, it is discoverable via a logged event or
  the object's own labels — never silent, unexplainable state.

## Non-goals

- Adding a typed `environment` field to the `PipelineRun` proto/CRD schema — the label remains a
  free-form key on the standard `ObjectMeta.Labels` map (`proto/api/v2/pipeline_run.proto:696-697`
  requires no changes for this RFC).
- Widening the label to other resource kinds (`Project`, `Pipeline`, `TriggerRun`) in this RFC.
  The naming/scoping question below is flagged as an open question, but resolving it is out of
  scope here to keep this change reviewable.
- Any policy enforcement based on the label's value (e.g. blocking promotion between
  environments) — this RFC only covers defaulting and propagation of the label's value.

## High-level architecture

Two independent code paths need the same defaulting/propagation logic:

```
Direct API create:
  client → go/api/handler.Create() → [NEW] setDefaultEnvironmentLabel(obj) → persist

Scheduled trigger fire:
  cron trigger → generatePipelineRunRequest() → [REFACTORED] defaultOrPropagateEnvironmentLabel(triggerRun.Labels, newLabels) → CreatePipelineRun
```

Both paths call into one shared helper and one shared constant, instead of the current situation
where the API path has no defaulting at all and the worker path inlines its own hardcoded
fallback.

## APIs and CRDs

No proto/CRD schema changes. This RFC proposes Go-level changes only:

- A single canonical `EnvironmentLabel` constant, promoted out of
  `go/worker/workflows/trigger/cron_trigger_workflows.go:48` into a shared package (e.g. `go/api`
  or `go/base`) so both `go/api/handler` and `go/worker` reference the same key. The duplicate
  local constant in `go/components/triggerrun/schedule_input.go:13` is removed in favor of this
  shared constant.
- A new `setDefaultEnvironmentLabel(obj client.Object)` helper in `go/api/handler/handler.go`,
  following the shape of the existing `setUpdateTimestamp` helper (`handler.go:485-497`) but with
  **set-if-absent** semantics (never overwrites an explicit value) and gated to `PipelineRun`
  objects specifically, called from `Create` (`handler.go:93`) alongside `setUpdateTimestamp`.
- A shared `defaultOrPropagateEnvironmentLabel(src, dst map[string]string)` helper, extracted from
  the existing inline logic in `generatePipelineRunRequest`
  (`cron_trigger_workflows.go:309-313`), reused by any other trigger re-fire path (e.g. backfill)
  that creates a new `PipelineRun` from a parent that may carry the label.
- The default value is **operator-configurable**, not a bare string literal or compile-time
  constant — see the new "Default value configuration" subsection below. Different deployments of
  this platform have already expressed wanting different out-of-the-box defaults, which rules out a
  single hardcoded real-environment value for everyone.

### Default value configuration

Add an `apiserver.pipelineRunDefaults.environment` Helm value (default unset), rendered into the
existing apiserver and worker ConfigMaps at install time — following the same
Helm-value-to-ConfigMap-to-Go-struct pattern already used for other operator settings (e.g. the
`Config` struct populated via `provider.Get(configKey).Populate(&conf)` in
`go/controllermgr/config.go:7-25`). Both `go/api/handler` (create-time defaulting) and
`go/worker` (trigger re-fire propagation) read the same rendered value via this pattern, injected
at process startup rather than read from a package-level global, so `setDefaultEnvironmentLabel`
and `defaultOrPropagateEnvironmentLabel` both take the configured default as a parameter.

If the operator sets no value, the packaged Helm chart ships a default-of-defaults of
`"unspecified"` — deliberately **not** a guessed real environment value (not `"development"`, not
`"production"`), and deliberately **not** `"unknown"` either. Those are two different states:

- `"unknown"` already means, throughout this codebase, "the system could not determine a value" —
  `getEnvironment()` (`go/components/pipelinerun/controller.go:635-641`) and every sibling
  extractor in that file return it as a read-time fallback for genuinely missing/legacy data
  (e.g. a `PipelineRun` created before this label existed). That meaning is unchanged by this RFC.
- `"unspecified"` means something different: the new defaulting/propagation logic *did* run, and
  *did* produce a value — the operator simply hasn't configured a real one yet. Reusing
  `"unknown"` for this would conflate "we don't know" with "nobody's configured it," which muddies
  both signals for anyone querying or dashboarding on this field later — a distinction called out
  during RFC review.

Two properties make `"unspecified"` the right zero-config default for the *new* write path:

1. **It keeps every `PipelineRun` groupable/filterable for metrics and dashboards** — the original
   problem this RFC set out to fix — without the correctness risk of guessing a real environment.
   An unlabeled run silently defaulting to `"production"` risks being swept into
   production-scoped cost/policy/promotion logic it was never meant to be part of; guessing
   `"development"` is safer but is still a guess. `"unspecified"` makes "no configured default"
   a first-class, queryable state, distinct from both a real environment and from "unknown."
2. **It keeps the existing `"unknown"` read-time fallback meaningful.** Pre-existing objects (or
   any future path that doesn't route through this new defaulting logic) still surface as
   `"unknown"` via `getEnvironment()`, unchanged — so `"unknown"` in a dashboard continues to mean
   what it always meant, rather than being diluted by every zero-config install's new writes.

This RFC scopes the configuration to a single, global, per-deployment value (one Helm value per
install), not a per-namespace or per-project override. The motivating scenario — different
deployments wanting different defaults — is fully solved by a per-install Helm value with no
in-cluster override needed. Per-namespace override would require a new lookup path (namespace-
scoped ConfigMap or CRD field) with its own precedence rules, which is a materially larger scope
than "make the constant configurable" and is left to a follow-up RFC if a single-cluster,
multi-tenant need for differing defaults emerges.

## Alternatives considered

### Alternative A: Kubernetes-style mutating admission webhook

**Pros:** Matches the community-consensus pattern used by Kubernetes generally and, per prior-art
research, is the mechanism most OSS operators will recognize; naturally re-applies on every
creation path without each call site needing to remember to invoke it; centralizes defaulting
completely outside business logic.

**Cons:** Michelangelo's `go/api/handler` already does inline, in-process mutation for comparable
concerns (`setUpdateTimestamp`) rather than using the k8s admission-webhook mechanism; introducing
a new webhook for a single label would add operational complexity (webhook deployment,
certificate management, failure-mode handling) disproportionate to the size of this feature.

**Why not chosen:** Given the existing `setUpdateTimestamp` precedent already lives inline in the
handler, extending that established pattern is more consistent with the codebase and avoids
introducing a new infrastructure component for this specific gap. A webhook-based approach could
be revisited if more defaulting/policy needs accumulate.

### Alternative B: Leave the `"production"` vs. no-default inconsistency as-is, document only

**Pros:** Zero code risk; ships as a docs-only PR.

**Cons:** Does not fix the underlying data-quality problem — `PipelineRun`s created directly via
the API would still have no environment label, and the `"unknown"` fallback would remain the norm
rather than the exception. Documentation alone does not change existing behavior for the
majority of affected callers.

**Why not chosen:** The problem statement is a functional gap, not merely an undocumented one;
documentation is necessary but not sufficient (see the DX checklist below, which requires
documentation as one of several changes, not a substitute for the code fix).

## Open questions

- [x] ~~What is the right default value, and where does an operator configure it?~~ **Resolved:**
  a single global `apiserver.pipelineRunDefaults.environment` Helm value per deployment; if unset,
  the zero-config default is the sentinel `"unspecified"` — deliberately distinct from the
  pre-existing `"unknown"` read-time fallback (`getEnvironment()`), which still means "the system
  couldn't determine a value" and is unchanged by this RFC. See "Default value configuration"
  above. Per-namespace/per-project override is explicitly deferred to a follow-up RFC.
- [ ] Should the label be widened beyond `PipelineRun` (e.g. to `Project`/`Pipeline`/`TriggerRun`)
  before or after this change ships? Building defaulting on a `PipelineRun`-scoped label now makes
  a later widening a breaking rename.
- [ ] What is the opt-out mechanism for a caller that intentionally wants no environment label
  (as distinct from "forgot to set it")?
- [ ] Does an already-defaulted label that a user later sets explicitly count as configuration
  drift for the purposes of `scheduleInputHash` (`schedule_input.go:19-32`), or should defaulted
  values be excluded from drift comparisons?
- [ ] Is a logged/eventable signal (e.g. a Kubernetes Event) sufficient visibility when a default
  is injected, or does this warrant a status field on `PipelineRun` itself?

## Rollout strategy

- **Phase 1 (this RFC's scope):** land the shared constant, the create-time defaulting helper, and
  the refactored propagation helper, behind table-driven tests matching the conventions in
  `go/components/triggerrun/schedule_input_test.go`. Existing `PipelineRun`s created before this
  change are unaffected — no backfill/migration job is required.
- **Migration path — real, called-out behavior change on one path:** the trigger re-fire path
  (`generatePipelineRunRequest` in `cron_trigger_workflows.go:309-313`) currently hardcodes
  `"production"` as its fallback when the firing object has no label. After this change, an
  unconfigured install's trigger-fired `PipelineRun`s will carry `"unspecified"` instead of
  `"production"` in that same no-label case. This should be called out explicitly in release
  notes: any dashboard/alert/query that assumed unlabeled trigger-fired runs were `"production"`
  needs to either configure `apiserver.pipelineRunDefaults.environment: "production"` to preserve
  today's behavior, or update its filters to account for `"unspecified"`. The direct-API-create
  path has no equivalent prior default, so its behavior there is purely additive. Note that
  `"unspecified"` is a new value, distinct from the pre-existing `"unknown"` read-time fallback —
  queries that already filter on `"unknown"` are unaffected by this change.
- **Rollback:** the defaulting/propagation helpers are additive and can be reverted independently
  of the `PipelineRun` create path itself; no data migration is introduced, so rollback is a
  straightforward code revert.

## References

- Prior art comparison (Argo Workflows, Kubeflow Pipelines, Flyte, Kubernetes admission webhooks)
  informing this design is available on request from the design discussion that produced this RFC.
- Argo Workflows — Template Defaults: https://argo-workflows.readthedocs.io/en/latest/template-defaults/
- Argo Workflows v3.5 release notes (creator-label propagation): https://terrytangyuan.github.io/2023/08/14/argo-workflows-v3.5/
- Flyte — open feature request for default launch-plan labels: https://github.com/flyteorg/flyte/issues/5774
- Kubernetes — Admission Webhook Good Practices: https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/
