# RFC-20260630: Michelangelo LLM Gateway — Pluggable Manager Design

- **Status:** Approved
- **Author(s):** @sally-lee
- **Created:** 2026-06-30

---

## Abstract

This RFC proposes a pluggable `LLMGatewayManager` interface for Michelangelo's control plane, together with `LLMGatewayManagerLiteLLM` as the first-party reference implementation. The interface gives Michelangelo a standard contract for provisioning virtual API keys, enforcing token budgets, syncing spend data, and checking gateway health — without binding the control plane to any specific gateway product. Organizations already running LiteLLM, Portkey, OpenRouter, or a custom OpenAI-compatible proxy can register their own implementation and gain full Michelangelo project-lifecycle integration immediately.

---

## Problem statement

ML engineers building LLM-powered pipelines on Michelangelo today face a fragmented and manual experience across three dimensions:

**1. Key management is ad-hoc and ungoverned.**
Teams manage provider API keys (OpenAI, Anthropic, Vertex AI, etc.) outside the platform — stored in environment variables, shared informally, and rotated (if at all) by hand. Michelangelo has no mechanism to issue, rotate, revoke, or audit LLM credentials at the project level. When a team rotates a key, pipelines silently fail until someone manually updates the secret. There is no audit trail.

**2. LLM inference has no standardized platform entrypoint.**
Teams wire provider SDKs directly into pipeline code, bypassing the Michelangelo control plane entirely. As a result, the platform cannot enforce token budgets, observe cost attribution per project, or allow teams to swap providers or models without code changes. Every team solves the same wiring problem independently, with no shared infrastructure.

**3. Gateway diversity is an adoption blocker.**
Organizations evaluating Michelangelo OSS already operate LLM gateway infrastructure — LiteLLM, Portkey, OpenRouter, Bifrost, and custom OpenAI-compatible proxies are all in production use across the industry. A platform that mandates a specific gateway, or provides no gateway integration, forces adopters to choose between Michelangelo and their existing investment. The platform must meet users where they are.

These three problems compound: a team running an ad-hoc key workflow has no incentive to adopt a platform that provides no governance for those keys. A team that can't observe LLM cost per project cannot make resource allocation decisions. And an adopter that must abandon their existing gateway gets nothing from Michelangelo's control plane for LLM workloads.

## Motivation

LLM-powered pipelines are increasingly central to ML platform use cases — feature engineering, training data synthesis, model evaluation, and inference serving all incorporate LLM calls today. Michelangelo's pipeline abstraction is well-suited to orchestrate this work, but the platform has no first-class model for the LLM layer: no key lifecycle, no spend attribution, no standard entrypoint, no gateway abstraction.

Competing platforms are moving. Kubeflow, Flyte, and Ray Serve are each developing or adopting LLM gateway integration patterns. Michelangelo's advantage is its project-and-team ownership model, which maps cleanly onto the per-project virtual key and budget concepts that OSS gateways like LiteLLM already implement. The control plane work to wire these together is modest; the value to adopters is high.

The design proposed here introduces a single Go interface — `LLMGatewayManager` — as the boundary between Michelangelo's control plane and the LLM gateway layer. The interface is intentionally narrow: six methods, no gateway-specific types. `LLMGatewayManagerLiteLLM` ships as the default implementation. Community implementations for Portkey, OpenRouter, and generic OpenAI-compatible proxies follow the same suffix-named pattern and can be registered without forking the core.

From a pipeline author's perspective, the change is invisible. Two environment variables — `MICHELANGELO_LLM_GATEWAY_URL` and `MICHELANGELO_LLM_VIRTUAL_KEY` — are injected at execution time. The pipeline uses the standard OpenAI SDK against them. No gateway-specific code, no key management, no budget tracking. The control plane handles everything.

## Goals

- A `LLMGatewayManager` Go interface in `pkg/llmgateway/` that any gateway backend can implement, with `ErrNotSupported` for graceful degradation of optional methods.
- `LLMGatewayManagerLiteLLM` as the first-party reference implementation, covering all six interface methods against LiteLLM's Admin API.
- An `LLMGateway` CRD (`michelangelo.ai/v1alpha1`) with `managerName`, `adminKeyRef`, `defaultKeyPolicy`, and `status` fields. Gateway-agnostic — `managerName` selects the implementation at runtime.
- `MICHELANGELO_LLM_GATEWAY_URL` and `MICHELANGELO_LLM_VIRTUAL_KEY` injected into pipeline execution environments when a project has an `LLMGateway` attached.
- A `LLMGatewayManagerGeneric` fallback that works with any OpenAI-compatible proxy using only `GET /health` and `GET /models`.
- Normalized Prometheus metrics under `michelangelo_llm_*` for per-project token usage, cost, and gateway health.
- A `writing-a-manager.md` guide enabling external contributors to ship community implementations as separate Go modules.

