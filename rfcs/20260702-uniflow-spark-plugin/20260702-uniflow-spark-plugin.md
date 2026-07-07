# RFC-20260702-uniflow-spark-plugin: SparkPlugin — Custom Spark Entrypoints Without a Wrapped Python Task

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

- Let a custom Spark entrypoint (Scala/JVM jar + `main_class`, or a standalone `.py` script) run as a Uniflow task via a new `TaskConfig` plugin, `SparkPlugin`, with no Python wrapper function containing real job logic.
- Support both a local/dev run path (for iteration without a live cluster) and a remote run path through the existing production submission mechanism.
- Reuse the existing `spark.create_job`/`spark.sensor_job` Starlark builtins and the existing `SparkJob` resource schema — introduce no new backend infrastructure.
- Make the trade-off between "wrapped Uniflow task with full typed I/O" and "custom `SparkPlugin` entrypoint with job-status-only output" explicit and unsurprising to users.

## Non-goals

- This RFC does not change or replace `SparkTask`. It remains the right choice for anyone who wants typed data passed automatically between workflow steps.
- This RFC does not add typed input/output marshaling for `SparkPlugin` tasks. A custom entrypoint's driver process (a JVM, or a bare `spark-submit`-launched Python script) never calls into Uniflow's I/O registry, so `SparkPlugin` tasks return job status only — a deliberate, permanent limitation, not a phase-1 gap to close later.
- This RFC does not modify the `TaskConfig` ABC, `RayTask`, or `SparkTask`.
- This RFC does not add a new Starlark builtin or backend activity. `spark.create_job`/`spark.sensor_job` (`go/worker/plugins/spark/starlark_module.go`) already provide the submit-then-wait mechanism this feature needs.

## High-level architecture

`SparkPlugin` is a thin `TaskConfig` implementation next to the existing `SparkTask`, reusing infrastructure that's already in the repo:

- `spark.create_job(job, timeout_seconds=0) -> job` and `spark.sensor_job(job, timeout_seconds=0, poll_seconds=10, assert_condition_type=...) -> job` — existing Starlark builtins in `go/worker/plugins/spark/starlark_module.go`, already used by `SparkTask`'s `task.star`.
- `SparkJobSpec` (`proto/api/v2/spark_job.proto`) — an existing resource spec with `main_class`, `main_application_file` ("e.g. jar or python file"), `main_args`, `deps.{jars, files, py_files}`, driver/executor pod specs, and `spark_conf`. This schema already supports both Scala/JVM and Python entrypoints.

```
User @task(config=SparkPlugin(...))
        |  TaskConfig.to_keywords() emits application_file, optional main_class, and the rest of the config
        v
plugins/spark/plugin_task.star  (new, thin — builds a SparkJob spec dict)
        |  spark.create_job(job=spec)
        v
spark plugin builtin (existing)  --Activity-->  CreateSparkJob-style activity (existing)
        |  spark.sensor_job(job=created, assert_condition_type=spark.succeeded_condition_type)
        v
spark plugin builtin (existing)  --Activity-->  SensorSparkJob-style activity, polling (existing)
        |
        v
terminal SparkJobStatus returned to the calling workflow step (job status only -- no data payload)
```

The only new artifacts are one Python `TaskConfig` dataclass and one `.star` file; nothing changes in the Go worker or the resource schema.

### Scope boundary: `SparkTask` keeps Uniflow I/O; `SparkPlugin` gets job status only

- `SparkTask` (unaffected): its Python entrypoint runs as the actual Spark driver process, so it can read/write through Uniflow's I/O registry — full typed input/output between workflow steps, same as `RayTask`.
- `SparkPlugin` (new): covers any custom entrypoint — a Scala/JVM jar or a standalone PySpark script — whose driver process is not invoked as a Uniflow task function and therefore cannot call into that I/O system. Its callable returns only the terminal job status. If a downstream step needs data the job produced, that data must be written by the job itself to a location the caller already knows (a table, a blob/S3 path passed via `args`/`spark_conf`) and referenced explicitly in the next task's `args` — not read back through Uniflow's I/O registry.

## APIs and CRDs

No new CRDs, gRPC services, or REST endpoints. All changes are internal to the Python `uniflow` SDK.

