# RFC-20260523: Integration Test Environment Management

- **Status:** Draft
- **Author(s):** <!-- GitHub handles -->
- **Created:** 2026-05-23
- **Source doc:** `michelangelo/docs/contributing/system-design/integration-test-env-proposal.md`

---

## Problem statement

Integration tests run daily on a GCP k3d runner, but three compounding problems degrade reliability and block OSS adoption:

1. **Unreproducible runs** — control plane images default to `:latest`/`:main`. If the tag moves between build and test, the SDK and control plane diverge. Failures cannot be reproduced by SHA checkout.
2. **Undocumented CI configuration** — four ad-hoc `--set` flags in the CI workflow exist nowhere in the chart docs. No single file declares what the CI environment looks like.
3. **Private test repo blocks contributors** — external contributors cannot see, run, or add integration tests. `runs-on: [self-hosted, linux, gcp]` means forks receive zero integration signal.

Additionally, there is no story for provisioning a `RayCluster` in `ma-dev-test` for lifecycle tests.

## Motivation

Michelangelo is an open-source ML platform comparable with Kubeflow, Flyte, and Ray — all of which keep integration tests in the main repo and provide per-PR CI that runs on forks. Every day the private test repo exists, external contributors cannot verify their changes work end-to-end. The four P0/P1 bugs found in the current test code (hardcoded internal job IDs, wrong default branch, wrong env var in README, silently skipped `test_deployment`) damage OSS reputation as soon as the repo becomes visible.

## Goals

- A single `values-integration-test.yaml` file is the canonical declaration of CI environment configuration — no `--set` flags scattered across CI workflows.
- All control plane images are pinned to the triggering commit SHA for every CI run.
- Integration tests live in `tests/integration/` in the main repo; external contributors can run the smoke tier locally with `make integration-smoke`.
- A `helm/ray/` chart manages `RayCluster` + Ray History Server as composable Layer-0 infra for both sandbox and CI environments.
- A per-PR smoke tier runs on `ubuntu-latest + k3d` with zero GCP cost, available to forks via maintainer label.

## Non-goals

- A separate `helm/michelangelo-integration-test` chart — every CI-only divergence is a values toggle, not a template-shape change.
- Managing MySQL, MinIO, or Cadence inside the Ray chart — these are external infrastructure.
- GPU lifecycle tests in the smoke tier — these belong in Tier 2 nightly only.
- Full Uber-internal test migration (private overlay extraction) in Sprint 1.

## High-level architecture

### Proposal A — Helm values layering

```
values.yaml                              ← base (production defaults)
  └── values-k3d.yaml                    ← local dev sandbox
        └── values-integration-test.yaml ← CI overrides (new)
  └── values-gke-staging.yaml            ← future: GKE staging
```

The four ad-hoc `--set` flags are replaced by `helm/michelangelo/values-integration-test.yaml`:

```yaml
# CI integration test overrides. Layer on top of values-k3d.yaml.
# helm upgrade --install michelangelo ./helm/michelangelo \
#   -f values-k3d.yaml -f values-integration-test.yaml \
#   --set controlPlaneTag=${SHA}

apiserver:
  metadataStorage:
    enable: true
  service:
    type: NodePort
    nodePort: 30009
  resources:
    requests: { cpu: 200m, memory: 512Mi }
    limits:   { cpu: 2,    memory: 2Gi }

objectStorage:
  existingSecret: minio-credentials

controllermgr:
  metadataStorage:
    enable: true
  jobs:
    k8sengine:
      mapper:
        logPersistence:
          enabled: false

worker:
  replicas: 1
  resources:
    requests: { cpu: 200m, memory: 512Mi }
    limits:   { cpu: 2,    memory: 4Gi }

images:
  pullPolicy: IfNotPresent   # SHA tags are immutable; Always causes unnecessary re-pulls
```

### Proposal B — Ray cluster Helm chart (`helm/ray/`)

A new chart managing **only** `RayCluster` CRD + Ray History Server. MySQL, MinIO, and Cadence are always external.