## Non-goals

- Deploying or managing any specific gateway product. `LLMGatewayManagerLiteLLM` manages keys and syncs usage via LiteLLM's Admin API; it does not install or upgrade LiteLLM. Gateway deployment is the operator's responsibility.
- Replacing or wrapping the provider inference path. Michelangelo manages the control plane (keys, budgets, observability); pipeline code calls the gateway's `/v1/chat/completions` endpoint directly.
- Multi-region federation of gateway endpoints. Initial design supports one `LLMGateway` per project. Cross-region routing is deferred to a future RFC.
- Guardrail enforcement (PII masking, content moderation). The interface has extension points for these; the implementation is out of scope for this RFC.
- Modifying the existing Michelangelo Helm chart. The LLM Gateway controller ships as a new optional component, deployable via `llmGateway.enabled: true` in `helm/michelangelo/values.yaml`.

## High-level architecture

The registry is keyed by `"namespace/name"` of the `LLMGateway` CRD — one `LLMGatewayManager` instance per gateway, shared across all projects that reference it. Multiple projects independently call `IssueKey()` and each receive their own `sk-...` virtual key with their own budget.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     MICHELANGELO CONTROL PLANE                       │
│                                                                      │
│  ┌─────────────────┐    ┌───────────────────────────────────────┐    │
│  │ Project         │    │        LLM Gateway Controller         │    │
│  │ Management      │───▶│                                       │    │
│  │ (API / UI)      │    │  LLMGatewayReconciler                 │    │
│  └─────────────────┘    │    → Health() every 10 min           │    │
│                         │    → update CRD status.conditions     │    │
│  Gateway resolved via:  │                                       │    │
│  1. explicit gatewayRef │  KeySyncReconciler  (watches Project) │    │
│  2. default annotation  │    → resolveGateway(project)         │    │
│  3. skip (no gateway)   │    → IssueKey / RevokeKey / Rotate   │    │
│                         │    → write sk-... to secret store     │    │
│                         │    → inject env vars at exec time     │    │
│                         │                                       │    │
│                         │  UsageSyncReconciler                  │    │
│                         │    → SyncUsage() every 5 min         │    │
│                         │                                       │    │
│                         │  Registry (singleton, thread-safe)   │    │
│                         │    key: "namespace/name" of CRD      │    │
│                         │    shared across all referencing      │    │
│                         │    projects — one manager per GW     │    │
│                         │  "ma-system/platform-default"        │    │
│                         │      → LLMGatewayManagerLiteLLM      │    │
│                         │  "ml-team/custom-gw"                 │    │
│                         │      → LLMGatewayManagerPortkey      │    │
│                         └───────────────────────────────────────┘    │
└─────────────────────────────────────┬────────────────────────────────┘
                                      │ HTTPS/mTLS · Bearer admin key
                                      │ ClusterIP — intra-cluster only
                                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│             DATA PLANE  (shared gateway, multiple projects)          │
│                                                                      │
│  LLMGateway: michelangelo-system/platform-default                   │
│  annotation: michelangelo.ai/default-gateway: "true"                │
│  annotation: michelangelo.ai/shared: "true"                         │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  LiteLLM Proxy (one instance)                                   │ │
│  │  Project: ml-ranking   → sk-aaa  ($200/mo budget)              │ │
│  │  Project: ml-ads-ctr   → sk-bbb  ($100/mo budget)              │ │
│  │  Project: ml-search    → sk-ccc  ($500/mo budget)              │ │
│  │  ← same manager instance, separate virtual keys per project    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  Pipeline env vars (injected by Michelangelo, standard across all): │
│  MICHELANGELO_LLM_GATEWAY_URL + MICHELANGELO_LLM_VIRTUAL_KEY        │
└──────────────────────────────────────────────────────────────────────┘
```

For the full interface definition, CRD spec, reference implementation sketch, security model (mTLS + NetworkPolicy), and phased rollout plan, see the [design document](https://vibe-mcp.uberinternal.com/v/webdocs/#/d/litellm-michelangelo-integration-oss-proposal).

## APIs and CRDs

### New CRD: `LLMGateway` (`michelangelo.ai/v1alpha1`)

The CRD is **namespace-scoped** — consistent with all existing Michelangelo CRDs. A gateway in namespace A can be referenced by projects in namespace B by adding the `michelangelo.ai/shared: "true"` annotation. Without that annotation, cross-namespace references are rejected by the controller.

```yaml
apiVersion: michelangelo.ai/v1alpha1
kind: LLMGateway
metadata:
  name: platform-default
  namespace: michelangelo-system
  annotations:
    michelangelo.ai/shared: "true"           # allows cross-namespace project references
    michelangelo.ai/default-gateway: "true"  # auto-assigned to projects with is_generative_ai=true