### New Python SDK class: `SparkPlugin(TaskConfig)` — `python/michelangelo/uniflow/plugins/spark/plugin_task.py`

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Dict, List, Optional

from michelangelo.uniflow.core.task_config import TaskBinding, TaskConfig

_binding = TaskBinding(
    star_file=Path(__file__).resolve().parent / "plugin_task.star",
    function="spark_plugin_task",
    export="__spark_plugin_task",
)

_config_binding = TaskBinding(
    star_file=Path(__file__).resolve().parent / "plugin_task.star",
    function="spark_plugin_config",
    export="__spark_plugin_config",
)


@dataclass
class SparkPlugin(TaskConfig):
    """TaskConfig for submitting a custom Spark entrypoint (a Scala/JVM jar
    with a main class, or a standalone PySpark script) without wrapping it
    in a Uniflow task function. Returns job status only -- see Non-goals."""

    application_file: str
    main_class: Optional[str] = None
    args: List[str] = field(default_factory=list)
    driver_cpu: Optional[int] = None
    driver_memory: Optional[str] = None
    executor_cpu: Optional[int] = None
    executor_memory: Optional[str] = None
    executor_instances: Optional[int] = None
    spark_conf: Dict[str, str] = field(default_factory=dict)
    deps_jars: List[str] = field(default_factory=list)
    deps_py_files: List[str] = field(default_factory=list)
    namespace: str = "default"

    def get_binding(self) -> TaskBinding:
        return _binding

    @classmethod
    def get_config_binding(cls) -> TaskBinding:
        return _config_binding

    def pre_run(self):
        """No-op -- the entrypoint manages its own runtime."""

    def post_run(self):
        """No-op -- see pre_run()."""
```

`to_keywords()` is inherited unchanged from `TaskConfig` — it already converts non-`None` dataclass fields into Starlark keyword arguments, the same mechanism `RayTask`/`SparkTask` rely on.

Field mapping onto the existing `SparkJobSpec`:

| `SparkPlugin` field | `SparkJobSpec` field |
|---|---|
| `application_file` | `spec.main_application_file` |
| `main_class` | `spec.main_class` (optional) |
| `args` | `spec.main_args` |
| `deps_jars` | `spec.deps.jars` |
| `deps_py_files` | `spec.deps.py_files` |
| `driver_cpu` / `driver_memory` | `spec.driver.pod.resource.{cpu,memory}` |
| `executor_cpu` / `executor_memory` / `executor_instances` | `spec.executor.pod.resource.{cpu,memory}` / `spec.executor.instances` |
| `spark_conf` | `spec.spark_conf` |

Field names deliberately mirror `SparkTask`'s existing `driver_cpu`/`driver_memory`/`executor_cpu`/`executor_memory`/`executor_instances` naming rather than inventing a second vocabulary for the same resource knobs.

### New `.star` file: `python/michelangelo/uniflow/plugins/spark/plugin_task.star`

```python
load("@plugin", "os", "spark", "time")
load("../../commons.star", "TASK_STATE_FAILED", "TASK_STATE_KILLED", "TASK_STATE_SUCCEEDED", "get_task_image", "get_task_name", "report_progress")

def spark_plugin_task(
        task_path,
        alias = None,
        application_file = "",
        main_class = None,
        args = [],
        driver_cpu = None,
        driver_memory = None,
        executor_cpu = None,
        executor_memory = None,
        executor_instances = None,
        spark_conf = {},
        deps_jars = [],
        deps_py_files = [],
        namespace = "default"):
    def callable(*call_args, **call_kwargs):
        task_name = get_task_name(task_path, alias)
        image = get_task_image(task_name)
        start_time = time.time()

        spec = {
            "driver": {"pod": {"resource": {"cpu": driver_cpu, "memory": driver_memory}, "image": image}},
            "executor": {"pod": {"resource": {"cpu": executor_cpu, "memory": executor_memory}, "image": image}, "instances": executor_instances},
            "sparkConf": spark_conf,
            "mainApplicationFile": application_file,
            "mainArgs": args,
            "deps": {"jars": deps_jars, "pyFiles": deps_py_files},
            "sparkVersion": "3.5.5",
        }
        if main_class:
            spec["mainClass"] = main_class

        job = {
            "kind": "SparkJob",
            "apiVersion": "michelangelo.api.v2",
            "metadata": {"namespace": namespace, "generateName": "uniflow-splg-"},
            "spec": spec,
        }

        response = spark.create_job(job)
        created = response["sparkJob"]
        final = spark.sensor_job(job = created, assert_condition_type = spark.succeeded_condition_type)

        state = TASK_STATE_SUCCEEDED
        conditions = final.get("status", {}).get("statusConditions", []) if type(final) == "dict" else []
        for c in conditions:
            if c and c["type"] == spark.killed_condition_type and c.get("status") == "CONDITION_STATUS_TRUE":
                state = TASK_STATE_KILLED
            if c and c["type"] == spark.succeeded_condition_type and c.get("status") == "CONDITION_STATUS_FALSE":
                state = TASK_STATE_FAILED

        report_progress(task_path = task_path, task_name = task_name, task_state = state, start_time = start_time, end_time = time.time(), task_message = "SparkPlugin job " + state, output = "")
        return final

    return callable

