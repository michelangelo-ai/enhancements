# CanvasFlex Pusher: Phase 1 Design — Open Source Implementation

**Team:** CanvasFlex Migration Team  
**Date:** 2026-05-19  
**Scope:** Open source implementation of `ModelPusherPlugin`, `EvalReportPusherPlugin`, `DatasetPusherPlugin`, and their shared dependencies  
**Status:** Ready for team review  
**Parent doc:** `CANVASFLEX_OPEN_SOURCE_DESIGN.md`  
**Source doc:** `CANVASFLEX_PHASE1_PLUGIN_DESIGN.md` (open source sections only)

---

## 1. Scope

This document covers **Phase 1 subset** only: the files required to make all three
built-in plugins work end-to-end in the open source `michelangelo` pip package.

**In scope:**

| File | Role |
|---|---|
| `pusher/exceptions.py` | Exception hierarchy used by all plugins |
| `pusher/types.py` | `AssembledModel`, `ModelArtifact`, `PusherResult` |
| `pusher/storage_backend.py` | `StorageBackend` ABC + `LocalStorageBackend` |
| `pusher/registry_client.py` | `ModelRegistryClient` ABC + `RegisteredModel` |
| `pusher/config.py` | `PusherConfig`, `PusherPluginConfig`, `ModelPluginConfig`, `EvalReportPluginConfig`, `DatasetPluginConfig`, `DatasetFormat` |
| `pusher/plugins/base.py` | `PusherPluginBase` ABC |
| `pusher/plugins/model_plugin.py` | `ModelPusherPlugin` |
| `pusher/plugins/eval_report_plugin.py` | `EvalReportPusherPlugin` |
| `pusher/plugins/dataset_plugin.py` | `DatasetPusherPlugin` |
| `pusher/registry.py` | `PluginRegistry` + `default_registry` |
| `pusher/pusher.py` | `push()` — core dispatch function |
| `pusher/plugins/__init__.py` | Populates `default_registry` with built-in plugins |
| `pusher/__init__.py` | Public API surface |
| `pusher/implementations/mlflow_client.py` | `MLflowRegistryClient` (optional extra) |
| `pusher/implementations/vertex_ai_client.py` | `VertexAIRegistryClient` (optional extra) |
| `pusher/tests/` | Full test suite for all files above |

**Still deferred (Phases 2–3):**

- All provider-specific layer files (custom storage backend, registry client, provider plugin registry)

---

## 2. File Layout

```
michelangelo/python/michelangelo/workflow/tasks/pusher/
├── __init__.py
├── exceptions.py
├── types.py
├── storage_backend.py
├── registry_client.py
├── config.py
├── registry.py
├── pusher.py
├── implementations/
│   ├── __init__.py
│   ├── mlflow_client.py
│   └── vertex_ai_client.py
├── plugins/
│   ├── __init__.py
│   ├── base.py
│   ├── model_plugin.py
│   ├── dataset_plugin.py
│   └── eval_report_plugin.py
└── tests/
    ├── __init__.py
    ├── test_exceptions.py
    ├── test_types.py
    ├── test_storage_backend.py
    ├── test_registry_client.py
    ├── test_config.py
    ├── test_registry.py
    ├── test_push_dispatch.py
    ├── implementations/
    │   ├── __init__.py
    │   ├── test_mlflow_client.py
    │   └── test_vertex_ai_client.py
    └── plugins/
        ├── __init__.py
        ├── test_model_plugin.py
        ├── test_dataset_plugin.py
        └── test_eval_report_plugin.py
```

Build in dependency order to avoid circular imports:

```
exceptions → types → storage_backend → registry_client
          → config          (needs exceptions)
          → registry        (needs exceptions; TYPE_CHECKING import of plugins.base)
          → plugins/base    (needs storage_backend, registry_client)
          → plugins/model_plugin, dataset_plugin, eval_report_plugin
          → plugins/__init__.py   (imports all plugins + default_registry; populates registry)
          → pusher.py             (imports registry, exceptions, config, types)
          → __init__.py           (imports pusher.py; triggers plugins/__init__.py population)
          → implementations/      (standalone; imported by users, not by push() itself)
          → tests/                (parallel)
```

---

## 3. Code Conventions

Matches the existing `model_manager` module patterns:

| Convention | Rule |
|---|---|
| Future annotations | `from __future__ import annotations` in every file |
| Imports | Absolute only: `from michelangelo.workflow.tasks.pusher.exceptions import ...` |
| Type hints | `list[X]`, `dict[K, V]`, `Optional[X]` — no `X \| None` (Python 3.9 compat) |
| Configs | `@dataclass` with `field(default_factory=...)` for mutable defaults |
| Docstrings | Google-style: `Attributes:`, `Args:`, `Returns:`, `Raises:`, `Example:` |
| Logging | `log = logging.getLogger(__name__)` at module level in plugin files |
| Tests | `unittest.TestCase` subclasses in `test_*.py`; method docstrings start `"""It ..."""` |
| Mocking | `unittest.mock.patch` / `MagicMock` |

---

## 4. Module Designs

### 4.1 `exceptions.py`

```python
class PusherError(Exception):
    """Base class for all pusher exceptions."""

class ArtifactNotFoundError(PusherError):
    """Raised when an artifact named in config is absent from the artifacts dict.

    Args:
        name: The artifact name that was expected.
        available: The artifact names that are actually present.
    """
    def __init__(self, name: str, available: list[str]) -> None:
        super().__init__(f"Artifact '{name}' not found. Available: {available}")

class ConfigurationError(PusherError):
    """Raised when a PusherConfig or PusherPluginConfig is invalid.

    Args:
        message: Human-readable description of the configuration problem.
    """
    def __init__(self, message: str) -> None:
        super().__init__(message)

class PusherPluginError(PusherError):
    """Raised when a plugin's execute() raises an unexpected exception.

    Args:
        artifact_name: The artifact the plugin was processing.
        plugin_name: The name of the plugin that raised.
    """
    def __init__(self, artifact_name: str, plugin_name: str) -> None:
        super().__init__(
            f"Plugin '{plugin_name}' failed for artifact '{artifact_name}'."
        )
```

---

### 4.2 `types.py`

`AssembledModel` contains only what the generic push workflow needs: paths and metadata.
All Uber-specific wrappers (`ModelVariable`, `FeaturePackageVariable`, report fields) are
stripped.

```python
@dataclass
class ModelArtifact:
    """A packaged model artifact ready for upload.

    Attributes:
        path: Local filesystem path to the packaged artifact directory or file.
        metadata: Key-value pairs forwarded to the model registry.
            Common keys: ``model_class``, ``training_framework``, ``schema_version``.
    """
    path: str
    metadata: dict[str, str] = field(default_factory=dict)


@dataclass
class AssembledModel:
    """A trained model with its deployable and raw artifacts, ready for pushing.

    Both artifacts must be fully packaged before passing to the pusher.
    Packaging is the assembler's responsibility (Ray worker with GPU access).
    The pusher only uploads and registers pre-packaged artifacts.

    Attributes:
        raw_model: Raw model package (weights + sample data) for offline validation.
        deployable_model: Serving-ready bundle (e.g., Triton config + weights).
        feature_package_path: Optional local path to a feature transformation package.
            Upload is the provider layer's responsibility.
    """
    raw_model: ModelArtifact
    deployable_model: ModelArtifact
    feature_package_path: Optional[str] = None


@dataclass
class PusherResult:
    """The outcome of a single plugin execution.

    Attributes:
        name: Artifact name from PusherPluginConfig.name.
        plugin: Plugin name that was invoked (e.g., ``"model_plugin"``).
        success: True if the plugin completed without error.
        value: Plugin-specific return data. Empty dict when success is False.
        error: Error description when success is False. None when success is True.
    """
    name: str
    plugin: str
    success: bool
    value: dict[str, Any] = field(default_factory=dict)
    error: Optional[str] = None
```

---

### 4.3 `storage_backend.py`

