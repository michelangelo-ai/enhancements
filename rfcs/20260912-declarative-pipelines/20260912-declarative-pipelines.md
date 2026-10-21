# RFC-20260912-declarative-pipelines: Declarative pipelines for Michelangelo

- **Status:** Draft
- **Author(s):** @onimsha
- **Created:** 2026-09-12
- **Internal ERD:** n/a (external contribution)


This RFC proposes a declarative authoring path for Michelangelo pipelines: a validated task-graph specification stored as data, executed by one version-pinned platform interpreter, and extended through a namespaced task-type registry. Because the spec is data, any system of record can hold it, and revision control is a first-class deployment pattern rather than an API constraint. It adds no new services, changes no existing manifest type, and adds no inline field to the manifest: the spec travels in the existing `content` slot as a typed `Any`.

You can evaluate this proposal from the body alone. The appendices show worked examples for readers who want to see a specific point made concrete.

## Problem statement

### What exists today

Michelangelo can deliver configuration to a running pipeline, but nothing authors or executes pipelines from that configuration. The delivery plumbing is complete and live:

- The API accepts per-run `workflow_config` and `task_configs` (`parameter.proto:39-42`).
- The trigger system generates them for scheduled runs (`cron_trigger_workflows.go:367-369`).
- The run path forwards them to the workflow as positional arguments and sets `YAML_BASED_PIPELINE=true` (`executeworkflow.go:561-577`).
- A validated task-config schema exists in the Python tree (`canvas/schema/v2alpha1/config.py`).

What's absent is both ends of the path: nothing in open source authors those configurations, and no workflow exists to receive them. Uber's recipients live in its monorepo as `workflow.defs.*`. The user documentation tells low-code users to write a `pipeline_conf.yaml` against that internal library. An adopter who follows the documentation cannot complete the setup.

### Why the gap matters

- **The documented path doesn't run outside Uber.** The asset it describes was never open-sourced.
- **Adopters fill the gap with per-pipeline code generation, and it fails structurally.** We built one: an external service that compiled declarative task graphs into generated workflow code. It worked in production and is now being retired. The failure modes were inherent, not incidental: unpinned vendoring of private runtime files, reverse-engineered contracts that break silently, string-quoting bugs manufactured by compiling user data into program text, and permanent "not supported" stubs on the features users request most.
- **Fixed workflow templates can't express user-defined DAGs.** The evidenced demand — in our production traffic and in RFC `20260428-unified-ml-data-pipelines` — is arbitrary graphs over user-chosen tasks.

The work required is not a YAML converter. It is a validated contract for pipeline topology, owned by the control plane and executed by one platform program, so that authoring surfaces can multiply without any of them compiling code.

## Motivation

Production ML platforms need a declarative surface: consoles that author pipelines by form, CI that generates them from templates, teams whose users don't write Python. Because OSS Michelangelo offers only the SDK path, every adopter who needs that surface builds a private translation layer against platform internals and maintains it forever.

If this lands:

- The pipeline experience the docs already describe becomes real outside Uber.
- Platform fixes ship as one new interpreter version that pipelines adopt explicitly, instead of a regeneration campaign over compiled artifacts.
- Any console, CLI, or scheduler authors pipelines by emitting the spec. Deployments register their own task types without forking.

This is a scoped increment of RFC `20260428-unified-ml-data-pipelines`, Phase 2 (Workflow IR), whose run-side plumbing is the code cited above. We — Grab's ML platform team, a production adopter — offer to implement, test, document, and co-maintain the whole of it.

## Goals

- Make the configuration-as-data path executable for every adopter, with no dependency on internal code.
- Add one manifest type that carries a validated, versioned task-graph spec: tasks, typed configs, resources, dependencies with outcome conditions, retries, and parameters with per-run overrides.
- Execute every declarative pipeline with one platform-owned interpreter workflow, version-pinned per pipeline, on the unmodified runtime. No per-pipeline code generation anywhere.
- Reject invalid pipelines at creation with precise errors. Show authors an explain-plan: a dry-run response that shows the resolved execution levels, parameters, and pinned interpreter version before anything runs.
- Pin execution semantics with a conformance suite: declarative and SDK pipelines behave identically over the features both paths express.
- Provide a namespaced task-type registry so deployments register custom types without forking.
- Keep observability parity: declarative runs report the same per-task states as SDK runs.
- Keep the spec round-trippable. The stored spec is retrievable as data, so an external system of record such as Git can hold it, and pipelines authored in a console can be exported to it. Every pipeline gets the same validation and features regardless of where its spec came from.

## Non-goals

- **An authoring UI.** This enables one; it isn't one.
- **New services or deployment components.**
- **Any change to the SDK path, the transpiler, or existing manifest types.**
- **Matching the SDK's full expressiveness.** Declarative v1 deliberately omits, relative to the SDK:

  | Capability | SDK | Declarative v1 |
  |---|---|---|
  | Static DAG, fan-in/fan-out, per-task retry/cache/resources | Yes | Yes — the conformance target |
  | Run a task on upstream *failure* | No (the transpiler rejects `try`, `build.py:627-632`) | Yes — new capability, also offered back to SDK authors |
  | Branch on a task's output value | Yes (`if result["auc"] > 0.85:`) | No — deferred; see below |
  | Loops and runtime fan-out | Yes | No; the interpreter can add them later without a format break |
  | Future handles, racing, `.result()` | Yes | No; the interpreter owns scheduling |
  | Sub-workflows, cross-pipeline triggers, durable sleep | Yes | No |

  This table is the v1 conformance scope: the conformance suite asserts identical behavior over the capabilities both paths support, and asserts the declarative-only rows against their stated semantics.
- **Value-conditional edges, in v1.** An edge that gates on a task's output value — for example, run `promote` only when `auc >= 0.85` — is deferred. Outcome gating (`when: SUCCEEDED | FAILED | COMPLETED`) covers failure handling, and a value-predicate surface deserves its own design review: typing and totality rules, and the pressure to grow into an expression language. Because edges are data, a later `condition` field on `TaskDependency` is an additive change, not a format break. Until then, value gates live inside the task's own code.
- **Migrating any existing workflow, or breaking any existing consumer.**

## High-level architecture

### End state

Replace per-client compilation with a data contract plus a platform interpreter:

