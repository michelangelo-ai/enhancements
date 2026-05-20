# RFC-20260427: Michelangelo Control Plane Helm Chart

- **Status:** Accepted
- **Author(s):** <!-- GitHub handles, e.g. @you -->
- **Created:** 2026-04-27
- **Internal ERD:** <!-- uPlan link, if applicable -->

---

## Problem statement

The Michelangelo control plane is deployed today by `sandbox.py` via sequential `kubectl apply` calls on raw YAML files. This approach has three compounding problems:

1. **No standard install UX.** External adopters cannot install Michelangelo with a single command against their own cluster. Every deployment requires understanding and running Python tooling written for Uber's internal k3d workflow.
2. **No upgrade path.** `kubectl apply` on raw YAML provides no diff, no rollback, and no revision history. Incremental upgrades require manual file edits or re-running the full sandbox.
3. **Bare Pods, no self-healing.** Three of the five control plane services (`apiserver`, `envoy`, `worker`) are deployed as bare `Pod` resources — they do not restart on failure and cannot be rolled out incrementally.

## Motivation

Michelangelo is an open-source ML platform. Competing platforms (Kubeflow, Flyte, Ray) are all installable via Helm in a single command. The absence of a Helm chart is the primary friction point for external adoption and the top barrier for contributor onboarding.

Internally, the lack of a standard upgrade mechanism makes sandbox maintenance fragile and means CI must re-create the full sandbox on every sync rather than performing a targeted `helm upgrade`.

## Goals

- `helm install michelangelo ./helm/michelangelo --set metadataStorage.host=... --set objectStorage.endpoint=... --set workflow.endpoint=...` installs the full control plane against any Kubernetes 1.27+ cluster.
- `helm install michelangelo ./helm/michelangelo -f helm/michelangelo/values-k3d.yaml` installs against the local k3d sandbox after `sandbox.py` sets up infrastructure.
- `helm upgrade michelangelo --reuse-values` replaces `sandbox.py`'s per-service redeployment loop.
- All five control plane services are promoted to `Deployment` resources.
- Optional Cadence and Temporal subcharts enable fully self-contained installs.

## Non-goals

- Replacing `sandbox.py` for infrastructure management (MySQL, MinIO, Prometheus, KubeRay, Spark Operator). The chart boundary is the control plane only — it consumes connection values; it does not provision backing services.
- Packaging the observability stack (Prometheus, Grafana) or experimental components (MLflow, Fluent Bit) into the chart.
- Supporting Kubernetes versions below 1.27.

## High-level architecture

The chart introduces a clean two-tier boundary:

```
┌─────────────────────────────────────────────────────────┐
│  helm/michelangelo  (this chart)                        │
│                                                         │
│  apiserver  ─── envoy  ─── ui                           │
│  worker     ─── controllermgr                           │
│  CRDs + RBAC                                            │
│                                                         │
│  optional subcharts:                                    │
│    cadence  (cadence.enabled=true)                      │
│    temporal (temporal.enabled=true)                     │
└─────────────────────────────────────────────────────────┘
         │ consumes connection values for:
┌─────────────────────────────────────────────────────────┐
│  Infrastructure (managed by sandbox.py or user)         │
│  MySQL · MinIO/S3 · Cadence/Temporal · KubeRay · Spark  │
└─────────────────────────────────────────────────────────┘
```

### Chart layout

```
helm/michelangelo/
├── Chart.yaml               # cadence + temporal optional dependencies
├── Chart.lock
├── values.yaml              # production defaults (ClusterIP, empty addresses)
├── values-k3d.yaml          # k3d overrides (NodePort, local infra addresses)
└── templates/
    ├── _helpers.tpl
    ├── NOTES.txt
    ├── crds/
    ├── rbac/
    │   ├── clusterrole.yaml
    │   ├── clusterrolebinding.yaml
    │   └── serviceaccount.yaml
    └── core/
        ├── apiserver-deployment.yaml   # schema-init container, optional TLS
        ├── apiserver-service.yaml
        ├── apiserver-configmap.yaml
        ├── apiserver-schema-init-configmap.yaml
        ├── apiserver-ingress.yaml      # Phase B: raw gRPC ingress
        ├── envoy-deployment.yaml
        ├── envoy-service.yaml
        ├── envoy-configmap.yaml
        ├── envoy-ingress.yaml          # Phase A: gRPC-Web ingress
        ├── ui-deployment.yaml
        ├── ui-service.yaml
        ├── ui-configmap.yaml
        ├── ui-ingress.yaml             # Phase A: HTTP ingress
        ├── worker-deployment.yaml
        ├── worker-configmap.yaml
        ├── controllermgr-deployment.yaml
        ├── controllermgr-configmap.yaml
        ├── controllermgr-service.yaml
        ├── metadata-storage-secret.yaml
        └── object-storage-secret.yaml
```

### Key values interface