```python
class StorageBackend(ABC):
    """Abstract base class for artifact storage backends.

    Implementations are infrastructure-specific (local FS, S3, object store).
    After a successful upload(), the returned URI must be passable to download().

    Example implementation::

        class S3StorageBackend(StorageBackend):
            def upload(self, local_path: str, destination_key: str) -> str:
                uri = f"s3://my-bucket/{destination_key}"
                # ... boto3 upload ...
                return uri

            def download(self, uri: str, local_path: str) -> None:
                # ... boto3 download ...
    """

    @abstractmethod
    def upload(self, local_path: str, destination_key: str) -> str:
        """Upload a local file or directory to the storage backend.

        Args:
            local_path: Absolute path to the local file or directory.
            destination_key: Logical key identifying the artifact within this
                backend (e.g., ``"models/my-classifier/v3"``).

        Returns:
            A URI string uniquely identifying the uploaded artifact.

        Raises:
            IOError: If the upload fails.
        """

    @abstractmethod
    def download(self, uri: str, local_path: str) -> None:
        """Download an artifact from the storage backend.

        Args:
            uri: The URI returned by a previous upload() call.
            local_path: Absolute destination path. Parent directory must exist.

        Raises:
            IOError: If the download fails.
            ValueError: If the URI format is not recognized by this backend.
        """


class LocalStorageBackend(StorageBackend):
    """StorageBackend backed by the local filesystem.

    Intended for development, testing, and single-machine workflows.

    Args:
        base_dir: Root directory for artifact storage. Must exist.
            Use ``tempfile.mkdtemp()`` for ephemeral storage.

    Example:
        >>> import tempfile
        >>> backend = LocalStorageBackend(base_dir=tempfile.mkdtemp())
        >>> uri = backend.upload("/tmp/model", "models/classifier/v1")
        >>> backend.download(uri, "/tmp/retrieved_model")
    """

    def __init__(self, base_dir: str) -> None:
        self._base_dir = base_dir

    def upload(self, local_path: str, destination_key: str) -> str:
        # os.makedirs + shutil.copy2 (file) or shutil.copytree (dir)
        # returns os.path.join(self._base_dir, destination_key)
        ...

    def download(self, uri: str, local_path: str) -> None:
        # raises ValueError if not uri.startswith(self._base_dir)
        # shutil.copy2 or copytree back
        ...
```

**Bug fix (§12.2):** `import os` must be at module level (alongside `import shutil`), not
inside `upload()`. The `download()` method uses `os.path` functions and would hit `NameError`
at runtime without it.

---

### 4.4 `registry_client.py`

```python
@dataclass
class RegisteredModel:
    """A model record returned after successful registration.

    Attributes:
        name: Model name in the registry.
        version: Registry-assigned version string (e.g., ``"3"`` or a UUID).
        registry_uri: URI uniquely identifying this model version.
        metadata: Additional key-value pairs stored with the registration.
    """
    name: str
    version: str
    registry_uri: str
    metadata: dict[str, Any] = field(default_factory=dict)


class ModelRegistryClient(ABC):
    """Abstract base class for model registry clients.

    Example implementation::

        class MLflowRegistryClient(ModelRegistryClient):
            def register_model(self, name, artifact_uri, ...) -> RegisteredModel:
                version = self._client.create_model_version(name, artifact_uri)
                return RegisteredModel(name=name, version=version.version,
                                       registry_uri=f"models:/{name}/{version.version}")
    """

    @abstractmethod
    def register_model(
        self,
        name: str,
        artifact_uri: str,
        deployable_artifact_uri: Optional[str] = None,
        schema: Optional[dict[str, Any]] = None,
        metadata: Optional[dict[str, str]] = None,
    ) -> RegisteredModel:
        """Register a model and its artifact URI.

        Args:
            name: Model name to register under.
            artifact_uri: URI of the raw model artifact (from StorageBackend.upload()).
            deployable_artifact_uri: Optional URI of the serving-ready bundle.
            schema: Optional model input/output schema.
            metadata: Optional string key-value pairs stored with the registration.

        Returns:
            A RegisteredModel describing the created registration.

        Raises:
            IOError: If the registry cannot be reached.
            ValueError: If the name is invalid per the registry's naming rules.
        """

    @abstractmethod
    def get_model(self, name: str, version: Optional[str] = None) -> RegisteredModel:
        """Retrieve a model registration.

        Args:
            name: Model name to look up.
            version: Specific version. If None, the registry's ``"latest"`` is returned.

        Returns:
            The RegisteredModel for the requested name and version.

        Raises:
            KeyError: If the model name or version is not found.
            IOError: If the registry cannot be reached.
        """
```

---

### 4.5 `config.py`

**Key design:** `resolved_plugin_name()` validates that exactly one plugin is specified,
fixing the silent ambiguity bug in the internal `which_plugin()` (which returns the first
non-None field without error if multiple are set).

**Bug fix (§12.1):** `_BUILTIN_FIELDS` must be annotated as `ClassVar[tuple[str, ...]]` so
dataclass machinery excludes it from `__init__`, `__repr__`, and `__eq__`.

```python
class DatasetFormat(Enum):
    """Supported output formats for DatasetPusherPlugin.

    Attributes:
        CSV: Comma-separated values text format.
        PARQUET: Columnar binary format (recommended for large datasets).
        JSON: JSON lines format, one record per line.
    """
    CSV = "csv"
    PARQUET = "parquet"
    JSON = "json"


@dataclass
class ModelPluginConfig:
    """Configuration for ModelPusherPlugin.

    Attributes:
        model_name: Name to register the model under. Generated if None.
        description: Optional description stored in the registry.
        extra_metadata: Additional key-value metadata forwarded to the registry.
    """
    model_name: Optional[str] = None
    description: Optional[str] = None
    extra_metadata: dict[str, str] = field(default_factory=dict)


@dataclass
class DatasetPluginConfig:
    """Configuration for DatasetPusherPlugin.

    Attributes:
        destination_path: Local path prefix where the output file is written.
        format: Output format. Defaults to PARQUET.
        partition_by: Optional list of column names to partition the output by.
    """
    destination_path: str = ""
    format: DatasetFormat = DatasetFormat.PARQUET
    partition_by: list[str] = field(default_factory=list)


@dataclass
class EvalReportPluginConfig:
    """Configuration for EvalReportPusherPlugin.

    Attributes:
        report_name: Name for the evaluation report. Generated if None.
        extra_fields: Additional key-value pairs merged into the report document.
    """
    report_name: Optional[str] = None
    extra_fields: dict[str, Any] = field(default_factory=dict)


@dataclass
class PusherPluginConfig:
    """Configuration for a single artifact push within a PusherConfig.

    Exactly one of the typed plugin config fields or the extension pair
    (plugin_name + plugin_config) must be set.

    Attributes:
        name: Artifact identifier matching a key in the artifacts dict.
        model_plugin: Config for ModelPusherPlugin.
        dataset_plugin: Config for DatasetPusherPlugin.
        eval_report_plugin: Config for EvalReportPusherPlugin.
        plugin_name: Name of a provider-registered plugin.
        plugin_config: Raw config dict for the custom plugin.
    """
    name: str
    model_plugin: Optional[ModelPluginConfig] = None
    dataset_plugin: Optional[DatasetPluginConfig] = None
    eval_report_plugin: Optional[EvalReportPluginConfig] = None
    plugin_name: Optional[str] = None
    plugin_config: Optional[dict[str, Any]] = None

    _BUILTIN_FIELDS: ClassVar[tuple[str, ...]] = (
        "model_plugin", "dataset_plugin", "eval_report_plugin"
    )

    def resolved_plugin_name(self) -> str:
        """Return the active plugin name.

        Returns:
            Plugin name string (e.g., ``"model_plugin"``).

        Raises:
            ConfigurationError: If zero or multiple plugins are specified.
        """
        active = [f for f in self._BUILTIN_FIELDS if getattr(self, f) is not None]
        if len(active) > 1:
            raise ConfigurationError(
                f"Artifact '{self.name}' has multiple plugin configs set: {active}. "
                "Exactly one must be specified."
            )
        if len(active) == 1:
            if self.plugin_name is not None:
                raise ConfigurationError(
                    f"Artifact '{self.name}' sets both '{active[0]}' and 'plugin_name'. "
                    "Use exactly one."
                )
            return active[0]
        if self.plugin_name:
            return self.plugin_name
        raise ConfigurationError(
            f"No plugin specified for artifact '{self.name}'. "
            f"Set one of: {list(self._BUILTIN_FIELDS)}, or set plugin_name."
        )

    def resolved_plugin_config(self) -> Any:
        """Return typed config for built-in plugins or raw dict for custom plugins."""
        plugin = self.resolved_plugin_name()
        typed = getattr(self, plugin, None)
        return typed if typed is not None else self.plugin_config


@dataclass
class PusherConfig:
    """Top-level configuration for a push() call.

    Attributes:
        items: Ordered list of artifact push configurations.
    """
    items: list[PusherPluginConfig] = field(default_factory=list)
```