```
DEFINITION PLANE               CONTROL PLANE                  EXECUTION PLANE
────────────────               ─────────────                  ───────────────
YAML/JSON task-graph spec  →   validate: schema, cycles,  →   one interpreter workflow
authored by any client:        references, task types         (version-pinned per pipeline)
CLI, console, CI               ↓                              walks the graph on the
no compile step                explain-plan to author         unmodified runtime,
no codegen                     ↓                              dispatching each task to the
                               store spec + pin               same implementations SDK
                               interpreter version            pipelines use
```

Four responsibilities, each with one owner:

| Responsibility | Owner | What it is |
|---|---|---|
| Specification | Author (any client) | A task graph as pure data. Zero execution code, by construction. |
| Enforcement | Apiserver | Creation-time validation (strict schema, graph well-formedness, reference resolution, task-type registration) plus an explain-plan. The controller does not validate. |
| Execution | Platform | One interpreter workflow walks any valid graph, dispatching to the same task implementations SDK pipelines use. Versioned like any component. |
| Extension | Deployments | Task types as namespaced registry entries: schema plus implementation. `core/*` upstream, vendor namespaces by adopters. |

Four properties define success. Each one closes a failure of the prior art:

1. No user-authored or user-derived program text enters the orchestration trust domain: the control plane, the interpreter, and the worker. The existing task-launch seam — where a factory builds the command line that starts the user's own code, inside the user's own task pod — is pre-existing runtime behavior and is unchanged by this proposal.
2. Every declared field is validated and honored, or the pipeline is rejected at creation. No declared-but-ignored configuration.
3. Execution behavior is a versioned component with sticky per-pipeline pins. A fix is one release adopted deliberately, never a mutation of stored artifacts.
4. Semantics are pinned by conformance tests against equivalent SDK pipelines, over the subset both express.

The interpreter follows the existing architecture. The worker already registers exactly one workflow type, `starlark-workflow`, whose entry point takes the pipeline as data (`service.go:24`). "One program, N pipelines as data" is how the system works today; this RFC moves the varying data one level up, from a tar of transpiled Starlark to a task graph.

### How execution works

1. **Create.** The apiserver unpacks the spec from `manifest.content`, validates it against the schema set of the interpreter version being pinned, then writes a resolved pin — version, artifact URL, digest — onto the pipeline record.
2. **Run.** The pipeline-run controller fetches the pinned interpreter tar (the same `blobStore.Get` call it uses for SDK tars today), verifies the digest, and starts `starlark-workflow` with the tar as input and the task graph as arguments.
3. **Execute.** The interpreter resolves the graph into levels and dispatches each task through its registered factory — the task type's Starlark entry function, defined in "How extension works." Tasks report progress through the same `report_progress` protocol SDK pipelines use. The worker is unmodified and cannot distinguish the two paths.

The interpreter's input is derived server-side from the stored, validated spec — never from raw run-request structures. Per-run overrides are re-validated at run submission: `workflow_config` against its typed message, parameter overrides against declared parameters only. A run request's raw `task_configs` never reach a declarative pipeline's interpreter, so the existing unvalidated per-run channel cannot bypass creation-time checks.

The interpreter must stay replay-deterministic. It iterates the task map in sorted name order, reads time only through `workflow.Now` (surfaced as `{{ pipeline.execution_time }}`), and performs all I/O through activities. The Phase 3 replay-determinism suite asserts these obligations.

Appendix A shows a worked spec and its explain-plan. Appendix C shows the artifacts step 1 and 2 consume.

### Execution semantics

These rules are normative. Appendix A.2 shows them applied to a worked example.

- **Gate evaluation.** An incoming edge is satisfied when the dependency task's terminal state matches the edge's `when`. `COMPLETED` means `SUCCEEDED` or `FAILED`.
- **Skip propagation.** A task whose gates aren't all satisfied is skipped. A skipped or killed dependency satisfies no gate — not `FAILED`, not `COMPLETED` — so its dependents are skipped in turn.
- **Run outcome.** A skip is not a failure: a run whose only non-successes are skips ends `SUCCEEDED`. A `when: FAILED` handler runs for cleanup and reporting; it never rewrites the run outcome, which stays `FAILED`. The exact step-state mapping for the run view is an open question.
- **Retry.** A task's own `retry` replaces `workflow_config.default_retry` wholesale. There is no field-level merge.

### Source of truth and revision control

Many adopters run an infrastructure-as-code policy: every production pipeline definition lives in Git, is reviewed there, and is debugged from there. The API supports that policy without depending on it.

**The spec is round-trippable data.** The apiserver stores the validated spec on the pipeline record and returns it from the pipeline read API in canonical form. A client can read a pipeline authored in a console and commit the result. The CRD already anticipates this: `PipelineSpec.manifest` is documented as the definition in a Git repo, used to reconcile code-driven and UI-driven mutations. The SDK path cannot offer this: Python to Starlark tar is one-way. The declarative path is therefore the first Michelangelo authoring path that can satisfy an infrastructure-as-code policy for pipelines created from a form.

**Recommended flow: Git holds the spec, the API stores the validated copy.** The spec lives in a repository. CI runs `ma pipeline apply`, which reads the file and calls create or update with the spec inline. This is the existing `mactl` apply path (`plugins/entity/pipeline/apply.py`), extended to the new manifest type. The API is the contract; revision control is a layer above it. This is the Kubernetes model: the kube-apiserver never fetches from Git, and GitOps tooling pushes to it. It needs no API change.

**Deferred flow: create by reference.** A `declarative_ref` field, an object address plus a mandatory SHA-256 digest that the apiserver resolves into `content` at create, is a natural extension and is deferred from v1. The reason is the validator's location. Validation runs in the apiserver, and the apiserver has no blob-store client today: `blobstore.Module` is linked by the controller manager and the worker, not by `go/cmd/apiserver`, and `external_storage` fields go to metadata storage, not to S3. Resolving a reference in the apiserver would add a blob-store dependency and credentials to the API tier. The inline path needs none of that, and it already serves a Git-held or S3-held spec: `ma pipeline apply` reads the object, and the apiserver validates what it receives. If the field is added later, the rules are fixed now: digest mandatory, resolved once at create, `content` and the ref mutually exclusive, the ref never dereferenced at run time. Field 11 on `PipelineManifest` is reserved for it.

