# RFC-20260702-uniflow-spark-plugin: `run_spark_job()` — Custom Spark Entrypoints Without a Wrapped Python Task

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-07-02

---

## Problem statement

Uniflow (`python/michelangelo/uniflow/`) orchestrates workflow tasks through a pluggable `TaskConfig` architecture (`core/task_config.py`), but the existing Spark plugin, `SparkTask` (`plugins/spark/task.py`), assumes the task's real logic lives in a Python function invoked as the Spark driver via `run_task.py`. Authors of **custom Spark entrypoints** — a standalone Scala/JVM jar with a `main_class`, or a standalone PySpark `.py` script that doesn't want to be restructured around Uniflow's task-function convention — have no supported path onto Uniflow today. The only way to run such a job through Uniflow currently is to write a throwaway Python wrapper whose sole job is to shell out to the real job.

Separately, `SparkTask` already provides a fully capable path for Spark jobs that *do* want typed Uniflow I/O, and this RFC does not touch it.

## Motivation

There is a large population of Spark users whose jobs are not, and should not need to become, Python-wrapped Uniflow tasks — plain Scala/JVM Spark jobs are the most obvious case, but the same friction applies to any standalone PySpark script an author doesn't want to restructure around Uniflow's convention. Removing that requirement lets Uniflow's workflow orchestration (DAG authoring, scheduling, retries at the workflow level) reach a much larger set of Spark jobs without asking every author to restructure their job around a framework convention that doesn't fit it.

## Goals

- Let a custom Spark entrypoint (Scala/JVM jar + `main_class`, or a standalone `.py` script) run through Uniflow via a single function, `run_spark_job()`, callable directly inside a `@workflow()` body — no Python wrapper function containing real job logic, and no `TaskConfig`/`@task` wrapper either (see "High-level architecture" for why).
- Support both a local/dev run path (for iteration without a live cluster) and a remote run path through the existing production submission mechanism.
- Reuse the existing `SparkJobSpec` resource schema and `SparkJobService` — introduce the smallest possible amount of new backend surface: one new combined Starlark builtin (`spark.run_job`), built by composing the two Cadence activities `spark.create_job`/`spark.sensor_job` already use internally.
- Make the trade-off between "wrapped Uniflow task with full typed I/O" (`SparkTask`) and "custom entrypoint with job-status-only output" (`run_spark_job()`) explicit and unsurprising to users.

## Non-goals

- This RFC does not change or replace `SparkTask`. It remains the right choice for anyone who wants typed data passed automatically between workflow steps.
- This RFC does not add typed input/output marshaling for custom entrypoints. A custom entrypoint's driver process (a JVM, or a bare `spark-submit`-launched Python script) never calls into Uniflow's I/O registry, so `run_spark_job()` returns job status only — a deliberate, permanent limitation, not a phase-1 gap to close later.
- This RFC does not modify the `TaskConfig` ABC, `RayTask`, or `SparkTask`.
- This RFC does not add a `TaskConfig` plugin, a `.star` file, or a local-run/`spark-submit` helper. An earlier draft of this RFC proposed all three (see "Alternatives considered" — Alternative D); none of them turned out to be necessary once we confirmed a `@task`-wrapped entrypoint's Python body is never actually invoked on the remote path (see below), which makes a plain function call the correct shape instead.

## High-level architecture

Custom Spark entrypoints are **not** modeled as a `TaskConfig`/`@task` plugin. Here's why: `SparkTask`'s Python task function body *is* meaningfully executed remotely — the Spark driver runs `run_task.py`, which imports the task's module and calls the function directly, so its body is real, live code on the remote path. A custom jar/script entrypoint has no such bridge: the driver process is the user's own jar or script, and it never calls back into any Python task function. Wrapping it in `@task(config=SomeTaskConfig(...))` would only produce a task whose Python body is dead code remotely (it would only ever run for local execution) — misleading, and solving a problem (getting a DAG node) that a plain function call already solves more directly, the same way `python/michelangelo/uniflow/plugins/pipeline/run.py`'s `run_pipeline()` does for child pipeline runs.

So the shape is a single function, callable directly inside a `@workflow()` body — no decorator, no config object:

