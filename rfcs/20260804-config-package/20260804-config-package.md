# RFC-20260804: Configuration Package Architecture

- **Status:** Draft
- **Author(s):** @craig-marker
- **Created:** 2026-08-04

---

## Problem statement

`@michelangelo-ai/core` is a rendering engine that currently hardcodes domain configuration inside its own source tree. This creates two problems:

1. **No reuse across consumers.** A second application built on `core` must either fork the config or rebuild it from scratch. Shared building blocks cannot be imported independently which adds friction for downstream consumers adopting new functionality.
2. **Engine and domain are coupled.** Core knows about Pipelines, Triggers, and Deployments. Changes to entity configuration require changes inside the engine package, even though the engine's rendering logic is unchanged.

## Motivation

Michelangelo's UI is designed to be adoptable by different teams, each with different entity sets, different phase structures, and potentially different backend transports. The rendering engine (`core`) was always designed to be agnostic to all of this, but today it isn't.

## Goals

- A consumer can import configuration building blocks and compose a bespoke user flow using standard TypeScript.
- `@michelangelo-ai/core` has zero domain knowledge.

## Non-goals

- Building an override engine (slot-based registration, deep merge utilities, builder APIs). Consumers compose with standard TypeScript.
- Auto-generating configuration from protobuf schemas. The config package is hand-maintained; protoc plugin integration is a future consideration.
- Splitting protobuf-generated types into a separate `@michelangelo-ai/api` package. Generated types remain in `@michelangelo-ai/rpc` until a concrete consumer needs types without the transport dependency. The config package depends on `rpc` via `import type` (no runtime cost).
- Prescribing the correct configuration of deployment approach of Michelangelo Studio. Operators can use whatever configuration works best for their deployment, including custom React components. 

## High-level architecture

One new package alongside the existing two:

```
┌─────────────────────────────────────────────────────────┐
│  Consumer Application                                   │
│                                                         │
│  Imports config pieces, composes category/phase tree,   │
│  passes StudioConfig to CoreApp                         │
└────────────┬──────────────┬─────────────────────────────┘
             │              │
     ┌───────▼───────┐  ┌──▼──────────────────┐
     │  @m-ai/config │  │  @m-ai/core         │
     │               │  │                     │
     │  Entity parts │  │  Rendering engine   │
     │  Shared fields│  │  ConfigProvider     │
     │  Default tree │  │  useStudioConfig()  │
     └───┬───────┬───┘  └─────────────────────┘
         │       │
   ┌─────▼──┐ ┌─▼────────┐
   │@m-ai/  │ │@m-ai/    │
   │  rpc   │ │  core    │
   │(types) │ │(all)     │
   └────────┘ └──────────┘
```

Dependency rules:

- `config` → `rpc` (for resource type interfaces, `import type` only) + `core` (types to provide type safety to configs; components for building custom React components for flows)
- `core` depends on neither `config` nor `rpc`
- Consumer app → all three, composing as needed

### `@michelangelo-ai/config`

A parts library of entity configurations and shared building blocks. Exports plain objects in camelCase. No factories, no registries, no override machinery.

**Example per-entity configurations:**

```ts
// @michelangelo-ai/config/pipeline
export const pipelineEntity: PhaseEntityConfig;
export const pipelineColumns: ColumnConfig[];
export const pipelineListView: ListViewConfig;
export const pipelineDetailView: DetailViewConfig;
```

**Example shared field definitions:**

```ts
// @michelangelo-ai/config/fields
export const metadataName: FieldConfig; // metadata.name with URL template
export const statusState: FieldConfig; // status.state with configurable color/text maps
```

### Export scoping

The root barrel (`@michelangelo-ai/config`) exports cross-entity shared pieces only — shared field definitions, utilities, and the `defaultConfig` reference. Entity-specific configs are scoped to their subpath and not re-exported from the root.

Subpath exports use the `package.json` `exports` field which enforces the public API contract. Internal refactors don't break consumers as long as the `exports` map is stable.

### Consumer usage