**Provenance already exists in the CRD.** `PipelineSpec` carries `commit` (`CommitInfo`: `git_ref` and `branch`) and `manifest.file_path`, and the `manifest` field is documented as "the pipeline definition in a GIT repo, used for reconciliation between code and UI driven mutations" (`pipeline.proto:173-176`). When `spec.commit` is set and revisioning is enabled, the pipeline controller snapshots a `Revision` with `source: Git` on every ready reconcile, and only `main` or `master` commits advance `status.latestRevision` (`pipeline/controller.go:118-128, 209-247`). This RFC adds no provenance field. `spec.commit` is client-stamped metadata: the apiserver never checks that the ref exists, in this path or the SDK path. The applier sets it, not the spec file: a file cannot hold its own commit SHA, so the CLI reads it from the checkout at apply time. The CLI stamps the full commit SHA, never a branch name. A branch name such as `main` names one mutable `Revision` that each reconcile overwrites, which defeats the history it is meant to keep. `mactl` already works this way: it overwrites whatever the file says with `HEAD`'s SHA and the active branch, and sets `manifest.file_path` to the repo-relative path of the applied file (`mactl/plugins/entity/pipeline/create.py:387-398`). Neither field has a validation rule beyond non-empty, and no Go component reads `file_path`. Both are provenance labels; the declarative CLI keeps that meaning. `export` omits `spec.commit`, so an exported file matches the committed file byte for byte. The explain-plan echoes `spec.commit` when present.

**What the API does not enforce.** The API cannot prove that a Git commit exists, and it should not try. A deployment that requires revision control enforces it as policy in the existing API hook. The hook rejects creates whose caller is not the CI identity, or whose `spec.commit` is absent. An adopter that runs console-only or stores specs in S3 uses the same API with no such policy. Both get identical validation.

### How extension works

A task type is three artifacts across three layers. Each layer's composition seam already exists and is keyed by a plain string — no enum, no `oneof`:

| Layer | Adopter writes | Binds at |
|---|---|---|
| Worker (Go) | a `service.IPlugin`: an ID plus a Starlark module of builtins | compile time, into the adopter's own worker binary |
| Workflow (Starlark) | a `task.star` factory calling those builtins | assembly time (declarative) or transpile time (SDK) |
| Authoring | a JSON Schema for `config`/`job_specs`; optionally a `TaskConfig` dataclass for SDK users | create time / import time |

A one-file **descriptor**, shipped in the vendor's package next to its factory, ties the layers together: registry key, plugin ID, factory location, schemas, and capabilities. An assembly tool compiles all descriptors into the two runtime artifacts — the interpreter tar and the apiserver's validation config — and stamps both with the descriptor-set digest. The two artifacts ship on independent timelines, the tar from CI and the config from GitOps. The apiserver never opens the tar, so skew between them is caught by the run controller: it fetches the pinned tar, compares the tar's `meta.json` digest with the pinned digest, and fails the run before `StartWorkflow` with an error naming both. The release sequence in Appendix C.4 keeps that check from firing in practice. Assembly fails on duplicate task-type keys, naming both descriptors, and rejects non-core packages that declare `core/*` keys. A descriptor without its schemas fails to register: without the schema half, creation-time validation can't cover that type.

One gap the digest can't check: the worker binary must link the plugins that the tar's factories load. A mismatch fails at workflow start with an error naming the missing plugin ID. Deployments prevent it by construction — the descriptor set that drives assembly is also the list of plugin IDs the worker binary must register, so an interpreter release and its worker binary ship from the same descriptor set and the same CI.

Factories follow a four-rule calling convention, checked by the conformance suite:

1. **Two-stage shape.** `factory(configuration) -> callable(task arguments)`.
2. **Progress protocol.** Every state transition calls `report_progress`. Michelangelo queries the live workflow for task state; a factory that never calls `report_progress` does not appear in the run view at all.
3. **Retry ownership.** The factory owns its retry loop. The interpreter never wraps an outer one.
4. **Outcome mode.** Opt-in `raise_on_failure=False` returns an outcome instead of failing the workflow, which is what makes `when: FAILED` edges possible. A type that doesn't declare the `outcome_mode` capability can only be depended on with `when: SUCCEEDED`; the apiserver enforces this at creation.

The conformance suite also checks that a type's registered schema and its factory's configuration parameters agree, by generating a factory call from the schema.

This shape runs in production in Grab's deployment: a second Ray implementation registered beside upstream's `ray` plugin, in the same worker binary, with the upstream plugin unmodified. Appendix B is a mock vendor package with these rules marked in the code.

### Changes required

We volunteer the implementation for all items; upstream's cost is review and ownership.

**New code (Phases 1–4):**

1. The spec schema, manifest type, creation-time validation, and explain-plan.
2. The interpreter workflow and its conformance, golden-graph, and replay-determinism suites.
3. The assembly tool: compose the interpreter and vendor factory packages into one immutable tar plus the matching apiserver validation config, both stamped with the descriptor-set digest. Assembly fails on duplicate task-type keys; the run controller fails a run whose tar digest disagrees with the pin, naming both digests.
4. The run-path branch that fetches the pinned tar and forwards the graph as arguments.

**Changes to existing code (small, each independently useful):**

5. **Outcome mode** in the shipped factories: opt-in `raise_on_failure=False`, 30–60 lines per factory, default behavior unchanged. The only change to shared runtime code in this proposal.
6. **An aliased load root** so shared Starlark helpers (`commons.star`) are loadable out-of-tree: `load("@michelangelo//commons.star", …)`, resolved against the pinned interpreter version. Today the helpers are reachable only by relative path, which forces vendors to keep unpinned private copies — a hard prerequisite for out-of-tree task types.
7. **Duplicate plugin ID as a boot error.** Registration is last-write-wins today, so a vendor ID can silently shadow a core plugin.

**Documentation and process:**

8. Declare the extension seams stable and versioned: `IPlugin` registration, the factory calling convention, the progress protocol, `TaskBinding`. Publish the conformance suite (the `service.TestSuite` harness already exists internally).
9. Document "build your own worker binary" as the supported extension path. Without this documentation, adopters conclude that they must fork.

## APIs and CRDs

Additive surface changes only:

- **One enum value.** `PipelineManifest.Type` gains `PIPELINE_MANIFEST_TYPE_DECLARATIVE`.
- **One new proto file** with six messages: `DeclarativeWorkflow`, `WorkflowSettings`, `ParameterSpec`, `DeclarativeTask`, `TaskDependency`, and `RetryPolicy`. `DeclarativeTask` is deliberately the Canvas `TaskConfig` shape plus `depends_on` and `retry`, under the enclosing `DeclarativeWorkflow.schema_version`. Provenance uses the existing `spec.commit` and `manifest.file_path`.
- **No new inline field on the manifest.** A declarative pipeline carries its `DeclarativeWorkflow` in the existing `PipelineManifest.content` field as a typed `Any`, the Envoy `TypedStruct` pattern the field's own comment recommends. The field is already shared in practice: `mactl` writes it for `UNIFLOW` pipelines too (`create.py:439-459`), and the run path decodes it without consulting `manifest.type` (`executeworkflow.go:625`). The comment that limits `content` to the YAML type is stale and is corrected in the same proto change. `external_storage` on `content` already gives the behavior a large graph wants: the list API omits the body, the get API returns it.
- **No by-reference field in v1.** `PipelineManifest` field 11 is reserved for a future `declarative_ref`; see "Deferred flow" above.
- **One dry-run response shape** for the explain-plan.
- **One validation extension and one API hook**, at existing injection points (`pipeline.pb.validation.go:153-155`; precedent `project/apihook/apihook.go:44-55`).
- **One typed decode branch and one `switch manifest.Type` branch** in the run path — roughly 50–80 lines plus table tests. The switch must run before the `content` decode in `getWorkflowInputs`: that decode unpacks `content` as `TypedStruct` unconditionally and fails on a `DeclarativeWorkflow` payload. The declarative branch bypasses only that decode; it keeps the env setup in the same function (`MA_NAMESPACE`, `MA_PIPELINE_RUN_NAME`, `STARLARK_TIME`, task image), because `STARLARK_TIME` is what `workflow.Now` and the replay-determinism obligation depend on. No Go code dispatches on `manifest.type` today, so this switch is the first.

Validation runs at creation, in a fixed order so the first error is the most specific one:

1. Unpack `content`. When `manifest.type` is `DECLARATIVE`, the `Any` type URL must be `type.googleapis.com/michelangelo.api.v2.DeclarativeWorkflow`; any other URL rejects. Strict decode; unknown fields reject.
2. Schema version.
3. Task names (charset `^[a-z][a-z0-9_]{0,62}$`) and count (500 or fewer in v1, configurable — the bound keeps workflow history and `task_progress` payloads finite).
4. Task-type resolution, with capabilities covering each task's *effective* retry — its own or the inherited default.
5. Per-type config schemas, strictly.
6. Graph well-formedness: no cycles, no duplicate edges, no self-edges.
7. Reference reachability.
8. Parameter coherence.
9. Write the interpreter pin.
10. Store the resolved spec in `content` on the pipeline record. If `spec.commit` is set, the existing pipeline controller snapshots a `Revision`; this RFC adds no code there.

**Who validates: the apiserver, in two layers. The controller validates nothing.**

- **Shape checks (steps 1 to 3)** run in the generated `Validate` path, through the `RegisterPipelineManifestValidateExt` extension point (`pipeline.pb.validation.go:153`). The handler calls `Validate` on create, update, and patch (`go/api/handler/handler.go:80,157,210`).
- **Semantic checks (steps 4 to 10)** run in a Pipeline API hook's `BeforeCreate` and `BeforeUpdate`, which the generated YARPC handler calls before the request reaches Kubernetes. Pipeline has no hook today; `project/apihook/apihook.go:44-55` is the precedent. Registry lookup and the pin write need the validation config, which the apiserver already loads. The apiserver reads nothing from the blob store: the pin's digest comes from the validation config, and the run controller verifies the tar against it.
- **The pipeline controller stays a status stamper.** Today it marks every Pipeline `READY` without reading the manifest (`pipeline/controller.go:129`). This RFC adds one branch: a `DECLARATIVE` pipeline with no interpreter pin goes to `PIPELINE_STATE_ERROR` with an `error_message` that says to re-apply through the API. It does not re-run the validator.

**The write path the apiserver does not see.** A Pipeline CR written straight to Kubernetes, with `kubectl apply`, bypasses both layers. Michelangelo runs no validating admission webhook; its only webhook is the version-conversion handler at `/convert` (`go/api/webhook/webhook.go`), and the CRD schema renders `content` as an opaque blob. This is the existing contract for every Michelangelo resource: the API is the write surface, and direct CR writes are unsupported. The controller branch above makes a bypassed declarative pipeline visible as `ERROR` instead of a first-run failure, which is the same outcome the SDK path has today. A validating webhook that calls the same validator is a possible later addition; it needs a webhook deployment that the Helm chart does not ship today, so it is out of v1 scope.

References use one grammar, resolved at run time and never by text rewriting: `{{ params.<name> }}`, `{{ tasks.<name>.output[.<field>] }}`, and `{{ pipeline.execution_time }}`. Resolution is single-pass and non-recursive. Every task reference must name a transitive dependency of the referencing task, checked at creation.

Appendix A carries the proposed proto, field semantics, and a worked example.

## Alternatives considered

### Alternative A: client-side per-pipeline code generation

The de-facto adopter status quo. No upstream changes, but it inherits every failure listed in the problem statement — we're retiring ours on production evidence.

### Alternative B: server-side per-pipeline code generation

Fixes contract ownership, keeps every compiler failure mode, and adds N stored artifacts that go stale on each platform fix. A bug fix becomes a regeneration campaign.

### Alternative C: open-source the internal standard-workflow library

Welcome, and orthogonal. Standard workflows cover fixed topologies; the demand is user-defined DAGs. Released templates would coexist as pre-baked alternatives to writing a graph.

### Alternative D: leave declarative support in adopter forks

The documented YAML path stays broken in OSS and every serious adopter privately re-learns the same failures. This alternative is today's status quo, and its costs are documented in the problem statement.

### Alternative E: require revision control at the API

Reject any create whose spec does not carry a resolvable Git revision. This satisfies an infrastructure-as-code policy by construction, but it hard-codes one adopter's policy into the upstream contract. Adopters that run console-only or hold specs in an artifact store would need to fork the validation hook. The proposal instead keeps the API agnostic, records provenance, and leaves enforcement to a deployment-side hook. An adopter that wants Alternative E can implement it in that hook without an upstream change.

## Open questions

