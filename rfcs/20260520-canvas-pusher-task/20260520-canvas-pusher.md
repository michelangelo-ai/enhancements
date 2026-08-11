# RFC-20260520-canvas-pusher: Canvas Pusher — Phase 1 Open Source Design

- **Status:** Accepted
- **Author(s):** kenns29, sallycr
- **Created:** 2026-05-20

---

## Problem statement

The Michelangelo pusher task (`ModelPusherPlugin`, `EvalReportPusherPlugin`, `DatasetPusherPlugin`) is tightly coupled to Uber-internal infrastructure. External adopters cannot use the pusher — there is no open-source-compatible implementation of any plugin, no public storage backend, and no registry client. This blocks the Canvas open source of making the full ML workflow (train → evaluate → push) runnable in open source.

## Motivation

Three built-in plugins for pusher task makes work end-to-end in the open source `michelangelo` pip package — with no Uber dependencies. 

## Goals

- `push()` is callable in open source with `LocalStorageBackend` and any `ModelRegistryClient` implementation.
- `ModelPusherPlugin`, `DatasetPusherPlugin`, and `EvalReportPusherPlugin` are fully functional with only stdlib + optional extras.
- `MLflowRegistryClient` and `VertexAIRegistryClient` ship as optional extras (`pip install "michelangelo[mlflow]"`, `pip install "michelangelo[vertex-ai]"`).
- Test coverage ≥ 90% per file, enforced by `diff-cover`.

## Non-goals

- Provider-specific storage backend, registry client, and plugin registry implementations — deferred to Phases 2–3.
- Async `push()` execution — synchronous only in Phase 1.
- Entry point–based plugin discovery — deferred; `register_entrypoints(group)` stub to be added as a future extension signal.
- Kubernetes or pipeline-runner integration — `push()` is a standalone function.

## High-level architecture

```
michelangelo.workflow.tasks.pusher/
│
├── Public API: push(config, artifacts, storage_backend, registry_client)
│
├── Core abstractions (ABCs)
│   ├── StorageBackend          ← LocalStorageBackend (open source)
│   ├── ModelRegistryClient     ← MLflowRegistryClient (mlflow extra)
│   │                           ← VertexAIRegistryClient (vertex-ai extra)
│   └── PusherPluginBase        ← ModelPusherPlugin
│                               ← DatasetPusherPlugin
│                               ← EvalReportPusherPlugin
│
└── PluginRegistry
    ├── default_registry  ← populated in plugins/__init__.py
    └── extend()          ← returns child registry for custom provider layer
```

### Dependency order (no circular imports)

```
exceptions → types → storage_backend → registry_client
          → config          (needs exceptions, ClassVar fix)
          → registry        (TYPE_CHECKING guard on PusherPluginBase)
          → plugins/base    (needs storage_backend, registry_client)
          → plugins/model_plugin, dataset_plugin, eval_report_plugin
          → plugins/__init__.py   (populates default_registry)
          → pusher.py             (imports registry, config, types)
          → __init__.py           (triggers plugins/__init__.py population)
          → implementations/      (standalone optional extras)
```

## APIs and CRDs

No new CRDs or gRPC/REST API changes. This is a Python package change only.

### Public API surface (`pusher/__init__.py`)

```python
from michelangelo.workflow.tasks.pusher import (
    push,                  # core dispatch function
    PusherConfig,          # top-level config
    PusherPluginConfig,    # per-artifact config
    ModelPluginConfig,     # config for ModelPusherPlugin
    DatasetPluginConfig,   # config for DatasetPusherPlugin
    EvalReportPluginConfig,# config for EvalReportPusherPlugin
    DatasetFormat,         # CSV / PARQUET / JSON enum
    AssembledModel,        # pre-packaged model artifact pair
    ModelArtifact,         # single packaged artifact
    PusherResult,          # per-plugin outcome
    PusherPluginBase,      # ABC for custom plugins
    StorageBackend,        # ABC for storage backends
    ModelRegistryClient,   # ABC for registry clients
    LocalStorageBackend,   # local FS backend (dev/test)
    RegisteredModel,       # registry response type
    PluginRegistry,        # plugin registry class
    default_registry,      # pre-populated registry singleton
    PusherError,           # base exception
    PusherPluginError,     # plugin execution failure
    ArtifactNotFoundError, # artifact key missing
    ConfigurationError,    # config validation failure
)
```

### `push()` signature

```python
def push(
    config: PusherConfig,
    artifacts: dict[str, Any],
    storage_backend: Optional[StorageBackend] = None,   # defaults to LocalStorageBackend(tempfile.mkdtemp())
    registry_client: Optional[ModelRegistryClient] = None,
    registry: Optional[PluginRegistry] = None,          # defaults to default_registry
    fail_fast: bool = True,
    on_error: Optional[Callable[[str, str, Exception], None]] = None,
) -> list[PusherResult]: ...
```

### Breaking changes from internal `PusherPluginBase`