def spark_plugin_config(driver_cpu = None, driver_memory = None, executor_cpu = None, executor_memory = None, executor_instances = None):
    overrides = {"driver_cpu": driver_cpu, "driver_memory": driver_memory, "executor_cpu": executor_cpu, "executor_memory": executor_memory, "executor_instances": executor_instances}
    return {k: v for k, v in overrides.items() if v != None}
```

This mirrors `SparkTask`'s existing `task.star`: build a `SparkJob` spec, `spark.create_job` it, `spark.sensor_job` it to a terminal state — just without the retry loop and `io_read_json(result_url)` step that only make sense for a wrapped Python entrypoint with cacheable args/kwargs.

### New local-run helper: `python/michelangelo/uniflow/plugins/spark/local.py`

```python
import subprocess


def run_spark_submit_local(config: "SparkPlugin", extra_args: list = None) -> dict:
    cmd = ["spark-submit", "--master", "local[*]"]
    if config.driver_memory:
        cmd += ["--driver-memory", config.driver_memory]
    if config.executor_memory:
        cmd += ["--executor-memory", config.executor_memory]
    if config.executor_instances:
        cmd += ["--num-executors", str(config.executor_instances)]
    if config.main_class:
        cmd += ["--class", config.main_class]
    for k, v in config.spark_conf.items():
        cmd += ["--conf", f"{k}={v}"]
    if config.deps_jars:
        cmd += ["--jars", ",".join(config.deps_jars)]
    if config.deps_py_files:
        cmd += ["--py-files", ",".join(config.deps_py_files)]
    cmd += [config.application_file, *config.args, *(extra_args or [])]
    result = subprocess.run(cmd, capture_output=True, text=True, check=True)
    return {"stdout": result.stdout, "returncode": result.returncode}
```

Ships so users don't hand-roll subprocess code for local testing — the same config object drives both the local `spark-submit` invocation and the remote `SparkJob` payload.

### Usage example

```python
from michelangelo.uniflow.core import task, workflow
from michelangelo.uniflow.plugins.spark.plugin_task import SparkPlugin
from michelangelo.uniflow.plugins.spark.local import run_spark_submit_local

# Scala/JVM
_scala_config = SparkPlugin(
    application_file="s3://my-bucket/my-app-1.0.jar",
    main_class="com.example.MySparkApp",
    args=["--date", "2026-07-02"],
    executor_cpu=4, executor_memory="4G", executor_instances=10,
)

@task(config=_scala_config)
def my_scala_spark_task(date: str) -> dict:
    return run_spark_submit_local(_scala_config, extra_args=["--date", date])

# Custom PySpark -- same plugin, no main_class, uses deps_py_files
_pyspark_config = SparkPlugin(
    application_file="s3://my-bucket/custom_job.py",
    deps_py_files=["s3://my-bucket/shared_utils.zip"],
)

@task(config=_pyspark_config)
def my_custom_pyspark_task(date: str) -> dict:
    return run_spark_submit_local(_pyspark_config, extra_args=["--date", date])


@workflow()
def my_pipeline(date: str):
    my_scala_spark_task(date)
    return my_custom_pyspark_task(date)