- [ ] **Relationship to RFC `20260428-unified-ml-data-pipelines`.** Land as its Phase-2 increment (our preference, pending its author) or standalone?
- [ ] **Manifest-type enum value.** Value 2 has been absent since the first open-source commit of `pipeline.proto`, with no `reserved` statement. The string `PIPELINE_MANIFEST_TYPE_ASL` survives in `go/api/api.go:75`, `go/base/notification/types/types.go:38`, and a triggerrun test, and `PipelineRunStatus` once carried `external_link_asl` (commit `a3be9d08`), so value 2 was most likely `ASL`. We propose `reserved 2;` and `DECLARATIVE = 4`. Maintainers confirm.
- [ ] **Pin field.** Packing is settled in v4: typed `Any` in the existing `content` field. Still open: a dedicated pin field on the manifest versus reusing the artifact reference.
- [ ] **Override and reset policy.** We propose per-run config overrides with topology never overridable. Stricter, looser, or finer?
- [ ] **Outcome contract and UI states.** The task return shape for outcome-gated edges, and step-state mapping for skipped, killed, and failed-but-handled tasks.
- [ ] **Task-type namespacing and promotion.** Does `core/*` plus vendor domains fit upstream naming? Is a promoted `core/notebook` welcome as the first post-v1 contribution?
- [ ] **By-reference specs.** Deferred from v1 because the apiserver has no blob-store client. Is reserving field 11 for `declarative_ref` acceptable, or should upstream prefer that references stay a CLI concern permanently?
- [ ] **`commons.star` reachability.** We propose the aliased load root (change 6). The alternative is promoting the helpers to a Go plugin reached via `load("@plugin", …)`. We'll implement whichever is preferred.

## Rollout strategy

The contract lands first. Each phase is independently reviewable and revertable.

| Phase | Delivers | Consumes the new type? | Target release (provisional) |
|---|---|---|---|
| 0 | This RFC: direction check and shepherd. Gate: co-sponsorship from the `20260428` author or a second adopter on record before final comment | — | — |
| 1 | Spec schema, manifest type, creation-time validation, explain-plan | No | v0.12.0 |
| 2 | Outcome mode in factories (change 5) — isolated for focused review | No | v0.12.0 |
| 3 | Interpreter, task-type registry, assembly tool, conformance suites | No | v0.13.0 |
| 4 | Control-plane wiring: semantic validation, pinning, run-path branch, end-to-end tests | Yes | v0.14.0 |
| 5 | CLI support in `mactl`: `ma pipeline apply` accepts the declarative type; a new `export` writes a stored pipeline back to a file. Docs, and a working end-to-end example replacing the broken documented one | Yes | v0.14.0 |

Target releases follow the biweekly minor-release train (v0.10.0 shipped 2026-09-08) and are provisional until the shepherd confirms them. One apiserver flag gates the feature, default off, until Phase 4 soaks. Adoption is opt-in per pipeline. Rollback is turning the flag off; no stored artifact needs rewriting, because nothing was generated.

## References

- Upstream: `github.com/michelangelo-ai/michelangelo` (HEAD `58aae830` at time of study); `michelangelo-ai/enhancements` — RFC process, template, RFC `20260428-unified-ml-data-pipelines`.
- `github.com/cadence-workflow/starlark-worker` v1.0.15 — runtime, plugin registry, Starlark dialect configuration.
- Canvas run-input fields: `proto/api/v2/parameter.proto:39-42`. Canvas task-config schema: `python/michelangelo/canvas/schema/v2alpha1/{config,job_specs}.py`.
- Transpiler: `python/michelangelo/uniflow/core/build.py`. Run path: `go/components/pipelinerun/actors/executeworkflow.go`.
- Full parity tables and evidence citations ship with the Phase 3 conformance suite. The Non-goals table in this document is the v1 conformance scope.

---

## Appendix A: Example — a declarative pipeline

*This appendix is illustrative. You don't need it to evaluate the proposal. Field names are proposals; maintainers assign the final ones.*

### A.1 Proposed proto (sketch)

```proto
// proto/api/v2/declarative.proto (new file)

message DeclarativeWorkflow {
  string schema_version = 1;                       // v1alpha1; unknown values reject
  WorkflowSettings workflow_config = 2;            // graph-wide interpreter settings
  map<string, ParameterSpec> parameters = 3;       // every {{ params.X }} must name a key
  map<string, DeclarativeTask> tasks = 4;          // map keys are task names
}

message WorkflowSettings {
  uint32 max_concurrency = 1;                      // per-level cap; unset = unbounded
  RetryPolicy default_retry = 2;                   // inherited by tasks with no retry
}

message ParameterSpec {
  string default = 1;                              // v1 keeps values string-typed
  string description = 2;
  bool required = 3;                               // required + default is rejected
}

message DeclarativeTask {
  string task_function = 1;                        // registry key: core/ray | acme.com/gpu-batch
  google.protobuf.Struct config = 2;               // validated against the type's schema
  google.protobuf.Struct job_specs = 3;            // type-scoped resource request
  repeated TaskDependency depends_on = 4;          // empty = root task
  RetryPolicy retry = 5;
}

message TaskDependency {
  string task = 1;
  enum Outcome { OUTCOME_UNSPECIFIED = 0; SUCCEEDED = 1; FAILED = 2; COMPLETED = 3; }
  Outcome when = 2;                                // UNSPECIFIED normalizes to SUCCEEDED
  // Field 3 is reserved for a future value-condition on the edge,
  // deferred from v1 (see Non-goals).
}

message RetryPolicy {
  uint32 max_attempts = 1;                         // total attempts including the first
  google.protobuf.Duration initial_interval = 2;
}
```

The change to `PipelineManifest` itself:

```proto
message PipelineManifest {
  enum Type {
    // ...existing values...
    reserved 2;                                  // ASL, removed before open-sourcing; see Open questions
    PIPELINE_MANIFEST_TYPE_DECLARATIVE = 4;
  }
  // Typed body of the manifest. Holds the run inputs for YAML and UniFlow
  // pipelines (TypedStruct) and the task graph for DECLARATIVE pipelines
  // (DeclarativeWorkflow). The Any type URL must match `type`.
  google.protobuf.Any content = 4 [(michelangelo.api.external_storage) = true];
  // ...existing fields 5-10 unchanged...

  reserved 11;                                   // future declarative_ref; see "Deferred flow"
}
```

The stored record holds the spec in `content`. Provenance is the existing `PipelineSpec.commit` and `PipelineManifest.file_path`; no new field.

Notes on deliberate choices:

- `job_specs` is an open `Struct`, not the closed Canvas `JobSpecs` message, so a vendor task type has somewhere to put its resource request. The `spark` and `ray` keys keep their exact Canvas meaning; a Canvas `job_specs` block is valid unchanged.
- `RetryPolicy` maps onto the factory's existing `retry_attempts` parameter. It is rejected at create time for task types that don't declare the `retry` capability.

### A.2 A worked spec