```yaml
# Caller must provide these — no defaults
metadataStorage:
  driver: mysql        # "mysql" or "postgres"
  host: ""
  port: 3306
  database: michelangelo
  rootPassword: ""

objectStorage:
  endpoint: ""         # e.g. "s3.amazonaws.com", "minio:9000"
  secure: false

workflow:
  endpoint: ""         # e.g. "cadence.internal:7933", "temporal-frontend:7233"
  engine: cadence      # "cadence" or "temporal"

# Per-service enable toggles (all default true)
apiserver.enabled: true
envoy.enabled: true
ui.enabled: true
worker.enabled: true
controllermgr.enabled: true

# Optional subcharts (both default false)
cadence.enabled: false
temporal.enabled: false
```

## APIs and CRDs

This RFC introduces no new CRDs or gRPC/REST API changes. All existing CRDs are packaged into `templates/crds/` and installed by Helm. The public API surface is unchanged.

**RBAC change:** The ClusterRole uses `resources: ["*"]` for the `michelangelo.api` API group (a private group owned entirely by Michelangelo). This is safe — a wildcard on a private group does not increase blast radius compared to an explicit list. All other API groups (`ray.io`, `apps`, `""`, etc.) retain explicit resource lists.

**Secret rename:** `minio-credentials` → `object-storage-credentials`. Annotated with `helm.sh/resource-policy: keep` to preserve externally injected credentials across upgrades. Requires coordinated changes in `sandbox.py`, `michelangelo-worker.yaml`, and `michelangelo-controllermgr.yaml`.

## Alternatives considered

### Alternative A: Keep raw YAML, add a shell installer script

Package the existing YAML files into a shell script that runs `kubectl apply` in the right order.

**Pros:** No new tooling; minimal change to existing files.

**Cons:** No upgrade path, no rollback, no templating for per-environment values, no ecosystem integration (Artifact Hub, OCI registries). Does not solve the bare Pod problem. External adopters would still need to understand and edit YAML files manually.

**Why not chosen:** Does not meet the goal of a single-command install with standard Helm tooling. Provides no upgrade path.

### Alternative B: Kustomize overlays

Use Kustomize `base/` + per-environment overlays instead of Helm.

**Pros:** No Go templates; pure YAML with strategic merge patches.

**Cons:** No built-in dependency management for subcharts (Cadence, Temporal). No `helm upgrade` semantics. Less adoption in the ML platform ecosystem compared to Helm.

**Why not chosen:** The ML platform ecosystem (Kubeflow, Flyte, Ray) converges on Helm. Cadence and Temporal only publish official Helm charts — composing them via Kustomize requires vendoring and maintaining fork copies.

## Open questions

- [ ] Should the chart support Kubernetes Namespaced Roles (in addition to ClusterRole) for users who want to restrict the controller to a single namespace? (`watchNamespace` values key is stubbed; implementation is not scoped to this RFC.)
- [ ] What is the correct versioning strategy for the chart (`version:` in `Chart.yaml`) relative to the `appVersion`? Should chart version track app releases 1:1 or follow its own semver?
- [ ] Should `helm.sh/resource-policy: keep` also apply to the `metadata-storage-secret`, or only to `object-storage-credentials`?

## Rollout strategy

### Phase 1 — CI gate (no chart files)
Add `.github/workflows/helm-lint.yaml` triggered on `paths: ['helm/**']`. Runs `helm dependency update`, `helm lint`, and 6 `helm template` scenarios including subchart-enabled and mutual-exclusivity guard tests.

### Phase 2 — `michelangelo` chart
Create `helm/michelangelo/` with all templates. Migrate all five control plane services from raw YAML to Helm-managed `Deployment` resources. Add schema-init container on `apiserver`. Validate with `helm install` against local k3d.

### Phase 3 — Observability + Experimental documentation (no chart changes)
Document the infrastructure boundary explicitly in `docs/contributing/dev-environment.md`. Confirm Prometheus, Grafana, MLflow, and Fluent Bit remain managed by `sandbox.py`.

### Phase 4 — `sandbox.py` integration
Replace `_deploy_services()` with `helm install michelangelo -f values-k3d.yaml`. Replace `_sync()` app redeployment with `helm upgrade michelangelo --reuse-values`. Read NodePorts from `values-k3d.yaml` at cluster-create time to eliminate the duplicate hardcoded port list.

**Rollback:** Each phase is an independent PR. Rolling back a phase means reverting the PR. Phase 4 is the highest risk — if Helm upgrade overhead causes CI timeouts (current limit: 90 min), revert Phase 4 and investigate before re-landing.

## References

- [Internal design doc: `docs/contributing/system-design/michelangelo-helm.md`](../michelangelo/docs/contributing/system-design/michelangelo-helm.md)
- [Cadence Helm chart](https://github.com/cadence-workflow/cadence-charts)
- [Temporal Helm chart](https://github.com/temporalio/helm-charts)
- [Kubeflow Pipelines Helm install](https://www.kubeflow.org/docs/components/pipelines/operator-guides/installation/)
- [Flyte Helm chart](https://github.com/flyteorg/flyte/tree/master/charts)