---

### 4.6 `plugins/base.py`

Constructor adds `storage_backend` and `registry_client` as injected arguments —
a breaking change from the internal `PusherPluginBase.__init__(conf, var)` which has no
infrastructure injection. Phase 3 will update internal plugins to accept these.

```python
class PusherPluginBase(ABC):
    """Abstract base class that all pusher plugins must implement.

    Plugins receive infrastructure dependencies via constructor injection,
    keeping plugin logic free of SDK-specific client initialization.
    A new instance is created per artifact per push() invocation.

    Args:
        config: Plugin-specific configuration dataclass or dict.
        artifact: The artifact value to push. May be None for config-only plugins.
        storage_backend: Storage backend for uploading files.
        registry_client: Model registry client. None for non-model plugins.

    Example::

        class MyPlugin(PusherPluginBase):
            def execute(self) -> dict[str, Any]:
                uri = self._storage_backend.upload(self._artifact.path, "key/v1")
                return {"uri": uri}
    """

    def __init__(
        self,
        config: Any,
        artifact: Any = None,
        storage_backend: Optional[StorageBackend] = None,
        registry_client: Optional[ModelRegistryClient] = None,
    ) -> None:
        self._config = config
        self._artifact = artifact
        self._storage_backend = storage_backend
        self._registry_client = registry_client

    @abstractmethod
    def execute(self) -> dict[str, Any]:
        """Execute the push operation for this plugin.

        Returns:
            Dict of plugin-specific result values stored in PusherResult.value.
            Use an empty dict for plugins that produce no structured output.

        Raises:
            Any exception is surfaced to push() and wrapped in PusherPluginError.
        """
```

---

### 4.7 `plugins/dataset_plugin.py`

Artifact type changes from a framework-specific data type to
`list[dict[str, Any]]` — a plain list of records. No external data framework dependency. Provider layers
can subclass and override `execute()` to write via a custom data sink.

```python
log = logging.getLogger(__name__)


class DatasetPusherPlugin(PusherPluginBase):
    """Plugin that writes a dataset to a file sink in a configurable format.

    Accepts a list of records (list of dicts) and writes to the destination
    path specified in the config. Supported formats: CSV, Parquet, JSON Lines.

    Provider layers that need Spark or remote sinks should subclass this plugin
    and override execute() to call their own sink function.

    Args:
        config: DatasetPluginConfig specifying destination_path, format,
            and optional partition_by columns.
        artifact: A list of dicts (records) representing the dataset to write.
        storage_backend: Unused by this built-in implementation. Available
            for subclasses that extend with remote upload after writing.
        registry_client: Unused by this built-in implementation.

    Example:
        >>> plugin = DatasetPusherPlugin(
        ...     config=DatasetPluginConfig(
        ...         destination_path="/tmp/eval_data",
        ...         format=DatasetFormat.PARQUET,
        ...     ),
        ...     artifact=[{"col1": 1, "col2": "a"}, {"col1": 2, "col2": "b"}],
        ... )
        >>> result = plugin.execute()
        >>> print(result["destination_path"], result["num_records"])
    """

    def execute(self) -> dict[str, Any]:
        """Write the dataset artifact to the configured destination path.

        Returns:
            A dict with keys:
            - destination_path: Absolute path to the written output file.
            - format: The format string ("csv", "parquet", or "json").
            - num_records: Number of records written.

        Raises:
            ImportError: If pandas or pyarrow is not installed (required for
                Parquet output).
            IOError: If the destination path is not writable.
        """
        import os

        dest = self._config.destination_path
        fmt = self._config.format
        records = self._artifact

        os.makedirs(dest, exist_ok=True)
        output_path = os.path.join(dest, f"data.{fmt.value}")

        if fmt == DatasetFormat.CSV:
            self._write_csv(records, output_path)
        elif fmt == DatasetFormat.PARQUET:
            self._write_parquet(records, output_path)
        elif fmt == DatasetFormat.JSON:
            self._write_json(records, output_path)

        log.info("Wrote %d records to '%s' (%s).", len(records), output_path, fmt.value)
        return {
            "destination_path": output_path,
            "format": fmt.value,
            "num_records": len(records),
        }

    @staticmethod
    def _write_csv(records: list[dict[str, Any]], path: str) -> None:
        import csv
        if not records:
            return
        with open(path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=list(records[0].keys()))
            writer.writeheader()
            writer.writerows(records)

    @staticmethod
    def _write_parquet(records: list[dict[str, Any]], path: str) -> None:
        import pandas as pd
        pd.DataFrame(records).to_parquet(path, index=False)

    @staticmethod
    def _write_json(records: list[dict[str, Any]], path: str) -> None:
        import json
        with open(path, "w") as f:
            for record in records:
                f.write(json.dumps(record) + "\n")
```

---

### 4.8 `plugins/model_plugin.py`

Stripped from internal: provider-specific artifact download, feature package upload,
checkpoint management, and workflow context utilities.

Added: proper 4-key return dict (internal returns bare `model_name` string).

```python
log = logging.getLogger(__name__)


class ModelPusherPlugin(PusherPluginBase):
    """Plugin that uploads a trained model and registers it in a model registry.

    Uploads both the raw model artifact and the deployable artifact to the
    configured storage backend, then registers the model in the registry.

    Artifacts must already be packaged before this plugin is invoked —
    packaging is an assembler-time concern (Ray worker with GPU/model access).

    Args:
        config: ModelPluginConfig with optional model_name, description, extra_metadata.
        artifact: An AssembledModel with pre-packaged raw and deployable artifacts.
        storage_backend: Backend for uploading artifacts. Required.
        registry_client: Registry for registering the uploaded model. Required.

    Raises:
        ConfigurationError: If storage_backend or registry_client is None.

    Example:
        >>> plugin = ModelPusherPlugin(
        ...     config=ModelPluginConfig(model_name="my-classifier"),
        ...     artifact=assembled_model,
        ...     storage_backend=LocalStorageBackend("/tmp/store"),
        ...     registry_client=my_registry,
        ... )
        >>> result = plugin.execute()
        >>> print(result["model_name"], result["version"])
    """

    def __init__(
        self,
        config: Any,
        artifact: Optional[AssembledModel] = None,
        storage_backend: Optional[StorageBackend] = None,
        registry_client: Optional[ModelRegistryClient] = None,
    ) -> None:
        super().__init__(config, artifact, storage_backend, registry_client)
        if storage_backend is None:
            raise ConfigurationError(
                "ModelPusherPlugin requires a storage_backend. "
                "Pass a StorageBackend implementation to push()."
            )
        if registry_client is None:
            raise ConfigurationError(
                "ModelPusherPlugin requires a registry_client. "
                "Pass a ModelRegistryClient implementation to push()."
            )

    def execute(self) -> dict[str, Any]:
        """Upload model artifacts and register the model.

        Returns:
            A dict with keys:
            - ``model_name``: Name under which the model was registered.
            - ``version``: Registry-assigned version string.
            - ``raw_artifact_uri``: URI of the uploaded raw model artifact.
            - ``deployable_artifact_uri``: URI of the uploaded deployable artifact.
        """
        model_name = getattr(self._config, "model_name", None) or self._generate_name()

        log.info("Uploading raw model artifact for '%s'.", model_name)
        raw_uri = self._storage_backend.upload(
            self._artifact.raw_model.path,
            f"models/{model_name}/raw",
        )

        log.info("Uploading deployable artifact for '%s'.", model_name)
        deployable_uri = self._storage_backend.upload(
            self._artifact.deployable_model.path,
            f"models/{model_name}/deployable",
        )

        log.info("Registering '%s' in model registry.", model_name)
        registered = self._registry_client.register_model(
            name=model_name,
            artifact_uri=raw_uri,
            deployable_artifact_uri=deployable_uri,
            metadata={
                **self._artifact.raw_model.metadata,
                **getattr(self._config, "extra_metadata", {}),
            },
        )

        return {
            "model_name": registered.name,
            "version": registered.version,
            "raw_artifact_uri": raw_uri,
            "deployable_artifact_uri": deployable_uri,
        }

    @staticmethod
    def _generate_name() -> str:
        return f"model-{uuid.uuid4().hex[:8]}"
```