```yaml
kind: Pipeline
metadata: { name: churn-training }
spec:
  manifest:
    type: PIPELINE_MANIFEST_TYPE_DECLARATIVE
    content:
      '@type': type.googleapis.com/michelangelo.api.v2.DeclarativeWorkflow
      schema_version: v1alpha1
      workflow_config:
        max_concurrency: 4
        default_retry: { max_attempts: 2, initial_interval: 15s }
      parameters:
        learning_rate: { default: "0.01", description: "Adam LR" }
      tasks:
        prepare_features:
          task_function: spark                       # aliases to core/spark
          config: { entry_point: jobs/prepare_features.py }
          job_specs:
            spark:
              driver:   { pod: { resource: { cpu: 4, memory: 16G } } }
              executor: { pod: { resource: { cpu: 8, memory: 32G } }, instances: 10 }

        train_model:
          task_function: ray
          depends_on: [ { task: prepare_features } ] # when omitted -> SUCCEEDED
          config:
            entry_point: train.py
            args: ["--lr", "{{ params.learning_rate }}",
                   "--features", "{{ tasks.prepare_features.output.path }}"]
          retry: { max_attempts: 3, initial_interval: 30s }

        promote_model:
          task_function: ray
          depends_on: [ { task: train_model } ]      # runs only if train_model succeeds
          config:
            entry_point: promote.py                  # promote.py itself applies the AUC gate
            args: ["--min-auc", "0.85",
                   "--metrics", "{{ tasks.train_model.output.metrics_url }}"]

        failure_report:
          task_function: acme.com/notebook           # vendor type, no proto change
          depends_on: [ { task: train_model, when: FAILED } ]
          config: { notebook_path: notebooks/failure_report.ipynb }
```

How it runs: `prepare_features` runs alone in level 1, inheriting `default_retry`. On success, `train_model` runs with its own retry policy (replacement, not merge). Level 3 holds `promote_model` and `failure_report`; at most one runs. If `train_model` succeeds, `promote_model` runs — its own code decides whether the AUC clears the threshold, because value-conditional edges are deferred from v1 — and `failure_report` is skipped; a skip isn't a failure, so the run ends `SUCCEEDED`. If `train_model` fails, `failure_report` runs and the run still ends `FAILED` — failure handlers report and clean up; they never rewrite an outcome. If `prepare_features` fails, all three downstream tasks are skipped: a skipped or failed dependency satisfies no `when: SUCCEEDED` gate, and skips propagate.

### A.3 The explain-plan

The same validation pass with `dry_run: true` returns what will execute, before anything runs:

```yaml
levels:
  - [prepare_features]
  - [train_model]
  - [promote_model, failure_report]
effective_parameters:
  learning_rate: "0.05"                  # override over default "0.01"
effective_settings:
  max_concurrency: 4
  retry: { prepare_features: "inherited default_retry (2)", train_model: "own (3)" }
resolved_references:
  train_model.config.args[1]: "{{ params.learning_rate }} -> 0.05"
  train_model.config.args[3]: "{{ tasks.prepare_features.output.path }} -> (runtime)"
  promote_model.config.args[3]: "{{ tasks.train_model.output.metrics_url }} -> (runtime)"
edges:
  - "train_model <- prepare_features : SUCCEEDED"
  - "promote_model <- train_model : SUCCEEDED"
  - "failure_report <- train_model : FAILED"
interpreter_version: core-v1.4.2
task_types: { prepare_features: core/spark, train_model: core/ray, failure_report: acme.com/notebook }
```

### A.4 The same pipeline held in Git

Two artifacts exist. The committed file holds the pipeline and nothing about its own commit. The apply request adds the existing provenance fields around it.

The file in Git, `pipelines/churn-training.yaml`. It is the A.2 resource unchanged, and it is what `export` writes back:

```yaml
kind: Pipeline
metadata: { name: churn-training }
spec:
  manifest:
    type: PIPELINE_MANIFEST_TYPE_DECLARATIVE
    content:
      schema_version: v1alpha1
      ...
```

The request CI sends. `ma pipeline apply` reads the file, reads the commit from the checkout, and fills the existing `spec.commit` and `manifest.file_path` before calling create or update. `mactl` does this today for every pipeline (`create.py:387-398`); the declarative type adds only the `@type` line:

```yaml
# ma pipeline apply --file=pipelines/churn-training.yaml
kind: Pipeline
metadata: { name: churn-training }
spec:
  commit:                                   # existing PipelineSpec.commit, set by mactl from the checkout
    git_ref: 9f2c1e7d4b8a
    branch: main
  manifest:
    type: PIPELINE_MANIFEST_TYPE_DECLARATIVE
    file_path: pipelines/churn-training.yaml # existing PipelineManifest.file_path, set by mactl
    content:                                  # the file contents, inline
      '@type': type.googleapis.com/michelangelo.api.v2.DeclarativeWorkflow   # added by mactl
      schema_version: v1alpha1
      ...
```

`git_ref` is the full SHA, so each commit gets its own `Revision`. With revisioning enabled, the existing controller snapshots `Revision` `pipeline-churn-training-9f2c1e7d4b8a` with `source: Git`, and, because the branch is `main`, advances `status.latestRevision`. A console edit produces a `Revision` with `source: ResourceUpdate` through the same mechanism.

`ma pipeline export`, new in Phase 5, reverses the flow: it reads the stored spec from the pipeline read API and writes the first file, without `spec.commit` and without the `@type` line, so a pipeline authored in a console can enter the same repository and a re-export of a Git-applied pipeline produces no diff.

---

## Appendix B: Example — a vendor task type

*This appendix is illustrative. None of this code has been compiled or run; it shows the shape of what an adopter writes, assembled from the idioms of upstream's own plugins (`go/worker/plugins/ray/`, `python/michelangelo/uniflow/plugins/ray/task.star`). Parts of it depend on changes 5–7, which don't exist yet.*

The mock adopter, Acme, runs Ray over its own HTTP cluster service and wants both Ray implementations live in one deployment. Everything below sits in Acme's repos.

```
acme-ma-plugins/
├── go/
│   ├── rayhttp/{plugin.go, starlark_module.go}   # IPlugin + builtins
│   ├── activities/activities.go                  # HTTP clients as activities
│   └── cmd/worker/main.go                        # Acme's worker binary
└── python/acme_ma_plugins/ray/
    ├── task.star                                 # the factory (workflow side)
    ├── task.py                                   # TaskConfig for SDK users
    ├── descriptor.yaml                           # the registration unit
    ├── config.schema.json
    └── job_specs.schema.json
```

### B.1 Worker plugin and binary