```
User, inside @workflow() body:
    run_spark_job(namespace=..., main_application_file=..., main_class=..., ...)
        |
        |  @star_plugin("spark.run_job") binding
        v
  ┌─────────────────────────────┬──────────────────────────────────────┐
  │  Local execution              │  Remote execution (transpiled)         │
  │  (plain Python call)          │  (core/build.py rewrites the call to   │
  │                                │   __spark__.run_job(...))              │
  ├─────────────────────────────┼──────────────────────────────────────┤
  │  create_spark_job() ->         │  Go: spark.run_job builtin              │
  │    APIClient.SparkJobService   │    (go/worker/plugins/spark/           │
  │    .create_spark_job()         │     starlark_module.go)                │
  │  poll_spark_job() ->           │    -> CreateSparkJob activity           │
  │    .get_spark_job() in a loop  │    -> SensorSparkJob activity, polling  │
  └─────────────────────────────┴──────────────────────────────────────┘
        |
        v
terminal SparkJobStatus returned to the calling workflow body (job status only -- no data payload)
```

`spark.run_job` is new (see "APIs and CRDs"), combining the two Cadence activities `spark.create_job`/`spark.sensor_job` already execute separately — no new activities, no new resource schema, no new CRD.

### Scope boundary: `SparkTask` keeps Uniflow I/O; `run_spark_job()` gets job status only

- `SparkTask` (unaffected): its Python entrypoint runs as the actual Spark driver process, so it can read/write through Uniflow's I/O registry — full typed input/output between workflow steps, same as `RayTask`.
- `run_spark_job()` (new): covers any custom entrypoint — a Scala/JVM jar or a standalone PySpark script. Its driver process is not invoked as a Uniflow task function and therefore cannot call into that I/O system. It returns only the terminal job status. If a downstream step needs data the job produced, that data must be written by the job itself to a location the caller already knows (a table, a blob/S3 path passed via `args`/`spark_conf`) and referenced explicitly in the next step's `args` — not read back through Uniflow's I/O registry.

## APIs and CRDs

No new CRDs, gRPC services, or REST endpoints. One new Starlark builtin, `spark.run_job` (`go/worker/plugins/spark/starlark_module.go`), composing the same two Cadence activities (`CreateSparkJob`, `SensorSparkJob`) the existing `create_job`/`sensor_job` builtins already drive — `createJob`/`sensorJob`'s bodies are refactored into shared private helpers that `run_job` also calls, so behavior for the two existing builtins is unchanged.

### New Python function: `run_spark_job()` — `python/michelangelo/uniflow/core/lib/spark/job.py`