spec:
  managerName: litellm          # selects LLMGatewayManager implementation
  endpoint: "https://..."       # gateway base URL
  adminKeyRef:
    secretName: litellm-admin-key
    key: master-key
  defaultKeyPolicy:
    monthlyBudgetUSD: 500.0
    hardBudgetEnforcement: false
    tpmLimit: 100000
    allowedModels: ["gpt-4o", "gemini-2.0-flash"]
```

**Shared gateway rules:**
- `michelangelo.ai/shared: "true"` is required for a project in namespace B to reference a gateway in namespace A.
- Cross-namespace refs use the format `[namespace/]name` in `gatewayRef` (e.g., `michelangelo-system/platform-default`). Same-namespace refs use just `name`.
- A gateway without the shared annotation is namespace-private.

**Default gateway rules:**
- Only one gateway per cluster may carry `michelangelo.ai/default-gateway: "true"`. The controller validates uniqueness.
- The default gateway must live in `michelangelo-system` (the platform namespace).
- Auto-assignment targets projects where `ProjectSpec.TypeInfo.IsGenerativeAi == true` **and** `ProjectSpec.LLMGatewayRef` is unset.

### Proto addition: `LLMGatewayRef` in `ProjectSpec`

```protobuf
// project.proto — field 11 added to ProjectSpec
message LLMGatewayRef {
  // gateway_ref is "[namespace/]name" of the LLMGateway CRD.
  // Omit namespace for same-namespace; include for cross-namespace (requires shared annotation).
  string gateway_ref = 1;

  // key_policy overrides the gateway's defaultKeyPolicy for this project.
  // If unset, the gateway's defaultKeyPolicy applies.
  KeyPolicy key_policy = 2;
}