```go
// rayhttp/plugin.go — the registry entry
const pluginID = "acme_ray"        // vendor-prefixed; plugin IDs are flat and global

var Plugin = &plugin{}

func (p *plugin) ID() string                              { return pluginID }
func (p *plugin) Create(_ service.RunInfo) starlark.Value { return newModule() }
func (p *plugin) Register(_ worker.Registry)              {}

func RegisterPlugin(registry map[string]service.IPlugin) {
    registry[Plugin.ID()] = Plugin
}
```

```go
// cmd/worker/main.go — a consumer of upstream's fx module, not a fork of it
func main() {
    fx.New(
        maworker.Module,                   // upstream, unmodified, from go.mod
        activities.Module,                 // Acme's HTTP-client activities
        fx.Invoke(rayhttp.RegisterPlugin), // adds "acme_ray" beside upstream's "ray"
    ).Run()
}
```

Each builtin in `starlark_module.go` recovers the workflow context from the Starlark thread and executes durable activities — the same shape as upstream's `go/worker/plugins/ray/starlark_module.go`. All I/O lives in activities, which is what keeps the Starlark side deterministic and replayable.

### B.2 The factory

The four calling-convention rules from the body, marked `[C1]`–`[C4]`:

```python
load("@plugin", "acme_ray", "json", "os", "time")

# Shared helpers via the aliased load root (change 6). Without it, this line
# is inexpressible outside the michelangelo tree.
load("@michelangelo//commons.star",
     "DEFAULT_RETRY_ATTEMPTS", "TASK_STATE_FAILED", "TASK_STATE_RUNNING",
     "TASK_STATE_SUCCEEDED", "TIME_FOMART", "get_result_url", "get_task_name",
     "io_read_json", "report_progress")

# [C1] Stage 1: configuration in, callable out. The keyword surface below IS
# the type's public schema — task.py's dataclass fields (SDK path) and
# config.schema.json's properties (declarative path) both map onto it 1:1.
def task(
        task_path,
        alias = None,
        retry_attempts = DEFAULT_RETRY_ATTEMPTS,
        cluster_profile = "standard",
        worker_instances = "1",
        raise_on_failure = True):            # [C4] outcome-mode switch

    # [C1] Stage 2: task arguments in, result out.
    def callable(*args, **kwargs):
        task_name = get_task_name(task_path, alias)
        start_time = time.utc_format_seconds(TIME_FOMART, time.time())
        result_url = get_result_url(task_path)

        # [C2] First visible transition. A factory that skips report_progress
        # doesn't degrade the run view — it renders an invisible task.
        report_progress(
            task_path = task_path, task_name = task_name,
            task_log = "", task_message = "Submitting to Acme Ray",
            task_state = TASK_STATE_RUNNING,
            start_time = start_time, end_time = "", output = "",
        )

        # [C3] The factory owns the retry loop; the interpreter never wraps one.
        failure = None
        for attempt in range(retry_attempts):
            job = acme_ray.create_job(spec = {
                "name":       task_name,
                "profile":    cluster_profile,
                "workers":    int(worker_instances),
                "entrypoint": _entrypoint(task_path, result_url, args, kwargs),
            })
            state = acme_ray.get_job(name = job["name"])["state"]  # durable poll

            if state == "SUCCEEDED":
                end_time = time.utc_format_seconds(TIME_FOMART, time.time())
                report_progress(                       # [C2] terminal transition
                    task_path = task_path, task_name = task_name,
                    task_log = job["log_url"], task_message = "Succeeded",
                    task_state = TASK_STATE_SUCCEEDED,
                    start_time = start_time, end_time = end_time,
                    output = result_url,
                )
                return io_read_json(result_url)

            failure = job.get("failure_message", state)
            report_progress(                           # [C2] per-attempt failure
                task_path = task_path, task_name = task_name,
                task_log = job["log_url"],
                task_message = "Attempt " + str(attempt + 1) + " failed: " + failure,
                task_state = TASK_STATE_FAILED,
                start_time = start_time, end_time = "",
                retry_attempt_id = str(attempt + 1),
                output = "",
            )

        if raise_on_failure:
            fail("acme_ray task failed: " + task_name + ": " + failure)

        # [C4] Outcome mode: return an outcome instead of aborting, enabling
        # `when: FAILED` edges. Exact shape is an open question; illustrative.
        return {"__task_outcome__": "FAILED", "error": failure}

    return callable

def _entrypoint(task_path, result_url, args, kwargs):
    # Same seam as core/ray: orchestration builds the command; the pod imports
    # the user's real Python by dotted path and runs it. This reproduces the
    # idiom of core/ray's ray_job_entrypoint, including its quoting weaknesses;
    # the command runs inside the user's own task pod (see the trust-domain
    # scoping in the body). A production factory should prefer an exec-array
    # form over a shell string.
    return ("python -m michelangelo.uniflow.core.run_task" +
            " --task '" + task_path + "'" +
            " --args '" + json.dumps(list(args)) + "'" +
            " --kwargs '" + json.dumps(dict(kwargs)) + "'" +
            " --result-url '" + result_url + "'")
```

### B.3 Descriptor and schemas

The descriptor is the registration unit. It lives next to the `.star` it points at — the only place it's versioned in lockstep with the factory:

```yaml
# descriptor.yaml
task_type: acme.com/ray          # spec-level registry key (what pipelines write)
calling_convention: v1           # pins the factory contract
plugin_id: acme_ray              # Starlark-side symbol (must be an identifier)

star:
  package: acme-ma-plugins==1.4.0
  file: acme_ma_plugins/ray/task.star
  factory: task

schema:
  config:    { $ref: acme_ma_plugins/ray/config.schema.json }
  job_specs: { $ref: acme_ma_plugins/ray/job_specs.schema.json }

capabilities: [retry, outcome_mode]   # no `cache`: a cache config for this
                                      # type is a create-time rejection, never
                                      # a silently dropped field
```

```json
// config.schema.json — strict; unknown fields reject at create time
{
  "type": "object",
  "additionalProperties": false,
  "required": ["entry_point"],
  "properties": {
    "entry_point":     { "type": "string" },
    "args":            { "type": "array", "items": { "type": "string" } },
    "cluster_profile": { "enum": ["standard", "gpu", "highmem"] }
  }
}
```

### B.4 What each consumer writes

SDK users (works today, except the `commons.star` load):

```python
from acme_ma_plugins.ray import AcmeRayTask   # pip install acme-ma-plugins

@task(AcmeRayTask(cluster_profile="gpu", worker_instances=4))
def train(features_path: str): ...
```

Declarative users (after this RFC lands):