`__init__` now accepts `storage_backend` and `registry_client` as injected arguments. The internal signature is `__init__(conf, var)` with no infrastructure injection. Phase 3 will update internal plugins to accept these.

### Bug fixes included

| Bug | Location | Fix |
|---|---|---|
| `import os` inside `upload()` causes `NameError` at `download()` runtime | `storage_backend.py` | Move `import os` to module level |
| `_BUILTIN_FIELDS` as plain annotation pollutes dataclass `__init__`/`__repr__`/`__eq__` | `config.py` | Annotate as `ClassVar[tuple[str, ...]]` |
| `which_plugin()` silently picks first non-None field when multiple plugins set | `config.py` | Replace with `resolved_plugin_name()` that raises `ConfigurationError` on ambiguity |

## Alternatives considered

### Alternative A: Keep Uber types, add adapters

Wrap `DatasetVariable` (Spark DataFrame) and `MessageVariable` (protobuf) in thin adapters that convert to plain Python types.

**Pros:** Minimal change to internal plugin code.  
**Cons:** Pulls Spark and protobuf into the open source package as transitive dependencies. External users cannot install these. Does not solve the registry client problem.  
**Why not chosen:** Adapters are leaky — every adapter surface becomes a compatibility burden across Uber and OSS versions.

### Alternative B: Copy-fork internal plugins into the OSS package

Duplicate the internal plugin files, strip Uber imports, ship both sets.

**Pros:** No breaking change to internal plugin APIs.  
**Cons:** Two implementations diverge immediately. Any fix must be applied twice. Phase 3 migration becomes harder, not easier.  
**Why not chosen:** A single-implementation strategy (OSS base + Uber extension) is more maintainable.

## Open questions

- [ ] **Entry point support:** Should `PluginRegistry.register_entrypoints(group)` be added as a stub in Phase 1, or deferred? Third-party plugins currently require explicit wiring at call site. (Researcher finding: add the stub now as an extensibility signal.)
- [ ] **Vertex AI `serving_container_image_uri`:** `VertexAIRegistryClient.register_model()` passes `serving_container_image_uri=None`, registering a model that cannot be deployed to a Vertex AI endpoint without additional steps. Should this be an optional constructor parameter in Phase 1?
- [ ] **Extras self-containment:** Should `pandas`+`pyarrow` be absorbed into the `mlflow` and `vertex-ai` extras so each is installable without also specifying `[pusher]`? (Researcher and DX findings both flag this.)
- [ ] **Minimum Python version:** Design uses `Optional[X]` (Python 3.9 compat). Should 3.9 be stated explicitly in `pyproject.toml`?
- [ ] **Thread safety documentation:** `default_registry` is a module-level singleton. Post-import `register()` calls are not thread-safe. Should this be documented as a constraint or guarded with a lock?

## Rollout strategy

**Phase 1 (this RFC):** Implement all files in `pusher/` with no Uber dependencies. Add `pusher`, `mlflow`, `vertex-ai` extras to `pyproject.toml`. Ship `MLflowRegistryClient` and `VertexAIRegistryClient`. Achieve ≥ 90% test coverage. Add Boston Housing end-to-end example.

**Phase 2:** Implement a provider plugin registry — calls `default_registry.extend()` and registers provider-specific plugins. No changes to open source files.

**Phase 3:** Implement provider-specific `StorageBackend` and `ModelRegistryClient` implementations. Update internal plugins to accept the new `PusherPluginBase.__init__` signature (storage_backend, registry_client injection).

**Rollback:** Each phase is an independent PR. Phase 1 adds new files only — no existing file is modified. Rollback = revert the PR. No migration needed.

**Verification:**

```bash
cd michelangelo/python

# Lint
poetry run ruff format --check michelangelo/workflow/tasks/pusher/
poetry run ruff check michelangelo/workflow/tasks/pusher/

# Tests + coverage
poetry run pytest michelangelo/workflow/tasks/pusher/tests/ -v \
  --cov=michelangelo/workflow/tasks/pusher \
  --cov-report=term-missing

# End-to-end example
MODEL_PATH=$(find /tmp/ray_results -name "model.ubj" | head -1)
PYTHONPATH=. poetry run python \
    examples/boston_housing_xgb/push_trained_model.py \
    --model-path "$MODEL_PATH" \
    --store-dir /tmp/boston-pushed
```

## References

- Source design doc: `cnavas-pusher-design-extension.md` (this folder)
- Agent team findings: `architect-findings.md`, `researcher-findings.md`, `dev-advocate-findings.md`, `coder-findings.md` (this folder)
- [MLflow Model Registry docs](https://mlflow.org/docs/latest/model-registry.html)
- [Vertex AI Model Registry docs](https://cloud.google.com/vertex-ai/docs/model-registry/introduction)
- [PluginRegistry extension pattern — similar: pytest plugin system](https://docs.pytest.org/en/stable/how-to/plugins.html)

## Implementation tracking

- **Parent Tracking Issue:** [michelangelo-ai/michelangelo#1778](https://github.com/michelangelo-ai/michelangelo/issues/1778)