message ProjectSpec {
  // ... existing fields 1-10 ...
  LLMGatewayRef llm_gateway_ref = 11;
}
```

### `KeySyncReconciler` gateway resolution

```go
func (r *KeySyncReconciler) resolveGateway(ctx context.Context, project *v1.Project) (*v1alpha1.LLMGateway, error) {
    // 1. Explicit reference from project spec.
    if ref := project.Spec.LLMGatewayRef.GetGatewayRef(); ref != "" {
        return r.lookupGateway(ctx, project.Namespace, ref)
    }

    // 2. Default gateway — only for generative AI projects.
    if project.Spec.TypeInfo.GetIsGenerativeAi() {
        gw, err := r.findDefaultGateway(ctx)
        if err == nil {
            return gw, nil
        }
        // No default gateway configured: skip without error.
    }

    // 3. Skip — project has no LLM gateway.
    return nil, nil
}
```

### New Go interface: `pkg/llmgateway/manager.go`

```go
type LLMGatewayManager interface {
    IssueKey(ctx context.Context, project ProjectRef, policy KeyPolicy) (KeyHandle, error)
    RevokeKey(ctx context.Context, handle KeyHandle) error
    RotateKey(ctx context.Context, old KeyHandle, policy KeyPolicy) (KeyHandle, error)
    Health(ctx context.Context) (GatewayHealth, error)
    SyncUsage(ctx context.Context, since time.Time) (UsageReport, error)
    ListModels(ctx context.Context) ([]ModelSpec, error)
}
```

`ErrNotSupported` may be returned by any method. Callers treat it as a signal to skip — not an error condition. The registry is keyed by `"namespace/name"` of the `LLMGateway` CRD. One `LLMGatewayManager` instance is created per gateway and reused across all projects that reference it. The registry is protected by `sync.RWMutex`.

### Pipeline environment variables

| Variable | Description |
|---|---|
| `MICHELANGELO_LLM_GATEWAY_URL` | Base URL of the attached gateway's inference endpoint |
| `MICHELANGELO_LLM_VIRTUAL_KEY` | Project-scoped virtual key issued by Michelangelo |

## Alternatives considered

<!-- To be filled in by the enhancement-design team (Researcher agent, Task #2). Pending OSS ecosystem comparison of how Kubeflow, Flyte, and Ray approach LLM gateway integration. -->

### Alternative A: Hard-code LiteLLM as the only supported gateway

**Pros:** Simpler implementation; no registry, no interface abstraction layer.  
**Cons:** Forces adopters already running Portkey, OpenRouter, or a custom proxy to either abandon their gateway or bypass Michelangelo's LLM integration entirely. Defeats the OSS-first principle.  
**Why not chosen:** Gateway diversity is a stated adoption requirement. The interface adds one package of indirection; the benefit — any gateway can integrate without forking the core — is substantial.

### Alternative B: Embed gateway management directly in the project controller

**Pros:** No new CRD; LLM lifecycle is a property of the Project rather than a separate resource.  
**Cons:** Couples the project reconcile loop to gateway availability. A degraded gateway causes project reconciliation to stall. Forces all projects to have an LLM gateway opinion even when they don't use one.  
**Why not chosen:** The `LLMGateway` CRD as a separate resource allows independent health tracking, optional adoption, and clean separation of concerns — consistent with how the existing chart separates the control plane from infrastructure.

## Open questions

- [ ] **Interface versioning** — Should the interface carry a version marker now, or wait until two real implementations exist? Starting unversioned with `ErrNotSupported` for graceful degradation is proposed, but community input is welcome.
- [ ] **Push vs. pull for `SyncUsage`** — Should Michelangelo expose a webhook receiver so managers can push spend data rather than polling? LiteLLM supports `success_callback` to a custom endpoint. Evaluate in Phase 2.
- [ ] **Budget enforcement default** — Soft cap (Michelangelo alerts) vs. hard cap (gateway blocks requests at limit). Proposal: soft cap as OSS default; hard cap opt-in via `keyPolicy.hardBudgetEnforcement: true`.
- [ ] **Community certification** — Should the project maintain a compatibility test suite that community implementations run against to be listed as "verified"? Proposed for Phase 3.

**Resolved:**
- ~~**`LLMGateway` CRD scope**~~ → **Namespace-scoped** (consistent with existing Michelangelo CRDs). Cross-namespace sharing is opt-in via `michelangelo.ai/shared: "true"` annotation. Cluster-scoped CRDs create cluster-admin RBAC requirements that conflict with the multi-tenant model.
- ~~**Default gateway mechanism**~~ → **Annotation-based** (`michelangelo.ai/default-gateway: "true"` on a gateway in `michelangelo-system`). Auto-assigned to projects with `IsGenerativeAi == true` and no explicit `LLMGatewayRef`. No new CRD or webhook required.

## Rollout strategy

**Phase 1 (5–6 weeks):** `LLMGatewayManager` interface, `LLMGateway` CRD, `LLMGatewayManagerLiteLLM`, key issuance + env var injection, basic health polling. Gated by `llmGateway.enabled: false` in the Helm chart.

**Phase 2 (4 weeks):** `SyncUsage` → observability store, Prometheus metrics, `LLMGatewayManagerGeneric` for bring-your-own proxies.

**Phase 3 (2–3 weeks):** `writing-a-manager.md` guide, LiteLLM quickstart, community announcement. `LLMGatewayManagerPortkey` and `LLMGatewayManagerOpenRouter` as community contributions.

**Phase 4 (future):** Multi-region federation, guardrails hooks, model allowlist enforcement at control-plane level.

**Feature flag:** `llmGateway.enabled: false` in `helm/michelangelo/values.yaml`. The controller is not installed unless explicitly opted in. No changes to existing pipeline behavior.

**Migration path:** No migration required. Existing projects without an `LLMGateway` CRD are unaffected. Opt-in per project by creating an `LLMGateway` resource and setting `project.llmGateway.gatewayRef`.

## References

- [RFC-20260427: Michelangelo Control Plane Helm Chart](../20260427-michelangelo-helmchart/20260427-michelangelo-helmchart.md) — establishes Helm chart conventions this RFC extends
- [LiteLLM Virtual Keys API](https://docs.litellm.ai/docs/proxy/virtual_keys)
- [LiteLLM Spend Tracking](https://docs.litellm.ai/docs/proxy/cost_tracking)
