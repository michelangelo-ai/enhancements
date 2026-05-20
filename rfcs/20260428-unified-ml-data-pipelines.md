# RFC: Unifying ML and Data Processing Pipelines in Michelangelo

| Field | Value |
|---|---|
| **Authors** | weric |
| **Date** | 2026-04-28 |
| **Status** | Draft |

---

## Summary

Michelangelo pipelines today are optimized for machine learning workflows — model training, evaluation, and deployment. Production-grade ML systems increasingly require orchestration across broader data workloads: ingestion, transformation, feature generation, batch computation, and large-scale distributed processing.

This RFC proposes evolving Michelangelo pipelines and Uniflow into a unified orchestration foundation capable of supporting both ML and data engineering workloads through a shared control plane and execution model.

---

## Problem Statement

The current orchestration architecture lacks clearly defined system boundaries between:

- **Workflow definition** — how workflows are authored and expressed
- **Orchestration semantics** — retry, recovery, cleanup, state management
- **Execution runtime** — how and where work is dispatched
- **Provider implementation** — Spark, Ray, Kubernetes compute behavior
- **Infrastructure lifecycle management** — metadata, credentials, governance

As the platform evolved organically around ML workflows, orchestration responsibilities became distributed implicitly across SDKs, workflow runtimes, plugins, orchestration engines, and backend infrastructure.

This leads to recurring foundational questions whenever new orchestration capabilities are introduced:

- Which layer owns retry and recovery semantics?
- Should workflow artifacts contain execution logic or execution intent?
- What belongs in the workflow SDK versus the orchestration control plane?
- How should workflow visualization be standardized across high-code and declarative workflows?
- How should dynamic DAG state be persisted and replayed?
- Which responsibilities belong to orchestration providers versus the orchestration platform itself?

Without a formal orchestration boundary model, new capabilities risk being implemented as workflow-specific behaviors instead of reusable platform primitives. The result is:

- Duplicated orchestration semantics across workload types
- Tightly coupled execution runtimes
- Inconsistent reliability behavior
- Fragmented observability models
- Provider-specific workflow logic
- Increasing migration complexity over time

**The primary challenge is not feature enablement. It is defining a generalized orchestration architecture with stable system boundaries capable of supporting future orchestration evolution consistently across both ML and data engineering workloads.**

---

## Motivation

Michelangelo pipelines already provide strong orchestration through Uniflow and Temporal/Cadence — particularly for ML workflows requiring dynamic DAG execution, long-running reliability, and Python/Starlark SDK authoring.

As Michelangelo expands beyond ML-centric workflows and is adopted by external partners, the primary opportunity is establishing Michelangelo as a generalized orchestration foundation rather than continuing to add ML-specific pipeline features.

Expected platform improvements:

| Area | Expected Outcome |
|---|---|
| Platform Fragmentation | Reduce duplicated orchestration systems across ML and DE workloads |
| Reliability Consistency | Standardize retry, recovery, cleanup, and execution semantics |
| Workflow Visibility | Enable visualization and observability for high-code workflows |
| Platform Extensibility | Allow future orchestration capabilities to evolve through shared primitives |
| Infrastructure Efficiency | Reduce duplicated orchestration infrastructure and operational overhead |

---

## Goals

- Define explicit, stable system boundaries across orchestration layers
- Converge orchestration semantics (retry, recovery, cleanup, credential persistence) into a shared control plane
- Introduce a Unified Workflow Intermediate Representation (IR) as the canonical orchestration contract
- Enable high-code workflows (Python/Starlark) to participate in the same observability and visualization model as declarative (YAML) workflows
- Support dynamic DAG execution (conditional branching, fan-out/fan-in, runtime task generation) through a late-bound topology model
- Preserve existing workflow authoring paradigms; no migration required for existing workflows

## Non-Goals

- Replacing existing Uniflow authoring paradigms
- Requiring migration or rewrite of existing workflows
- Replacing Temporal/Cadence orchestration infrastructure
- Building a standalone orchestration platform outside Michelangelo

---

## High-Level Architecture

The proposed architecture establishes five explicit layers with defined responsibilities:

```
┌─────────────────────────────────────────────────────────────┐
│                   Workflow Authoring Layer                    │
│            Python SDK  │  Starlark  │  YAML                  │
└────────────────────────┬────────────────────────────────────┘
                         │  Workflow IR (normalized)
┌────────────────────────▼────────────────────────────────────┐
│               Orchestration Control Plane                     │
│   Pipeline lifecycle · State · Retry/Recovery · Governance   │
└────────────────────────┬────────────────────────────────────┘
                         │  Orchestration intent
┌────────────────────────▼────────────────────────────────────┐
│                    Workflow Runtime                           │
│        Temporal / Cadence (durable, dynamic DAG)             │
└────────────────────────┬────────────────────────────────────┘
                         │  Execution requests
┌────────────────────────▼────────────────────────────────────┐
│                  Execution Provider Layer                     │
│              Spark  │  Ray  │  Kubernetes                    │
└────────────────────────┬────────────────────────────────────┘
                         │  Shared services
┌────────────────────────▼────────────────────────────────────┐
│                  Infrastructure Services                      │
│     Metadata · Observability · RBAC · Credential Lifecycle   │
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| Workflow Authoring | Python SDK, Starlark, YAML; expresses orchestration intent |
| Orchestration Control Plane | Pipeline lifecycle, orchestration state, retry/recovery, trigger orchestration, governance, workflow reconciliation |
| Workflow Runtime | Durable dynamic DAG execution via Temporal/Cadence; late-bound topology |
| Execution Providers | Provider-specific compute: Spark, Ray, Kubernetes |
| Infrastructure Services | Metadata, observability, RBAC, credential and token lifecycle |

### Four Architectural Principles

| Principle | Description |
|---|---|
| Workload-Agnostic Orchestration | Orchestration primitives remain independent from ML-specific semantics |
| Separation of Definition and Execution | Workflow definitions express intent; execution behavior is resolved through providers |
| Platform-Managed Reliability | Retry, recovery, cleanup, and credential persistence are centralized platform responsibilities |
| Unified Observability Model | All workflows converge into a shared visibility model regardless of authoring paradigm |

### Unified Workflow IR

All workflows — Python SDK, Starlark, and YAML — are normalized into a shared Intermediate Representation before execution. The IR is the canonical orchestration contract across the platform.

The IR standardizes:

- DAG structure and execution dependencies
- Retry semantics
- Observability and lineage metadata
- Orchestration lifecycle hooks

This makes high-code workflows first-class orchestration graphs rather than opaque execution artifacts, enabling visualization and observability parity with declarative workflows.

### Dynamic DAG Execution

Unlike static DAG systems where topology is fixed at submission time, the workflow runtime supports **late-bound topology**:

- Conditional branching
- Iterative execution
- Runtime fan-out / fan-in
- Dynamic task generation

Orchestration behavior adapts to runtime state while preserving deterministic execution guarantees via Temporal/Cadence.

### Control Plane as Orchestration Authority

The Michelangelo API Server becomes the centralized orchestration authority responsible for:

- Pipeline lifecycle management
- Orchestration state persistence
- Retry and recovery coordination
- Trigger orchestration
- Governance enforcement
- Workflow reconciliation

The control plane acts as the durable orchestration boundary between workflow definition and runtime execution, preventing orchestration semantics from leaking into SDKs or provider implementations.

---

## APIs and CRDs

*To be defined in follow-on implementation RFCs. This RFC establishes architectural boundaries; API surface changes will be proposed per component as implementation proceeds.*

Key areas expected to produce public API changes:

- **Workflow IR schema** — versioned contract for the normalized workflow representation
- **Provider plugin interface** — standardized interface for registering and invoking execution providers
- **Orchestration control plane API** — pipeline lifecycle, state query, and reconciliation endpoints

---

## Alternatives Considered

### 1. Extend existing Uniflow workflows ad hoc

Continue adding ML and DE capabilities directly to Uniflow workflows as one-off features without addressing the architectural boundaries.

**Tradeoff:** Lowest short-term cost, but perpetuates the fragmentation described in the problem statement. Each new capability re-opens foundational architectural questions, and migration complexity grows over time. Rejected because the cumulative cost outweighs incremental savings.

### 2. Build a separate data engineering pipeline system

Create a parallel orchestration system for DE workloads, independent from Michelangelo pipelines.

**Tradeoff:** Clean separation of concerns in the short term, but introduces two independent control planes, two observability models, and two sets of reliability semantics. Rejected because it directly increases the fragmentation this proposal aims to reduce.

### 3. Adopt an external orchestration platform (e.g. Airflow, Prefect, Metaflow)

Replace the Michelangelo orchestration layer with an external system for generalized workflow orchestration.

**Tradeoff:** Leverages existing community investment but requires migration of all existing Uniflow and Temporal-based workflows, loses platform-specific integrations (RBAC, credential lifecycle, observability), and cedes platform control. Rejected as inconsistent with Non-Goals and with Michelangelo's role as the orchestration foundation.

---

## Open Questions

1. **IR versioning strategy** — How should the Workflow IR schema be versioned? What is the compatibility guarantee across platform versions?

2. **Dynamic DAG state persistence** — What durability guarantees does the workflow runtime provide for dynamically generated tasks? How is replay handled for fan-out operations?

3. **Provider plugin contract** — Should execution providers be in-process plugins or out-of-process services? What are the isolation and failure semantics?

4. **Observability convergence** — What is the minimal IR metadata required to enable visualization parity between high-code and declarative workflows in the short term?

5. **Credential lifecycle scope** — Which credential types (service tokens, user-delegated tokens, storage credentials) are managed by the platform versus delegated to providers?

6. **Migration path for existing workflows** — Although migration is a non-goal, what opt-in path exists for existing workflows to adopt the new IR and observability model without a full rewrite?

---

## Rollout Strategy

This proposal is intentionally architecture-first. Implementation is expected to proceed in phases:

**Phase 1 — Control plane boundary formalization**
Define the Orchestration Control Plane contract. Establish the Michelangelo API Server as the authoritative lifecycle manager. No changes to existing workflow authoring or execution.

**Phase 2 — Unified Workflow IR**
Introduce the IR schema. Begin converting YAML workflows to IR at submission time. High-code workflows continue to execute opaquely but IR conversion becomes available as an opt-in.

**Phase 3 — Observability convergence**
Route IR-derived DAG metadata to the shared observability model. Enable workflow visualization for high-code workflows that have adopted IR conversion.

**Phase 4 — Provider plugin interface**
Formalize the provider plugin contract. Migrate Spark, Ray, and Kubernetes provider implementations to the standardized interface.

**Phase 5 — Platform-managed reliability**
Centralize retry, recovery, and credential lifecycle into the control plane. Deprecate provider-specific reliability implementations.

Feature flags will gate each phase independently. Existing workflows remain unaffected until they explicitly opt into new behaviors.

---

## References

- [Michelangelo platform overview](https://eng.uber.com/michelangelo-machine-learning-platform/)
- [Kubernetes Enhancement Proposals (KEPs)](https://github.com/kubernetes/enhancements/tree/master/keps) — process inspiration
- [Ray Enhancement Proposals (REPs)](https://github.com/ray-project/enhancements/tree/main/reps) — lightweight RFC format inspiration