```

## Alternatives considered

### Alternative A: Build `plugin_task.star` directly against Kubernetes APIs, bypassing the `spark` plugin builtins

**Pros:** No dependency on the `spark` plugin module.
**Cons:** `spark.create_job`/`spark.sensor_job` already implement the submit+wait sequence correctly, including status-condition checking and the job lifecycle `task.star` already relies on for `SparkTask`. Reimplementing this would duplicate real production logic.
**Why not chosen:** No benefit over reusing the existing, already-correct mechanism.

### Alternative B: Add a `local_callable` abstract method to the `TaskConfig` ABC

**Pros:** Would formalize a distinct local-run contract for plugins whose local behavior differs from "call the function in-process."
**Cons:** Breaks the ABC contract for `RayTask`/`SparkTask`, both of which would need to implement it, with no benefit to those plugins.
**Why not chosen:** Unjustified breaking change to an existing, working contract.

### Alternative C: Keep it Scala-only (`ScalaSpark`), add a separate `PySparkPlugin` later if needed

**Pros:** Simpler initial scope.
**Cons:** The underlying mechanism, config shape, and job-status-only limitation are identical regardless of entrypoint language — the only difference is whether `main_class` is populated and whether `deps_jars` or `deps_py_files` is used. A second plugin class would duplicate the entire `.star` wrapper for no benefit.
**Why not chosen:** Generalizing to `SparkPlugin` costs nothing extra and directly serves both custom-Scala and custom-PySpark authors with one plugin.

## Open questions

- [ ] `SparkTask` supports result caching keyed on Python args/kwargs (`cache_version`/`cache_enabled`). `SparkPlugin` has no serializable Python args to key a cache on — should caching be out of scope for v1?
- [ ] Confirm the condition-checking logic in `plugin_task.star` (killed/succeeded checks) matches what `spark.sensor_job`'s blocking-until-terminal behavior actually returns, cross-checked against `SparkTask`'s own `report_spark_job_terminated`.
- [ ] Should `SparkPlugin.__post_init__` validate that `main_class` is set when `application_file` ends in `.jar`, or is silent omission acceptable?
- [ ] `SparkTask` has a retry loop (`retry_attempts`) — should `SparkPlugin` support the same, given it has no cache/result state to reconcile between attempts?
- [ ] Test strategy: for `.star` testing, is there an existing Starlark test harness pattern (e.g. `go/worker/plugins/spark/starlark_module_test.go`) that `SparkPlugin`'s tests should follow?

## Rollout strategy

Purely additive — no phasing, feature flags, or migration needed:

- `TaskConfig`, `RayTask`, and `SparkTask` are all unchanged.
- Users adopt `SparkPlugin` by importing it and passing it to `@task(config=SparkPlugin(...))` — there is no migration path because there is nothing to migrate from for this specific plugin.
- No new infrastructure to deploy: `spark.create_job`, `spark.sensor_job`, their backing activities, and the `SparkJobSpec` schema's dual-entrypoint support all already exist.
- Recommended rollout: land the Python dataclass + `.star` file + local-run helper together, validate against a real jar and a real standalone `.py` script, then document the plugin (README + docs page covering the job-status-only limitation) before wider announcement.

## References

- `python/michelangelo/uniflow/core/task_config.py` — `TaskConfig`/`TaskBinding` (existing plugin pattern this design follows)
- `python/michelangelo/uniflow/plugins/spark/task.py` + `task.star` — `SparkTask` (existing wrapped-task plugin, the reference implementation for `SparkPlugin`)
- `python/michelangelo/uniflow/plugins/ray/task.py` + `task.star` — `RayTask` (second reference point for the `TaskConfig` plugin shape)
- `go/worker/plugins/spark/starlark_module.go` — `spark.create_job`/`spark.sensor_job` builtins
- `proto/api/v2/spark_job.proto` — `SparkJobSpec` schema

## Issues

- [michelangelo-ai/michelangelo#1457](https://github.com/michelangelo-ai/michelangelo/issues/1457) — `SparkApplication.Type` is hardcoded to `Python` regardless of entrypoint language (`go/components/spark/job/client/client.go:54`). Found while sandbox-validating the `core/lib/spark` builtins this RFC's implementation depends on; doesn't currently break execution (the operator falls back to `--class` when `main_class` is set) but the declared CRD type is wrong for jar/Scala entrypoints.