Lives alongside `create_job()`/`sensor_job()` (the separate, lower-level builtins already implemented in this file — see the "core/lib/spark builtins" work this RFC's implementation started with). `run_spark_job()` is their combined form, decorated `@star_plugin("spark.run_job")`:

```python
@star_plugin("spark.run_job")
def run_spark_job(
    namespace: str,
    main_application_file: str,
    main_class: str | None = None,
    args: list[str] | None = None,
    driver_cpu: int | None = None,
    driver_memory: str | None = None,
    executor_cpu: int | None = None,
    executor_memory: str | None = None,
    executor_instances: int | None = None,
    spark_conf: dict[str, str] | None = None,
    deps_jars: list[str] | None = None,
    deps_py_files: list[str] | None = None,
    image: str | None = None,
    spark_version: str = "3.5.5",
    timeout_seconds: int = 0,
    poll_seconds: int = 10,
) -> dict:
    """Create a SparkJob and wait for it to reach a terminal state, synchronously.

    Local execution: calls create_spark_job() then poll_spark_job() directly
    against APIClient.SparkJobService. Remote execution: transpiles to the
    spark.run_job Starlark builtin, which does the equivalent via Cadence
    activities. Returns job status only -- no data payload (see Non-goals).
    """
    created = create_spark_job(
        namespace=namespace, main_application_file=main_application_file,
        main_class=main_class, args=args, driver_cpu=driver_cpu,
        driver_memory=driver_memory, executor_cpu=executor_cpu,
        executor_memory=executor_memory, executor_instances=executor_instances,
        spark_conf=spark_conf, deps_jars=deps_jars, deps_py_files=deps_py_files,
        image=image, spark_version=spark_version,
    )
    return poll_spark_job(
        namespace=namespace, name=created["metadata"]["name"],
        timeout_seconds=timeout_seconds, poll_seconds=poll_seconds,
    )
```

(Exact parameter list/defaults should match whatever `create_job()`/`sensor_job()` already settled on in `core/lib/spark/job.py` — this is illustrative, not a literal diff.)

Field mapping onto `SparkJobSpec` (unchanged from the original draft):

| `run_spark_job()` param | `SparkJobSpec` field |
|---|---|
| `main_application_file` | `spec.main_application_file` |
| `main_class` | `spec.main_class` (optional) |
| `args` | `spec.main_args` |
| `deps_jars` | `spec.deps.jars` |
| `deps_py_files` | `spec.deps.py_files` |
| `driver_cpu` / `driver_memory` | `spec.driver.pod.resource.{cpu,memory}` |
| `executor_cpu` / `executor_memory` / `executor_instances` | `spec.executor.pod.resource.{cpu,memory}` / `spec.executor.instances` |
| `spark_conf` | `spec.spark_conf` |
| `image` | `spec.driver.pod.image` / `spec.executor.pod.image` (found missing during sandbox validation — see #1457-adjacent fix, required for the SparkOperator to actually schedule pods) |

### New Go builtin: `run_job` — `go/worker/plugins/spark/starlark_module.go`

Registered alongside the existing `create_job`/`sensor_job` builtins in the same module. Implementation combines `createJob`'s activity-execution body and `sensorJob`'s poll loop (including its cancellation/`TerminateSparkJob` handling) into shared private helpers, then `runJob` calls both in sequence — mirroring how `go/worker/plugins/pipeline/starlark_module.go`'s `runPipeline` already combines its own create+sensor activities into one builtin. `createJob`/`sensorJob` remain available as separate builtins, calling the same extracted helpers, with unchanged behavior (covered by existing tests).

### Usage example

```python
from michelangelo.uniflow.core import workflow
from michelangelo.uniflow.core.lib.spark.job import run_spark_job

@workflow()
def my_pipeline(date: str):
    scala_result = run_spark_job(
        namespace="ma-examples",
        main_application_file="s3://my-bucket/my-app-1.0.jar",
        main_class="com.example.MySparkApp",
        args=["--date", date],
        executor_cpu=4, executor_memory="4G", executor_instances=10,
        image="my-registry/my-spark-image:latest",
    )

    # Custom PySpark -- same function, no main_class, uses deps_py_files
    pyspark_result = run_spark_job(
        namespace="ma-examples",
        main_application_file="s3://my-bucket/custom_job.py",
        deps_py_files=["s3://my-bucket/shared_utils.zip"],
        args=["--date", date],
        image="my-registry/my-spark-image:latest",
    )

    return [scala_result, pyspark_result]
```

No `TaskConfig`, no `@task` decorator, no separate local-run helper — `run_spark_job()`'s own body already handles both local and remote execution via the `@star_plugin` binding.

## Alternatives considered

### Alternative A: Wrap it as a `TaskConfig` plugin, `SparkPlugin(TaskConfig)` (original draft of this RFC)

**Pros:** Consistent with `RayTask`/`SparkTask`'s existing plugin shape; `@task(config=...)` is a familiar pattern.
**Cons:** For a custom jar/script entrypoint, the Spark driver process is the user's own jar or script — it never calls back into a Python task function the way `SparkTask`'s `run_task.py` does. Wrapping it in `@task(config=SparkPlugin(...))` would make the decorated Python function body dead code on the remote path (only ever executed locally), which is confusing and misrepresents what actually runs where.
**Why not chosen:** A `TaskConfig` implies "this Python function's body is the task," which isn't true here. A direct function call (`run_spark_job(...)`) matches reality and mirrors `pipeline.run_pipeline()`, which has the same "no task body, just an orchestration call" shape.

### Alternative B (pure-Python wrapper, no new Go builtin): `run_spark_job()` implemented purely in Python, calling `create_job()`/`sensor_job()` internally without being itself `@star_plugin`-bound

**Pros:** Zero Go changes.
**Cons:** Confirmed by tracing the transpiler (`core/build.py`'s `visit_Name`): any plain function referenced inside a `@workflow()` body that isn't itself `@task`/`@star_plugin`/`@workflow`/a `TaskConfig` hits a transpiler crash (`TypeError: issubclass() arg 1 must be a class`) — `@workflow()` bodies are transpiled to Starlark, and a bare Python function with real logic has no Starlark equivalent to transpile to. This wrapper would work fine as a standalone script but crash the moment it's called inside an actual `@workflow()` body — the primary use case.
**Why not chosen:** Doesn't actually satisfy the goal of being callable directly inside a `@workflow()` body.

### Alternative C: Keep it Scala-only (`ScalaSpark`), add a separate PySpark variant later if needed

**Pros:** Simpler initial scope.
**Cons:** The underlying mechanism, config shape, and job-status-only limitation are identical regardless of entrypoint language — the only difference is whether `main_class` is populated and whether `deps_jars` or `deps_py_files` is used. A second function would duplicate the entire wrapper for no benefit.
**Why not chosen:** Generalizing `run_spark_job()` to cover both costs nothing extra and directly serves both custom-Scala and custom-PySpark authors with one function.

### Alternative D: `TaskConfig` plugin + new `.star` file + local `spark-submit` helper (superseded draft, kept for history)

An earlier iteration of this RFC combined Alternative A's `TaskConfig` shape with a new `plugins/spark/plugin_task.star` file and a `local.py` helper that shelled out to `spark-submit` for local iteration. Once Alternative A was rejected (see above), the `.star` file and the `spark-submit` helper were no longer needed either: the chosen design's local path already runs via `core/lib/spark/job.py`'s plain Python calls against `APIClient.SparkJobService`, with no separate local-run mechanism required.

## Open questions

- [x] ~~Test strategy: for `.star`/Starlark testing, is there an existing test harness pattern that this feature's tests should follow?~~ **Resolved:** yes — `go/worker/plugins/spark/starlark_module_test.go`'s existing `suite.Suite` pattern (mock activities via `env.RegisterActivity`/`env.OnActivity`, execute a named Starlark test function via `s.env.Cadence.ExecuteFunction`). The new `run_job` builtin's test (`TestRunJobSuccessfully`) reuses this pattern directly, extending `testdata/test.star` with a `test_run_job()` function.
- [ ] `SparkTask` supports result caching keyed on Python args/kwargs (`cache_version`/`cache_enabled`). `run_spark_job()` has no serializable Python args to key a cache on (job specs are structural, not memoizable business logic) — should caching be out of scope for v1? Current answer: yes, out of scope — a re-run should just submit a new job.
- [ ] Should `run_spark_job()` validate that `main_class` is set when `main_application_file` ends in `.jar`, or is silent omission acceptable (SparkOperator will surface its own error either way)?
- [ ] `SparkTask` has a retry loop (`retry_attempts`) — should `run_spark_job()` support the same at the Python level, or is relying on `@workflow()`-level retry (if any) sufficient?

## Rollout strategy

Purely additive — no phasing, feature flags, or migration needed:

- `TaskConfig`, `RayTask`, and `SparkTask` are all unchanged.
- Users adopt `run_spark_job()` by importing it and calling it directly inside a `@workflow()` body — there is no migration path because there is nothing to migrate from for this specific capability.
- New infrastructure to deploy: the `spark.run_job` Starlark builtin (`go/worker/plugins/spark/starlark_module.go`) needs to ship in the worker build before any remote `@workflow()` calling `run_spark_job()` can execute on the remote path; the Python side (`core/lib/spark/job.py`) can land independently for local-execution testing.
- Recommended rollout: land the Go `run_job` builtin and the Python `run_spark_job()` function together (they're two halves of the same feature), validate end-to-end against a real jar and a real standalone `.py` script — both locally and via `ma pipeline run` on the sandbox — then document (README covering the job-status-only limitation) before wider announcement.

## References

- `python/michelangelo/uniflow/plugins/pipeline/run.py` — `run_pipeline()`, the reference pattern this design follows: a plain `@star_plugin`-bound function, callable directly inside a `@workflow()` body, no `TaskConfig`/`@task` wrapper.
- `python/michelangelo/uniflow/core/lib/spark/job.py` — `create_job()`/`sensor_job()` (existing lower-level builtins) and `run_spark_job()` (this RFC's combined function).
- `python/michelangelo/uniflow/plugins/spark/task.py` + `task.star` — `SparkTask` (existing wrapped-task plugin; contrast case, not the pattern followed here).
- `go/worker/plugins/spark/starlark_module.go` — `create_job`/`sensor_job` builtins (existing) and `run_job` (new, this RFC).
- `go/worker/plugins/pipeline/starlark_module.go` — `runPipeline`, the reference pattern for combining create+sensor activities into a single Go builtin.
- `proto/api/v2/spark_job.proto` — `SparkJobSpec` schema.

## Issues

- [michelangelo-ai/michelangelo#1463](https://github.com/michelangelo-ai/michelangelo/issues/1463) — Implement `SparkPlugin` (tracking issue for this RFC's implementation).
- [michelangelo-ai/michelangelo#1457](https://github.com/michelangelo-ai/michelangelo/issues/1457) — `SparkApplication.Type` is hardcoded to `Python` regardless of entrypoint language (`go/components/spark/job/client/client.go:54`). Found while sandbox-validating the `core/lib/spark` builtins this RFC's implementation depends on; doesn't currently break execution (the operator falls back to `--class` when `main_class` is set) but the declared CRD type is wrong for jar/Scala entrypoints.