---

### 4.9 `plugins/eval_report_plugin.py`

Takes a plain `dict` and writes it to JSON. No protobuf dependency. Provider layers can subclass and
override `execute()` to post to a remote reporting service.

`tempfile.mkdtemp()` (not hardcoded `/tmp/`) keeps tests clean and safe for concurrent
execution. The caller owns cleanup of the temp directory.

```python
log = logging.getLogger(__name__)


class EvalReportPusherPlugin(PusherPluginBase):
    """Plugin that persists a structured evaluation report as a JSON file.

    Provider layers can subclass this and override execute() to post the report
    to a database or API instead of writing to disk.

    Args:
        config: EvalReportPluginConfig with optional report_name and extra_fields.
        artifact: A dict representing the evaluation report document.
        storage_backend: Unused by this built-in implementation.
        registry_client: Unused by this built-in implementation.

    Example:
        >>> plugin = EvalReportPusherPlugin(
        ...     config=EvalReportPluginConfig(report_name="run-20260519"),
        ...     artifact={"accuracy": 0.93, "f1": 0.91, "num_samples": 1200},
        ... )
        >>> result = plugin.execute()
        >>> print(result["report_name"], result["output_path"])
    """

    def execute(self) -> dict[str, Any]:
        """Write the evaluation report to a JSON file.

        Returns:
            A dict with keys:
            - ``report_name``: Name assigned to the report.
            - ``output_path``: Absolute path to the written JSON file.
            - ``num_keys``: Number of top-level keys in the artifact dict
                (does not count extra_fields or the internal ``_report_name`` key).
        """
        report_name = getattr(self._config, "report_name", None) or self._generate_name()
        extra = getattr(self._config, "extra_fields", {})
        document = {**self._artifact, **extra, "_report_name": report_name}

        output_dir = tempfile.mkdtemp(prefix="michelangelo_reports_")
        output_path = os.path.join(output_dir, f"{report_name}.json")
        with open(output_path, "w") as f:
            json.dump(document, f, indent=2)

        log.info("Wrote evaluation report '%s' to '%s'.", report_name, output_path)
        return {
            "report_name": report_name,
            "output_path": output_path,
            "num_keys": len(self._artifact),
        }

    @staticmethod
    def _generate_name() -> str:
        return f"eval-report-{uuid.uuid4().hex[:8]}"
```

---

### 4.10 `registry.py`

`default_registry` is declared here as an empty `PluginRegistry`. It is populated in
`plugins/__init__.py` to avoid circular imports. The `TYPE_CHECKING` guard on
`PusherPluginBase` breaks the potential circular import cycle.

```python
"""Plugin registry for mapping plugin names to their implementations."""

from __future__ import annotations

from typing import TYPE_CHECKING, Optional

if TYPE_CHECKING:
    from michelangelo.workflow.tasks.pusher.plugins.base import PusherPluginBase

from michelangelo.workflow.tasks.pusher.exceptions import ConfigurationError


class PluginRegistry:
    """Registry mapping plugin names to their implementation class and artifact type.

    The open source library ships a ``default_registry`` pre-populated with
    the three built-in plugins. Provider layers call ``extend()`` to create a
    child registry and add their own plugins without mutating the shared default.

    Lookups fall through to the parent registry when a name is not found
    locally, forming a chain: provider registry → ``default_registry``.

    Args:
        parent: Optional parent registry to inherit registrations from.

    Example::

        from michelangelo.workflow.tasks.pusher.registry import default_registry

        custom_registry = default_registry.extend()
        custom_registry.register("my_custom_plugin", MyCustomPlugin, MyArtifactType)
        plugin_class, artifact_type = custom_registry.get("model_plugin")
    """

    def __init__(self, parent: Optional["PluginRegistry"] = None) -> None:
        self._registry: dict[str, tuple[type["PusherPluginBase"], Optional[type]]] = {}
        self._parent = parent

    def register(
        self,
        name: str,
        plugin_class: type["PusherPluginBase"],
        artifact_type: Optional[type] = None,
    ) -> None:
        """Register a plugin under a given name.

        Args:
            name: Plugin identifier used in ``PusherPluginConfig``.
            plugin_class: Concrete subclass of ``PusherPluginBase``.
            artifact_type: Expected Python type of the artifact value. Pass
                ``None`` for config-only plugins.

        Raises:
            ValueError: If ``name`` is already registered in this instance.
        """
        if name in self._registry:
            raise ValueError(
                f"Plugin '{name}' is already registered. "
                "To override a parent registration, use extend() to create a "
                "child registry and register the override there."
            )
        self._registry[name] = (plugin_class, artifact_type)

    def get(self, name: str) -> tuple[type["PusherPluginBase"], Optional[type]]:
        """Look up a plugin by name, falling through to the parent if needed.

        Raises:
            ConfigurationError: If ``name`` is not found in this registry or
                any ancestor registry.
        """
        if name in self._registry:
            return self._registry[name]
        if self._parent is not None:
            return self._parent.get(name)
        raise ConfigurationError(
            f"No plugin registered under name '{name}'. "
            f"Registered names: {self.registered_names()}."
        )

    def registered_names(self) -> list[str]:
        """Return all plugin names visible from this registry, including parents."""
        names: set[str] = set(self._registry.keys())
        if self._parent is not None:
            names.update(self._parent.registered_names())
        return sorted(names)

    def extend(self) -> "PluginRegistry":
        """Create a child registry that inherits all registrations from this one."""
        return PluginRegistry(parent=self)


# Declared here; populated in plugins/__init__.py to avoid circular imports.
default_registry: PluginRegistry = PluginRegistry()
```

---

### 4.11 `plugins/__init__.py`

The only place in the codebase that calls `default_registry.register()`. Importing this
module (directly or via `pusher/__init__.py`) triggers registration.

```python
"""Built-in plugin registration and public re-exports for the plugins package."""

from michelangelo.workflow.tasks.pusher.plugins.dataset_plugin import DatasetPusherPlugin
from michelangelo.workflow.tasks.pusher.plugins.eval_report_plugin import EvalReportPusherPlugin
from michelangelo.workflow.tasks.pusher.plugins.model_plugin import ModelPusherPlugin
from michelangelo.workflow.tasks.pusher.registry import default_registry
from michelangelo.workflow.tasks.pusher.types import AssembledModel

default_registry.register("model_plugin", ModelPusherPlugin, AssembledModel)
default_registry.register("dataset_plugin", DatasetPusherPlugin, list)
default_registry.register("eval_report_plugin", EvalReportPusherPlugin, dict)

__all__ = [
    "ModelPusherPlugin",
    "DatasetPusherPlugin",
    "EvalReportPusherPlugin",
]
```