```
helm/ray/
  Chart.yaml
  values.yaml                      ← base OSS defaults
  values-sandbox.yaml              ← local dev (CPU workers, History Server on MinIO)
  values-integration-test.yaml     ← CI (CPU workers, History Server disabled)
  templates/
    raycluster.yaml
    history-server-deployment.yaml
    history-server-service.yaml
    history-server-ingress.yaml    ← optional
    rbac.yaml
```

Two-chart composition:

```bash
# Step 1: Ray (warm-state between CI runs, installed once per environment)
helm install michelangelo-ray ./helm/ray \
  -f helm/ray/values-integration-test.yaml -n ma-dev-test

# Step 2: Control plane (every CI run)
helm upgrade --install michelangelo ./helm/michelangelo \
  -f helm/michelangelo/values-k3d.yaml \
  -f helm/michelangelo/values-integration-test.yaml \
  --set controlPlaneTag=${SHA}
```

### Proposal C — Integration tests in main repo

Move `functional/` + `fixtures/` from the private `michelangelo-ai/integration-test` repo to `tests/integration/` in the main repo. Mark provider-specific tests `@pytest.mark.internal` and move them to a private overlay loaded only in internal CI.

### Test tiers

| Tier | Trigger | Infrastructure | Tests | Duration |
|---|---|---|---|---|
| **1 — Smoke** | Every PR (incl. forks, gated by label for forks) | `ubuntu-latest` + k3d | CRUD + pipeline_train + bert-cola CLI + Ray health check | ~10 min |
| **2 — Full nightly** | Daily 03:00 UTC | Self-hosted GCP k3d runner | Full CRUD + MySQL + CLI + lifecycle | ~2 hr |
| **3 — Fork opt-in** | Maintainer applies label | GCP runner | Tier 1 on fork HEAD | ~10 min |

## APIs and CRDs

No new CRDs or API changes. Changes are confined to:
- `helm/michelangelo/values-integration-test.yaml` (new file)
- `helm/ray/` (new chart)
- `.github/workflows/integration-test.yml` (modified)
- `tests/integration/` (migrated from private repo)

## Alternatives considered

### Alternative A: Separate `helm/michelangelo-integration-test` chart

**Pros:** Complete isolation; CI config cannot bleed into production chart.  
**Cons:** Template drift — any control plane change must be applied twice. Signals to the OSS community that CI is special-cased rather than a standard overlay. Diff between environments becomes non-obvious.  
**Why not chosen:** Every CI-only difference is a values toggle. One chart with layered values files matches how Kubeflow, Argo, and Flyte handle the same problem.

### Alternative B: Keep integration tests in a separate (public) repo

**Pros:** Smallest change from current state.  
**Cons:** Tests drift from code — a PR that changes an API must update two repos. External contributors still face a two-repo workflow. No OSS ML platform (MLflow, KFP, KubeRay, Flyte) uses a separate integration-test repo; all maintain tests in the main repository.  
**Why not chosen:** Misaligned with OSS norm. One-PR = code + test is the contributor experience that makes the ecosystem work.

### Alternative C: `helm/ray/` in a separate `michelangelo-ai/helm-ray` repo

**Pros:** Independent Ray chart release cadence; reusable by other teams.  
**Cons:** Every control-plane + Ray config change requires two PRs. Version coupling between Michelangelo and the Ray chart becomes implicit and undocumented.  
**Why not chosen:** For Phase 1, tight lifecycle coupling between control plane and Ray cluster outweighs the flexibility of a separate repo. The chart can be extracted later if independent versioning becomes a requirement.

## Open questions