```yaml
tasks:
  train:
    task_function: acme.com/ray
    config: { entry_point: train.py, cluster_profile: gpu }
    job_specs: { acme_ray: { worker_instances: 4 } }
    retry: { max_attempts: 3 }        # admitted: the descriptor declares `retry`
    depends_on: [ { task: prepare } ]
```

Users never see the descriptor or the schemas. The apiserver holds them and validates user specs against them.

---

## Appendix C: Example — assembly and release tooling

*This appendix is illustrative. The assembly tool doesn't exist today; it's part of change 3 (Phase 3), built on `build.py`'s existing packaging (`Package.to_tarball_bytes`, the `_add_star_file` load-resolution recursion). Command names, file shapes, and the release-naming scheme are proposals.*

### C.1 Why assembly exists

Starlark `load()` paths are string literals, resolved before execution:

```python
load(spec.task_function, "task")   # ILLEGAL — you can't load by a runtime value
```

So every factory an interpreter version can dispatch must be linked in when its tar is built. The interpreter tar is assembled, not shipped monolithic.

Contrast with the SDK build: the SDK compiles *each user's program* into its own tar (one per pipeline, containing transpiled user code). Assembly links *the platform's one program* into one tar per release, containing no user code; the user's pipeline arrives at run time as arguments. No transpilation, no AST work — file copying plus one generated dict.

### C.2 Inputs

One release manifest, owned by the deployment. Each named package carries its own descriptors:

```yaml
# interpreter-release.yaml — checked into the deployment's CI repo
name: acme-2026.08.1              # opaque release name; composition is data, not name
core: v1.4.2
extensions:
  - acme-ma-plugins==1.4.0        # contributes acme.com/ray
```

```
ma assemble --release interpreter-release.yaml --out ./dist
```

### C.3 Outputs

**Output 1 — the interpreter tar.** The generated entry file is the whole "execution table":

```python
# main.star — GENERATED; do not edit
load("michelangelo/interpreter/engine.star", "run_graph")
load("michelangelo/uniflow/plugins/ray/task.star",   __f_core_ray   = "task")
load("michelangelo/uniflow/plugins/spark/task.star", __f_core_spark = "task")
load("acme/ray/task.star",                           __f_acme_ray   = "task")

TASK_FACTORIES = {
    "core/ray":     __f_core_ray,
    "core/spark":   __f_core_spark,
    "acme.com/ray": __f_acme_ray,
}

def main(workflow_config, task_configs):
    return run_graph(workflow_config, task_configs, TASK_FACTORIES)
```

`main(workflow_config, task_configs)` is exactly the signature the run path already delivers positionally (`executeworkflow.go:561-577`). The tar's `meta.json` records provenance:

```json
{
  "main_file": "main.star",
  "main_function": "main",
  "interpreter_version": "acme-2026.08.1",
  "components": { "core": "v1.4.2", "extensions": { "acme.com/ray": "1.4.0" } },
  "descriptor_set_digest": "sha256:9f2c81d4…"
}
```

**Output 2 — the apiserver validation config**, same release name, same digest, plus the tar's address:

```yaml
declarative:
  interpreters:
    acme-2026.08.1:
      artifact: s3://ma-artifacts/interpreters/acme-2026.08.1.tar.gz
      descriptor_set_digest: sha256:9f2c81d4…
      components: { core: v1.4.2, extensions: { acme.com/ray: 1.4.0 } }
      task_types:
        core/ray:     { config_schema: …, job_specs_schema: …, capabilities: [retry, cache, outcome_mode] }
        core/spark:   { config_schema: …, job_specs_schema: …, capabilities: [retry, cache, outcome_mode] }
        acme.com/ray: { config_schema: …, job_specs_schema: …, capabilities: [retry, outcome_mode] }
```

No human writes either output. Users never see them; they write only the pipeline spec (Appendix A.2), which the apiserver validates against output 2.

### C.4 Using the outputs

| Output | Goes to | Read by | When |
|---|---|---|---|
| Tar | artifact store (S3), at the path output 2's `artifact:` names | pipeline-run controller, via `blobStore.Get(pin.artifact)` | every run |
| Validation config | apiserver config (existing ConfigMap pattern: `config/base/apiserver`, `config/overlays/<env>/apiserver`), rolled out by GitOps | apiserver validation hook | every create/update |

The worker reads neither: it receives the tar bytes inside the workflow start event.

Release sequence: assemble → upload tar → merge config → the release name becomes pinnable. The two outputs ship on independent timelines (CI vs GitOps), so both carry the descriptor-set digest; the run controller rejects a run whose tar digest disagrees with the pinned one, naming both. The apiserver pins from the config alone and never reads the tar. The naming scheme is the container-image split: the release name is the tag, the digest is the identity, the `components` block is the manifest. Composition is never encoded in the name.

### C.5 End-to-end lifecycle

```
pipeline CREATE (apiserver):
    unpack content: Any type URL == DeclarativeWorkflow?          else reject
    look up the spec's interpreter name in the validation config
    ├─ every task_function present in that release's table?      else reject
    ├─ config/job_specs valid against each type's schema?        else reject
    ├─ capabilities cover retry / outcome edges?                 else reject
    └─ write the resolved pin onto the pipeline record:
         { version, artifact URL, digest }        # tag→digest resolution;
                                                  # config may change later, the pin cannot
       store the validated spec in content. Revision snapshot: existing controller.
       The apiserver reads nothing from the blob store.

pipeline RECONCILE (pipeline controller):
    if manifest.type == DECLARATIVE and no pin:  state = ERROR   # bypassed the API
    else:                                        state = READY   # unchanged; no validation here

pipeline RUN (pipeline-run controller):
    switch manifest.type                          # before the content decode in
      case DECLARATIVE:                           # getWorkflowInputs, which would fail on
                                                  # this payload; env setup still runs
    tar = blobStore.Get(pin.artifact)             # same call shape as today's UniflowTar fetch
    verify sha256(tar) == pin.digest              # else fail loudly
    verify meta.json digest == pin.digest         # config/tar skew surfaces here, naming both
    StartWorkflow("starlark-workflow", tar, "", "",
                  [workflow_config, task_configs], ...)

worker:
    receives tar bytes in the start event; untars in memory, reads meta.json,
    runs main. Never touches the artifact store. Zero changes.
```

One alignment the digest can't check: the worker binary must link the plugins the tar's factories load. A mismatch fails at workflow start with a load error naming the plugin ID. Deployments close it by construction: the descriptor set that drives assembly is also the list of plugin IDs the worker binary must register, so an extension-bearing interpreter release and its worker binary ship from the same descriptor set and CI.