---

### 4.12 `pusher.py`

```python
"""Core push() dispatch function."""

from __future__ import annotations

import logging
import tempfile
from typing import Any, Callable, Optional

from michelangelo.workflow.tasks.pusher.config import PusherConfig
from michelangelo.workflow.tasks.pusher.exceptions import (
    ArtifactNotFoundError,
    ConfigurationError,
    PusherPluginError,
)
from michelangelo.workflow.tasks.pusher.registry import PluginRegistry, default_registry
from michelangelo.workflow.tasks.pusher.registry_client import ModelRegistryClient
from michelangelo.workflow.tasks.pusher.storage_backend import LocalStorageBackend, StorageBackend
from michelangelo.workflow.tasks.pusher.types import PusherResult

_logger = logging.getLogger(__name__)


def push(
    config: PusherConfig,
    artifacts: dict[str, Any],
    storage_backend: Optional[StorageBackend] = None,
    registry_client: Optional[ModelRegistryClient] = None,
    registry: Optional[PluginRegistry] = None,
    fail_fast: bool = True,
    on_error: Optional[Callable[[str, str, Exception], None]] = None,
) -> list[PusherResult]:
    """Push one or more artifacts to their configured destinations.

    Args:
        config: Top-level pusher configuration listing the artifacts to push
            and which plugin to use for each.
        artifacts: Mapping from artifact name to artifact value.
        storage_backend: Backend for file uploads. Defaults to a
            ``LocalStorageBackend`` backed by a temporary directory.
        registry_client: Model registry client. Required when any configured
            plugin is a ``ModelPusherPlugin``.
        registry: Plugin registry for resolving plugin names. Defaults to
            ``default_registry``. Pass a child registry via
            ``default_registry.extend()`` to enable additional plugin names.
        fail_fast: When ``True`` (default), raises ``PusherPluginError`` on
            the first plugin failure. When ``False``, collects all results.
        on_error: Optional callback invoked when a plugin raises. Receives
            ``(artifact_name, plugin_name, exception)``.

    Returns:
        List of ``PusherResult``, one per item in ``config.items``, in order.

    Raises:
        ArtifactNotFoundError: If an artifact named in config is absent.
        ConfigurationError: If a plugin name cannot be resolved or artifact
            type does not match.
        PusherPluginError: If ``fail_fast=True`` and any plugin raises.

    Example::

        from michelangelo.workflow.tasks.pusher import push, PusherConfig, PusherPluginConfig
        from michelangelo.workflow.tasks.pusher.config import ModelPluginConfig
        from michelangelo.workflow.tasks.pusher.implementations.mlflow_client import (
            MLflowRegistryClient,
        )

        config = PusherConfig(items=[
            PusherPluginConfig(
                name="classifier",
                model_plugin=ModelPluginConfig(model_name="my-classifier"),
            )
        ])
        results = push(
            config,
            {"classifier": assembled_model},
            registry_client=MLflowRegistryClient("http://mlflow:5000"),
        )
        print(results[0].success, results[0].value["model_name"])
    """
    _registry = registry if registry is not None else default_registry
    _storage = storage_backend or LocalStorageBackend(tempfile.mkdtemp())
    results: list[PusherResult] = []

    for item_config in config.items:
        artifact_name = item_config.name

        if artifact_name not in artifacts:
            raise ArtifactNotFoundError(artifact_name, list(artifacts.keys()))

        plugin_name = item_config.resolved_plugin_name()
        plugin_class, expected_type = _registry.get(plugin_name)
        artifact = artifacts[artifact_name]

        if expected_type is not None and not isinstance(artifact, expected_type):
            raise ConfigurationError(
                f"Plugin '{plugin_name}' expects an artifact of type "
                f"'{expected_type.__name__}', but received "
                f"'{type(artifact).__name__}' for artifact '{artifact_name}'."
            )

        plugin_config = item_config.resolved_plugin_config()
        plugin = plugin_class(
            config=plugin_config,
            artifact=artifact,
            storage_backend=_storage,
            registry_client=registry_client,
        )

        _logger.info("Executing '%s' for artifact '%s'.", plugin_name, artifact_name)
        try:
            value = plugin.execute()
            results.append(
                PusherResult(name=artifact_name, plugin=plugin_name, success=True, value=value)
            )
        except Exception as err:
            if on_error is not None:
                on_error(artifact_name, plugin_name, err)
            if fail_fast:
                raise PusherPluginError(artifact_name, plugin_name) from err
            _logger.exception(
                "Plugin '%s' failed for artifact '%s': %s", plugin_name, artifact_name, err
            )
            results.append(
                PusherResult(
                    name=artifact_name, plugin=plugin_name, success=False, value={}, error=str(err)
                )
            )

    return results
```

---

### 4.13 `__init__.py` (public API surface)

Side-effecting import of `plugins` must come first to populate `default_registry` before
any user code can call `push()`.

```python
# Trigger default_registry population — must come before any other local imports.
import michelangelo.workflow.tasks.pusher.plugins  # noqa: F401

from michelangelo.workflow.tasks.pusher.config import (
    DatasetFormat,
    DatasetPluginConfig,
    EvalReportPluginConfig,
    ModelPluginConfig,
    PusherConfig,
    PusherPluginConfig,
)
from michelangelo.workflow.tasks.pusher.exceptions import (
    ArtifactNotFoundError,
    ConfigurationError,
    PusherError,
    PusherPluginError,
)
from michelangelo.workflow.tasks.pusher.plugins.base import PusherPluginBase
from michelangelo.workflow.tasks.pusher.plugins.dataset_plugin import DatasetPusherPlugin
from michelangelo.workflow.tasks.pusher.plugins.eval_report_plugin import EvalReportPusherPlugin
from michelangelo.workflow.tasks.pusher.plugins.model_plugin import ModelPusherPlugin
from michelangelo.workflow.tasks.pusher.pusher import push
from michelangelo.workflow.tasks.pusher.registry import PluginRegistry, default_registry
from michelangelo.workflow.tasks.pusher.registry_client import ModelRegistryClient, RegisteredModel
from michelangelo.workflow.tasks.pusher.storage_backend import LocalStorageBackend, StorageBackend
from michelangelo.workflow.tasks.pusher.types import AssembledModel, ModelArtifact, PusherResult

__all__ = [
    "push",
    "PusherConfig",
    "PusherPluginConfig",
    "ModelPluginConfig",
    "DatasetPluginConfig",
    "EvalReportPluginConfig",
    "DatasetFormat",
    "AssembledModel",
    "ModelArtifact",
    "PusherResult",
    "PusherPluginBase",
    "StorageBackend",
    "ModelRegistryClient",
    "LocalStorageBackend",
    "RegisteredModel",
    "PluginRegistry",
    "default_registry",
    "PusherError",
    "PusherPluginError",
    "ArtifactNotFoundError",
    "ConfigurationError",
    "ModelPusherPlugin",
    "DatasetPusherPlugin",
    "EvalReportPusherPlugin",
]
```

---

## 5. Concrete `ModelRegistryClient` Implementations

### 5.1 `implementations/mlflow_client.py`