- [ ] **Ray chart repo location** — `helm/ray/` in the michelangelo repo vs `michelangelo-ai/helm-ray`. Does the Ray chart version need to track Ray upstream independently of the Michelangelo release cycle?
- [ ] **Temporal CI lane** — Add a parallel Temporal nightly job now (cheap: just pass `--workflow temporal` to the same values) or wait until staging cuts over?
- [ ] **`ma sandbox sync` forward compatibility** — Forward `--values` flag to `helm upgrade --install` inside `sandbox sync`, or replace `sandbox sync` with a direct Helm call in CI?
- [ ] **Uber-internal test overlay** — What is the extraction strategy for `@pytest.mark.internal` tests before the main-repo migration? This gates Sprint 2.
- [ ] **Layer-0 infra bundle** — Codify MySQL + MinIO + Cadence bootstrap as `helm/infra/` vs keep as VM state managed by `sandbox.py`?
- [ ] **TestPyPI SDK publishing** — Publish to TestPyPI now to unblock external SDK PRs, or wait?

## Rollout strategy

### Sprint 1 — Reproducibility and config (low risk, ~3–5 days)
- Create `helm/michelangelo/values-integration-test.yaml` (replaces 4 `--set` flags)
- Apply workflow diff: SHA-based image pinning using `github.event.workflow_run.head_sha` (not `github.sha`) with `pullPolicy: IfNotPresent`
- **Fix scheduled nightly gap:** nightly cron trigger must also pin to a SHA (e.g., latest green commit on `main`) — defaulting to `:latest` re-introduces P1
- Fix `DEFAULT_BRANCH = 'master'` → `'main'` in `common_fixture.py`
- Fix README `API_SERVICE_NAME` → `MA_API_SERVER`

### Sprint 2 — Ray chart and test repo migration (~1–2 weeks)
- Design Uber-internal test overlay strategy **before starting** (2–4 hour decision session — gates all migration work)
- Kick off `helm/ray/` chart: align scope, implement `values-sandbox.yaml` + `values-integration-test.yaml`
- Move `tests/integration/` into main repo; apply `@pytest.mark.internal` to provider-specific tests
- Add per-PR smoke job (`ubuntu-latest` + k3d, no GCP cost)
- Remove/replace hardcoded provider-specific job IDs in `check_prerequisites.py`

### Sprint 3 — Full DX and lifecycle (~1–2 weeks)
- Add `helm/infra/` bootstrap bundle (MySQL + MinIO + Cadence) for reproducible Layer-0 setup
- Fix `test_deployment` skip — expose `DeploymentService` in the OSS `APIClient`
- Add `.devcontainer/` configuration for Codespaces (zero-install contributor path)
- Add `make integration-smoke` Makefile target
- Add Temporal CI lane (parallel nightly job, cheap)
- Create `tests/TIER_MANIFEST.md` enumerating which tests run in which tier

### Rollback
Each sprint is an independent PR set. Sprint 1 is purely additive (new file + workflow diff) — rollback is reverting one PR. Sprint 2 migration uses an atomic cutover (new CI job reads from main repo; old job is disabled in the same PR). Sprint 3 is additive only.

### Verification

```bash
# Tier 1 smoke (runs locally, no GCP)
make integration-smoke

# Tier 2 nightly (simulate locally)
helm upgrade --install michelangelo ./helm/michelangelo \
  -f helm/michelangelo/values-k3d.yaml \
  -f helm/michelangelo/values-integration-test.yaml \
  --set controlPlaneTag=$(git rev-parse HEAD)
pytest tests/integration/ -m "not internal" -v

# Verify image SHA pinning
kubectl get pods -n ma-dev-test -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' \
  | grep -v "sha-$(git rev-parse HEAD)" && echo "DRIFT DETECTED" || echo "OK"
```

## References

- Source proposal: `michelangelo/docs/contributing/system-design/integration-test-env-proposal.md`
- Agent findings: `architect-findings.md`, `researcher-findings.md`, `dev-advocate-findings.md`, `coder-findings.md` (repo root, gitignored)
- [Helm chart-testing `ci/` convention](https://github.com/helm/chart-testing)
- [KubeRay Helm chart structure](https://github.com/ray-project/kuberay/tree/master/helm-chart)
- [KFP integration tests in main repo](https://github.com/kubeflow/pipelines/tree/master/test)
- [Flyte integration test approach](https://github.com/flyteorg/flytekit/tree/master/tests)