**Pre-built app (zero-config):** Run the Docker image. No code required.

**Custom app on `core`:** Import pieces from `@michelangelo-ai/config`, compose a `StudioConfig`, pass to `CoreApp`. The `defaultConfig` export is the reference configuration used by the pre-built app — available as a starting point for consumers who want to modify the stock setup.

**Shared building blocks** are available from the root import:

```ts
import {
  metadataName,
  metadataCreatedAt,
  statusState,
  defaultConfig,
} from "@michelangelo-ai/config";
```

**Entity-specific configs** are scoped to subpath imports:

```ts
import {
  pipelineEntity,
  pipelineColumns,
} from "@michelangelo-ai/config/pipeline";
import { triggerEntity } from "@michelangelo-ai/config/trigger";
```

**Compose a custom tree:**

```ts
import { pipelineEntity } from "@michelangelo-ai/config/pipeline";
import { triggerEntity } from "@michelangelo-ai/config/trigger";

const train: PhaseConfig = {
  id: "train",
  name: "Train",
  icon: "train",
  state: "active",
  entities: [pipelineEntity, triggerEntity],
};

const config: StudioConfig = {
  categories: [{ id: "core-ml", name: "Core ML", phases: [train] }],
};
```

**Override one column:**

```ts
import { pipelineColumns } from "@michelangelo-ai/config/pipeline";

const myColumns = pipelineColumns.map((c) =>
  c.id === "metadata.name" ? { ...c, label: "Pipeline Name" } : c,
);
```

## APIs and CRDs

No CRD or backend API changes. This RFC affects the JavaScript package structure only.

| Package                   | Subpath exports                                                         |
| ------------------------- | ----------------------------------------------------------------------- |
| `@michelangelo-ai/config` | `./pipeline`, `./pipeline-run`, `./trigger`, `./deployment`, `./fields` |

## Alternatives considered

### Alternative A: Move config to the consumer app, no shared package

Config files move from `core` to the consumer `app/` directory.

**Pros:** Simple. No new package to maintain.

**Cons:** No sharing. Common building blocks (shared columns, field definitions) get duplicated when a second consumer appears.

**Why not chosen:** Does not satisfy goal for consumers to import configuration building blocks and compose bespoke user flows.

### Alternative B: Config package with override engine

Config exports a `createStudioConfig()` with a slot-based override registration system. Consumers register overrides at specific scopes rather than composing plain objects.

**Pros:** Supports rich customization and reduces friction for consumers with simple day 1 customizations and complex day N customizations.

**Cons:** Significant API surface to build and maintain.

**Why not chosen:** Solves for a use case that doesn't exist yet. Given the use case is unclear, it will be hard to get the API surface correct.

### Alternative C: Separate package for protobuf-generated types

Extract buf-generated types from `@michelangelo-ai/rpc` into a dedicated `@michelangelo-ai/api` package, so consumers can import resource types without a transport dependency.

**Pros:** Clean separation of data shapes from transport. Idiomatic in the protobuf-ES ecosystem.

**Cons:** Additional package to version and publish.

**Why not chosen:** No concrete consumer today that needs types without `rpc`. The config package's dependency on `rpc` is `import type` only — no runtime cost.

## Rollout strategy

**Phase 1 — Create `@michelangelo-ai/config`:** Move entity configs from `packages/core/config/` to `packages/config/`. Set up subpath exports and shared fields. Update the existing app to import from the new package. Remove config from core, make `CoreApp`'s `config` prop required.

No feature flags. No phased rollout. This is a structural refactor, so application behavior is identical before and after.

### Publishing

`@michelangelo-ai/config` is published to npm alongside the existing `@michelangelo-ai/core` and `@michelangelo-ai/rpc`. All three packages share the same version number and release together.

## References

- `packages/core/ARCHITECTURE.md` — rendering engine architecture
- `proto/api/v2/` — v2 resource protobuf definitions
- PR #1627 — `ConfigProvider` + `useStudioConfig()` hook
- PR #1643 — test-utils subpath export