```python
"""MLflow Model Registry client implementation."""

from __future__ import annotations

import logging
from typing import Any, Optional

from michelangelo.workflow.tasks.pusher.registry_client import ModelRegistryClient, RegisteredModel

_logger = logging.getLogger(__name__)


class MLflowRegistryClient(ModelRegistryClient):
    """ModelRegistryClient backed by an MLflow Model Registry.

    Requires ``mlflow`` to be installed::

        pip install "michelangelo[mlflow]"

    Args:
        tracking_uri: MLflow tracking server URI. If ``None``, uses the
            ``MLFLOW_TRACKING_URI`` environment variable, falling back to a
            local ``./mlruns`` directory.
    """

    def __init__(self, tracking_uri: Optional[str] = None) -> None:
        try:
            import mlflow
            from mlflow.tracking import MlflowClient
        except ImportError as exc:
            raise ImportError(
                "MLflowRegistryClient requires the 'mlflow' package. "
                "Install it with: pip install 'michelangelo[mlflow]'"
            ) from exc

        if tracking_uri:
            mlflow.set_tracking_uri(tracking_uri)
        self._client = MlflowClient()
        self._mlflow = mlflow
        _logger.info("MLflowRegistryClient ready (tracking_uri=%s).", mlflow.get_tracking_uri())

    def register_model(
        self,
        name: str,
        artifact_uri: str,
        deployable_artifact_uri: Optional[str] = None,
        schema: Optional[dict[str, Any]] = None,
        metadata: Optional[dict[str, str]] = None,
    ) -> RegisteredModel:
        """Register a model version in MLflow Model Registry.

        Creates the registered model if it does not already exist. The deployable
        artifact URI is stored as the model-version tag ``deployable_artifact_uri``.
        The ``schema`` argument is ignored — MLflow has no native schema field.

        Returns:
            ``RegisteredModel`` with the MLflow integer version string and
            ``"models:/{name}/{version}"`` as ``registry_uri``.

        Raises:
            IOError: If the MLflow tracking server cannot be reached.
            ValueError: If model registration fails.
        """
        tags = dict(metadata or {})
        if deployable_artifact_uri:
            tags["deployable_artifact_uri"] = deployable_artifact_uri

        try:
            try:
                self._client.get_registered_model(name)
            except self._mlflow.exceptions.MlflowException:
                self._client.create_registered_model(name)

            mv = self._client.create_model_version(name=name, source=artifact_uri, tags=tags)
        except Exception as exc:
            raise ValueError(f"MLflow registration failed for model '{name}'.") from exc

        _logger.info("Registered '%s' v%s in MLflow.", name, mv.version)
        return RegisteredModel(
            name=mv.name,
            version=str(mv.version),
            registry_uri=f"models:/{name}/{mv.version}",
            metadata={k: v for k, v in (mv.tags or {}).items()},
        )

    def get_model(self, name: str, version: Optional[str] = None) -> RegisteredModel:
        """Retrieve a model version from MLflow.

        Returns the latest version when ``version`` is ``None``.

        Raises:
            KeyError: If the model name or version is not found.
            IOError: If the MLflow tracking server cannot be reached.
        """
        try:
            if version is not None:
                mv = self._client.get_model_version(name, version)
            else:
                rm = self._client.get_registered_model(name)
                if not rm.latest_versions:
                    raise KeyError(f"Model '{name}' has no versions.")
                mv = self._client.get_model_version(name, rm.latest_versions[0].version)
        except KeyError:
            raise
        except Exception as exc:
            raise KeyError(
                f"Model '{name}' version '{version}' not found in MLflow."
            ) from exc

        return RegisteredModel(
            name=mv.name,
            version=str(mv.version),
            registry_uri=f"models:/{name}/{mv.version}",
            metadata={k: v for k, v in (mv.tags or {}).items()},
        )
```

### 5.2 `implementations/vertex_ai_client.py`

```python
"""Vertex AI Model Registry client implementation."""

from __future__ import annotations

import logging
from typing import Any, Optional

from michelangelo.workflow.tasks.pusher.registry_client import ModelRegistryClient, RegisteredModel

_logger = logging.getLogger(__name__)


class VertexAIRegistryClient(ModelRegistryClient):
    """ModelRegistryClient backed by Google Cloud Vertex AI Model Registry.

    Requires GCP credentials discoverable from the environment::

        pip install "michelangelo[vertex-ai]"

    Args:
        project_id: GCP project ID.
        location: Vertex AI region (default ``"us-central1"``).
        credentials: Optional GCP credentials object.
    """

    def __init__(
        self,
        project_id: str,
        location: str = "us-central1",
        credentials: Optional[Any] = None,
    ) -> None:
        try:
            from google.cloud import aiplatform
        except ImportError as exc:
            raise ImportError(
                "VertexAIRegistryClient requires 'google-cloud-aiplatform'. "
                "Install it with: pip install 'michelangelo[vertex-ai]'"
            ) from exc

        self._aiplatform = aiplatform
        self._project_id = project_id
        self._location = location
        aiplatform.init(project=project_id, location=location, credentials=credentials)
        _logger.info(
            "VertexAIRegistryClient ready (project=%s, location=%s).", project_id, location
        )

    def register_model(
        self,
        name: str,
        artifact_uri: str,
        deployable_artifact_uri: Optional[str] = None,
        schema: Optional[dict[str, Any]] = None,
        metadata: Optional[dict[str, str]] = None,
    ) -> RegisteredModel:
        """Upload and register a model in Vertex AI Model Registry.

        The deployable artifact URI is stored as the label
        ``deployable-artifact-uri`` (hyphens required by Vertex AI label rules,
        values truncated to 63 characters). The ``schema`` argument is ignored.

        Returns:
            ``RegisteredModel`` with the Vertex AI resource ID suffix as
            ``version`` and the full resource name as ``registry_uri``.

        Raises:
            IOError: If the Vertex AI API cannot be reached.
            ValueError: If registration fails.
        """
        labels = dict(metadata or {})
        if deployable_artifact_uri:
            labels["deployable-artifact-uri"] = deployable_artifact_uri[:63]

        try:
            model = self._aiplatform.Model.upload(
                display_name=name,
                artifact_uri=artifact_uri,
                labels=labels,
                serving_container_image_uri=None,
            )
        except Exception as exc:
            raise ValueError(f"Vertex AI registration failed for model '{name}'.") from exc

        resource_id = model.resource_name.split("/")[-1]
        _logger.info("Registered '%s' in Vertex AI (resource_id=%s).", name, resource_id)
        return RegisteredModel(
            name=name,
            version=resource_id,
            registry_uri=model.resource_name,
            metadata=dict(labels),
        )

    def get_model(self, name: str, version: Optional[str] = None) -> RegisteredModel:
        """Retrieve a model from Vertex AI Model Registry.

        When ``version`` (resource ID) is provided, fetches directly by ID.
        Otherwise, lists models by display name and returns the first match.

        Raises:
            KeyError: If no matching model is found.
            IOError: If the Vertex AI API cannot be reached.
        """
        try:
            if version is not None:
                resource_name = (
                    f"projects/{self._project_id}/locations/{self._location}"
                    f"/models/{version}"
                )
                model = self._aiplatform.Model(resource_name)
            else:
                models = self._aiplatform.Model.list(filter=f'display_name="{name}"')
                if not models:
                    raise KeyError(f"Model '{name}' not found in Vertex AI.")
                model = models[0]
        except KeyError:
            raise
        except Exception as exc:
            raise KeyError(f"Model '{name}' not found in Vertex AI.") from exc

        resource_id = model.resource_name.split("/")[-1]
        return RegisteredModel(
            name=model.display_name,
            version=resource_id,
            registry_uri=model.resource_name,
            metadata=dict(model.labels or {}),
        )
```

### 5.3 `implementations/__init__.py`

```python
"""Concrete ModelRegistryClient implementations (optional extras).

- ``MLflowRegistryClient``: pip install "michelangelo[mlflow]"
- ``VertexAIRegistryClient``: pip install "michelangelo[vertex-ai]"
"""

from michelangelo.workflow.tasks.pusher.implementations.mlflow_client import MLflowRegistryClient
from michelangelo.workflow.tasks.pusher.implementations.vertex_ai_client import VertexAIRegistryClient

__all__ = [
    "MLflowRegistryClient",
    "VertexAIRegistryClient",
]
```

### 5.4 `ModelRegistryClient` ABC argument mapping

| Argument | MLflow mapping | Vertex AI mapping |
|---|---|---|
| `name` | Registered model name | Display name |
| `artifact_uri` | MLflow run URI / S3 path | GCS path |
| `deployable_artifact_uri` | Stored as version tag | Stored as label `deployable-artifact-uri` |
| `schema` | Ignored | Ignored |
| `metadata` | Stored as version tags | Stored as labels |
| Return `version` | Integer version string `"3"` | Vertex AI resource ID suffix |
| Return `registry_uri` | `"models:/name/3"` | Full resource name |

---

## 6. Key Design Decisions

| Decision | Rationale |
|---|---|
| `storage_backend` + `registry_client` injected in `__init__` | Plugins are single-use per `push()` call; injection keeps plugin logic free of client initialization. |
| `ModelPusherPlugin.execute()` returns dict (not bare string) | Internal returns `model_name` string only. Dict with 4 keys is more useful and consistent with all other plugins. |
| No HDFS download in OS `ModelPusherPlugin` | HDFS → local copy is Uber-specific. The open source plugin assumes artifacts are already local (assembler's responsibility). |
| `DatasetPusherPlugin` takes `list[dict]`, writes to file | Internal takes a Spark DataFrame via Uber-specific `DataSink`. OS uses plain list of records — no Spark dependency. |
| `DatasetFormat` enum (CSV / Parquet / JSON) | Replaces `DataSink` type list. Infrastructure-agnostic; covers the most common formats. |
| Parquet as default `DatasetFormat` | Best compression and columnar efficiency; pandas + pyarrow already in dev extras. |
| `EvalReportPusherPlugin` takes `dict`, writes JSON | Internal wraps a protobuf `MessageVariable`. OS uses a plain dict — no proto dependency. |
| `tempfile.mkdtemp()` for eval report output dir | Avoids hardcoded `/tmp/`, safe for concurrent runs, test-friendly. |
| `resolved_plugin_name()` raises on ambiguity | Internal `which_plugin()` silently picks first non-None field — a hidden bug. |
| `_BUILTIN_FIELDS` as `ClassVar` | Plain annotation on a dataclass makes it an instance field — appears in `__init__`, `__repr__`, `__eq__`. `ClassVar` excludes it. |
| `import os` at module level in `storage_backend.py` | `download()` uses `os.path` functions; importing inside `upload()` only causes `NameError` at runtime. |
| `on_error` callback in `push()` | Lets provider layers wire metric emission without forking `push()`. |
| `fail_fast=True` default | Matches current internal behavior. |
| `LocalStorageBackend(tempfile.mkdtemp())` as `push()` default | Keeps `push()` usable in unit tests with no setup. |
| Dataclasses not Pydantic | Matches existing lib style. |

---

## 7. `pyproject.toml` Changes

File: `michelangelo/python/pyproject.toml`

**Add under `[tool.poetry.dependencies]`:**

```toml
google-cloud-aiplatform = { version = "^1.38.0", optional = true }
```

**Add under `[tool.poetry.extras]`:**

```toml
pusher    = ["pandas", "pyarrow"]
mlflow    = ["mlflow"]
vertex-ai = ["google-cloud-aiplatform"]
```

`mlflow`, `pandas`, and `pyarrow` are already declared as optional — only
`google-cloud-aiplatform` is a new dependency entry.

**Installation matrix:**

```bash
pip install michelangelo                             # core only
pip install "michelangelo[pusher]"                   # adds Parquet support
pip install "michelangelo[pusher,mlflow]"            # adds MLflow registry client
pip install "michelangelo[pusher,vertex-ai]"         # adds Vertex AI registry client
pip install "michelangelo[pusher,mlflow,vertex-ai]"  # all of the above
```

---

## 8. Test Coverage Plan

All tests use `unittest.TestCase`. Target: **≥90% per file** (enforced by `diff-cover`).

### `test_model_plugin.py` — 8 tests

| Test | What it verifies |
|---|---|
| `test_execute_uploads_both_artifacts_and_registers` | `upload` called twice, `register_model` once, returns 4-key dict |
| `test_execute_uses_config_model_name` | `ModelPluginConfig(model_name="x")` → registered under `"x"` |
| `test_execute_generates_name_when_none` | No `model_name` → name starts with `"model-"` |
| `test_execute_merges_metadata` | Artifact metadata + `extra_metadata` merged into `register_model` call |
| `test_execute_returns_correct_dict_keys` | All four keys present |
| `test_raises_when_storage_backend_none` | `ConfigurationError` raised at `__init__` |
| `test_raises_when_registry_client_none` | `ConfigurationError` raised at `__init__` |
| `test_upload_order` | Raw artifact uploaded before deployable (via `call_args_list`) |

### `test_eval_report_plugin.py` — 7 tests

| Test | What it verifies |
|---|---|
| `test_execute_writes_json_file` | Output file exists, is valid JSON, contains artifact keys |
| `test_execute_uses_config_report_name` | `report_name="my-report"` → filename is `my-report.json` |
| `test_execute_generates_name_when_none` | No `report_name` → filename starts with `"eval-report-"` |
| `test_execute_merges_extra_fields` | `extra_fields` merged into written document |
| `test_execute_returns_correct_dict_keys` | All three keys: `report_name`, `output_path`, `num_keys` |
| `test_execute_num_keys_counts_artifact_only` | `num_keys` counts only artifact keys, not `extra_fields` or `_report_name` |
| `test_execute_report_name_in_document` | `_report_name` key present in written JSON |

### `test_dataset_plugin.py` — 8 tests

| Test | What it verifies |
|---|---|
| `test_execute_writes_csv_file` | CSV file exists, has header row, correct number of data rows |
| `test_execute_writes_parquet_file` | Parquet file exists, readable via pandas, correct shape |
| `test_execute_writes_json_file` | JSON Lines file exists, each line is valid JSON, correct record count |
| `test_execute_uses_config_destination_path` | Output file written under configured `destination_path` |
| `test_execute_returns_correct_dict_keys` | All three keys: `destination_path`, `format`, `num_records` |
| `test_execute_num_records_matches_artifact_length` | `num_records` equals `len(artifact)` |
| `test_execute_creates_destination_dir_if_absent` | Directory created if it doesn't exist |
| `test_execute_empty_artifact_writes_zero_row_parquet` | Empty list → valid zero-row Parquet file |

### `test_registry.py` — 7 tests

| Test | What it verifies |
|---|---|
| `test_register_and_get` | Registered plugin is retrievable by name |
| `test_get_falls_through_to_parent` | Child registry returns parent's registration for unknown name |
| `test_extend_child_overrides_parent` | Same name in child shadows parent without mutating it |
| `test_duplicate_registration_raises` | `ValueError` when same name registered twice in same instance |
| `test_get_unknown_name_raises` | `ConfigurationError` with name listed when no ancestor has the key |
| `test_registered_names_includes_parents` | `registered_names()` returns union of child and ancestor |
| `test_default_registry_populated_after_import` | After import, `default_registry` has all three built-in names |

### `test_push_dispatch.py` — 9 tests

| Test | What it verifies |
|---|---|
| `test_push_success_single_plugin` | Single item, success path, returns 1 `PusherResult` with `success=True` |
| `test_push_success_multiple_plugins` | Multiple items processed in order, all succeed |
| `test_push_artifact_not_found_raises` | `ArtifactNotFoundError` when artifact key absent |
| `test_push_unknown_plugin_name_raises` | `ConfigurationError` when plugin name not in registry |
| `test_push_type_mismatch_raises` | `ConfigurationError` when artifact type doesn't match registration |
| `test_push_fail_fast_true_stops_on_first_error` | First failure raises `PusherPluginError`; second plugin not called |
| `test_push_fail_fast_false_collects_all_results` | All plugins run; failures recorded in `PusherResult.error` |
| `test_push_on_error_callback_called` | `on_error` receives `(artifact_name, plugin_name, exception)` |
| `test_push_uses_custom_registry` | Plugin registered in child registry is resolved correctly |

### `test_config.py` — includes extension plugin config test (§12.4)

```python
def test_resolved_plugin_config_returns_raw_dict_for_extension_plugin(self):
    """It returns plugin_config dict, not None, for provider-registered plugins."""
    raw_cfg = {"database_name": "ml_evals", "table_name": "runs"}
    cfg = PusherPluginConfig(
        name="eval_data",
        plugin_name="custom_hive_plugin",
        plugin_config=raw_cfg,
    )
    self.assertIs(cfg.resolved_plugin_config(), raw_cfg)
```

---

## 9. Example: Boston Housing End-to-End Evaluate + Push

**New file:** `examples/boston_housing_xgb/push_trained_model.py`

Extends the existing Boston housing XGBoost example with evaluate + push steps.
Uses only `LocalStorageBackend` and an inline `InMemoryRegistryClient` — no
infrastructure required.

```python
"""Evaluate and push the Boston Housing XGBoost model using michelangelo.workflow.tasks.pusher.

Run after train_workflow completes:

    PYTHONPATH=. poetry run python \\
        examples/boston_housing_xgb/push_trained_model.py \\
        --model-path /tmp/boston-xgb/model.ubj \\
        --store-dir /tmp/boston-xgb/pushed
"""

from __future__ import annotations

import argparse
import logging
import os
import tempfile
from typing import Any, Optional

import numpy as np
import pandas as pd
import xgboost

from michelangelo.workflow.tasks.pusher import (
    AssembledModel,
    EvalReportPusherPlugin,
    LocalStorageBackend,
    ModelArtifact,
    ModelPusherPlugin,
    PusherResult,
)
from michelangelo.workflow.tasks.pusher.config import EvalReportPluginConfig, ModelPluginConfig
from michelangelo.workflow.tasks.pusher.registry_client import ModelRegistryClient, RegisteredModel

log = logging.getLogger(__name__)


class InMemoryRegistryClient(ModelRegistryClient):
    """Registry client that stores registrations in memory (demo only)."""

    def __init__(self) -> None:
        self._store: dict[str, list[RegisteredModel]] = {}

    def register_model(
        self,
        name: str,
        artifact_uri: str,
        deployable_artifact_uri: Optional[str] = None,
        schema: Optional[dict[str, Any]] = None,
        metadata: Optional[dict[str, str]] = None,
    ) -> RegisteredModel:
        version = str(len(self._store.get(name, [])) + 1)
        entry = RegisteredModel(
            name=name,
            version=version,
            registry_uri=f"memory://{name}/{version}",
            metadata=metadata or {},
        )
        self._store.setdefault(name, []).append(entry)
        log.info("Registered '%s' v%s from %s", name, version, artifact_uri)
        return entry

    def get_model(self, name: str, version: Optional[str] = None) -> RegisteredModel:
        versions = self._store.get(name, [])
        if not versions:
            raise KeyError(f"Model '{name}' not found.")
        return versions[-1] if version is None else next(
            v for v in versions if v.version == version
        )


def evaluate(model_path: str, label_col: str = "target") -> dict[str, Any]:
    """Load trained XGBoost model, evaluate on Boston Housing validation set."""
    data_url = "http://lib.stat.cmu.edu/datasets/boston"
    raw_df = pd.read_csv(data_url, sep=r"\s+", skiprows=22, header=None)
    X = np.hstack([raw_df.values[::2, :], raw_df.values[1::2, :2]])  # noqa: N806
    y = raw_df.values[1::2, 2]
    columns = [
        "CRIM", "ZN", "INDUS", "CHAS", "NOX",
        "RM", "AGE", "DIS", "RAD", "TAX", "PTRATIO", "B", "LSTAT",
    ]
    df = pd.DataFrame(X, columns=columns)
    df[label_col] = y

    rng = np.random.default_rng(1)
    mask = rng.random(len(df)) < 0.25
    val_df = df[mask].reset_index(drop=True)

    feature_cols = [c for c in val_df.columns if c != label_col]
    dval = xgboost.DMatrix(val_df[feature_cols])

    model = xgboost.Booster()
    model.load_model(model_path)
    predictions = model.predict(dval)
    actual = val_df[label_col].values

    rmse = float(np.sqrt(np.mean((predictions - actual) ** 2)))
    mae = float(np.mean(np.abs(predictions - actual)))
    ss_res = float(np.sum((actual - predictions) ** 2))
    ss_tot = float(np.sum((actual - np.mean(actual)) ** 2))
    r2 = 1.0 - ss_res / ss_tot if ss_tot > 0 else 0.0

    return {"rmse": rmse, "mae": mae, "r2": r2, "num_samples": int(len(actual))}


def push_model(
    model_path: str,
    store_dir: str,
    registry: InMemoryRegistryClient,
    model_name: str = "boston-housing-xgb",
) -> PusherResult:
    """Push the XGBoost model using ModelPusherPlugin."""
    artifact = ModelArtifact(
        path=model_path,
        metadata={"framework": "xgboost", "task": "regression"},
    )
    assembled = AssembledModel(raw_model=artifact, deployable_model=artifact)
    plugin = ModelPusherPlugin(
        config=ModelPluginConfig(
            model_name=model_name,
            description="Boston Housing price regression (XGBoost)",
            extra_metadata={"dataset": "boston_housing", "training_framework": "xgboost"},
        ),
        artifact=assembled,
        storage_backend=LocalStorageBackend(base_dir=store_dir),
        registry_client=registry,
    )
    return plugin.execute()


def push_eval_report(
    metrics: dict[str, Any],
    report_name: str = "boston-housing-eval",
) -> PusherResult:
    """Persist evaluation metrics using EvalReportPusherPlugin."""
    plugin = EvalReportPusherPlugin(
        config=EvalReportPluginConfig(
            report_name=report_name,
            extra_fields={"model": "xgboost", "dataset": "boston_housing"},
        ),
        artifact=metrics,
    )
    return plugin.execute()


def main() -> int:
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")

    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--model-path", required=True)
    parser.add_argument("--store-dir", default=tempfile.mkdtemp(prefix="boston_push_"))
    parser.add_argument("--model-name", default="boston-housing-xgb")
    args = parser.parse_args()

    os.makedirs(args.store_dir, exist_ok=True)
    registry = InMemoryRegistryClient()

    print("\n── Evaluating model ──────────────────────────────────────────────")
    metrics = evaluate(args.model_path)
    print(f"  RMSE:        {metrics['rmse']:.4f}")
    print(f"  MAE:         {metrics['mae']:.4f}")
    print(f"  R²:          {metrics['r2']:.4f}")
    print(f"  Val samples: {metrics['num_samples']}")

    print("\n── Pushing model ─────────────────────────────────────────────────")
    push_result = push_model(args.model_path, args.store_dir, registry, args.model_name)
    print(f"  model_name:  {push_result['model_name']}")
    print(f"  version:     {push_result['version']}")
    print(f"  stored at:   {push_result['raw_artifact_uri']}")

    print("\n── Pushing eval report ───────────────────────────────────────────")
    report_result = push_eval_report(metrics, report_name=f"{args.model_name}-eval")
    print(f"  report:      {report_result['output_path']}")

    print("\n✓ Done.")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

**How to run:**

```bash
cd /home/user/michelangelo/python

MODEL_PATH=$(find /tmp/ray_results -name "model.ubj" | head -1)

PYTHONPATH=. poetry run python \
    examples/boston_housing_xgb/push_trained_model.py \
    --model-path "$MODEL_PATH" \
    --store-dir /tmp/boston-pushed \
    --model-name boston-housing-xgb
```

---

## 10. Verification

```bash
cd /home/user/michelangelo/python

# Lint
poetry run ruff format --check michelangelo/workflow/tasks/pusher/
poetry run ruff check michelangelo/workflow/tasks/pusher/

# Tests + coverage (target ≥90% per file)
poetry run pytest michelangelo/workflow/tasks/pusher/tests/ -v \
  --cov=michelangelo/workflow/tasks/pusher \
  --cov-report=term-missing
```

Expected: all tests green, zero ruff violations, ≥90% coverage on all new files.
