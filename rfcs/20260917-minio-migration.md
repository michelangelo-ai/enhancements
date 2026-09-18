# RFC-20260917-minio-migration: Migrate object storage from MinIO to SeaweedFS

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-09-17
- **Internal ERD:** N/A

---

## Problem statement

Michelangelo's sandbox and production deployments default to MinIO as the
S3-compatible object store, but MinIO's community edition is no longer a
maintainable dependency:

- May 2021: license changed Apache-2.0 → AGPLv3.
- Feb 2025: the admin GUI was removed from the community edition.
- Oct 2025: MinIO stopped publishing new Docker images for the community
  edition.
- Dec 2025: the community edition entered maintenance mode.
- Feb 12, 2026: the community edition repository was fully archived, with
  its README redirecting to the paid **AIStor** product (starting at
  $96,000/year).

Michelangelo currently pins `minio/minio:RELEASE.2025-05-24T17-08-30Z` in the
sandbox deployment (`python/michelangelo/cli/sandbox/resources/minio.yaml`).
That image still runs today but will never receive another security patch.

## Motivation

Every user who runs the Michelangelo sandbox, and every operator who deploys
Michelangelo with an in-cluster MinIO instance, is relying on unmaintained,
unpatched software. Continuing to default to MinIO leaves the project
recommending a dead end to new adopters and a growing security liability to
existing ones. This should be solved before the archived image's known CVEs
(present or future) become a blocker for anyone doing a fresh install.

This is tracked in
[michelangelo-ai/michelangelo#672](https://github.com/michelangelo-ai/michelangelo/issues/672),
which already proposes four candidate replacements. This RFC validates that
proposal against Michelangelo's actual code paths, client libraries, and
deployment shape, and turns it into a concrete, staged migration plan.

## Goals

- Recommend a single, best-fit, actively maintained S3-compatible
  replacement for MinIO, justified against Michelangelo's actual usage
  patterns (not a generic popularity comparison).
- Provide a staged migration plan covering every current MinIO touchpoint,
  where each stage is independently shippable and revertible.
- Preserve today's zero-forced-migration contract: operators who already run
  their own MinIO (or any other S3-compatible endpoint) in production must
  be unaffected by this change landing.

## Non-goals

- Implementing the migration. This RFC defines the design and an
  implementation-ready Stage 1 manifest; execution happens in follow-up
  implementation PRs against the core repository.
- Standing up a live throughput/latency benchmark between candidates. A
  desk-based evaluation against Michelangelo's real usage (plain
  CRUD/list, no multipart/versioning/presigned URLs) is sufficient to break
  the tie between candidates.
- Deprecating or removing MinIO support. The Helm chart's `objectStorage`
  section is, and remains, bring-your-own-endpoint; operators who choose to
  keep running MinIO are unaffected.
- Choosing a newer or more novel candidate purely because it is a "cleaner"
  drop-in, if doing so trades away production maturity (see Alternatives
  considered).

## High-level architecture

**Recommendation: migrate to [SeaweedFS](https://github.com/seaweedfs/seaweedfs)**
(Apache-2.0, Go).

Michelangelo's object-storage client code is already well-abstracted and
already server-agnostic:

- The Go side (`go/base/blobstore/blobstore.go`'s `BlobStoreClient`
  interface, with exactly two methods: `Get` and `Scheme`) is implemented
  against `minio-go/v7`, which is a generic S3-compatible client SDK, not a
  MinIO-server-specific one. No MinIO-only behavior (multipart, versioning,
  lifecycle, presigned URLs) is used anywhere on this path.
- The Python side has exactly one module genuinely coupled to a
  MinIO-branded library: `python/michelangelo/lib/artifact_manager/minio_backend.py`,
  which uses the `minio` Python SDK (`fput_object`, `fget_object`,
  `list_objects`, `stat_object`, `bucket_exists`, `make_bucket` — again, all
  plain S3 CRUD/list/bucket operations). Every other Python file that
  mentions MinIO (`uniflow/core/file_sync.py`, `uniflow/plugins/ray/io.py`,
  `uniflow/registration/uniflow_tar.py`, `workflow/tasks/functions/sinks/s3.py`)
  already uses `fsspec`, `s3fs`, or `pyarrow.fs.S3FileSystem` and needs no
  client-library change — only an endpoint/config value change.

Because of this, replacing MinIO is primarily a **server swap**, not a
client rewrite, provided the replacement's S3 API compatibility is solid
enough for these specific operations. SeaweedFS is the best fit because it:

- Ships an all-in-one `weed server` mode (master, volume, filer, and S3
  gateway in a single process), which is the closest architectural match to
  the sandbox's current single-Pod `minio.yaml` deployment.
- Runs a dedicated S3-compatibility test suite in CI and publishes an
  explicit "Supported APIs vs MinIO" comparison, covering every operation
  Michelangelo's code actually calls.
- Ships an in-repo Helm chart (also listed on Artifact Hub) plus a
  community operator, so production operators who want to adopt it have an
  officially supported packaging path.
- Is Apache-2.0 licensed, matching Michelangelo's own license, and has the
  longest production track record and largest community of the four
  candidates evaluated (it predates MinIO's AGPL relicensing).

No new abstraction is introduced. The existing `BlobStoreClient` interface
and `MinioStorageBackend` class already hide the concrete server behind a
contract, because both SDKs in use were already generic S3 clients rather
than MinIO-only ones. This RFC does not propose renaming the `minio`
package/module for cosmetic accuracy — that is deferred to a future,
separate, purely-cosmetic change so as not to inflate the migration's diff
for zero behavioral gain.

## APIs and CRDs

None. This is an internal object-storage backend swap with no public
API or CRD surface changes.

`helm/michelangelo/values.yaml`'s `objectStorage` section (endpoint,
secure, region, bucket, credentials/`existingSecret`) is already
server-agnostic today — it documents `s3.amazonaws.com`,
`storage.googleapis.com`, and `minio:9000` as interchangeable example
endpoints. This migration adds SeaweedFS as another documented example
endpoint; it does not add, remove, or rename any chart key. Operators
already using the chart today require no config changes to keep working
exactly as before.

## Alternatives considered

### Alternative A: RustFS

**Pros:** Explicitly designed as a drop-in MinIO replacement (same claimed
on-disk data format, same S3 semantics), single binary, closest to a
literal drop-in swap of the four candidates.

**Cons:** Tagged `v1.0.0-alpha` as of this writing, with the maintainers'
own public guidance against production deployment before a stable 1.0.
Official Helm chart availability was not confirmed.

**Why not chosen:** Accepting pre-1.0 risk for a production storage
dependency, purely because it looks like the easiest technical swap, is
exactly the trade this RFC's non-goals rule out. Worth revisiting if RustFS
reaches a stable release with an official Helm chart before this migration
ships.

### Alternative B: Garage

**Pros:** Geo-distributed, minimal-ops design; production use since 2020 by
its own maintainers and at least one other organization; ships an in-repo
Helm chart.

**Cons:** Actually licensed **AGPL-3.0** — the same copyleft family as the
MinIO version being replaced (issue #672's original table incorrectly
listed it as LGPL-3.0). It is also architecturally optimized for
small-to-medium, geo-distributed deployments rather than a single-cluster
shape, and has the smallest community/adoption footprint of the four
candidates.

**Why not chosen:** The license correction removes what was likely its main
differentiator, and its target deployment shape doesn't match
Michelangelo's single-cluster sandbox/production topology.

### Alternative C: versitygw

**Pros:** Apache-2.0 licensed; the best packaging of the four candidates
(an official, OCI-distributed Helm chart); every PR is gated on a
comprehensive S3-compatibility test suite; vendor-backed and described by
its own docs as production-ready.

**Cons:** Architecturally different from the other three — it is a
stateless S3-to-filesystem gateway, not a storage engine with its own
on-disk format. Adopting it means provisioning and managing a separate
backing filesystem/PVC behind it, rather than a self-contained drop-in Pod
replacement.

**Why not chosen:** This is a legitimate design if Michelangelo later wants
a stateless, horizontally-scalable storage-gateway tier in front of shared
filesystem infrastructure it already operates, but that is a larger,
separate architectural decision than this migration's scope (replace an
archived server with a maintained one, with minimal operational
disruption). Flagged as the strongest runner-up if SeaweedFS's all-in-one
mode doesn't satisfy the open items below.

## Open questions

- [ ] Exact SeaweedFS image tag to pin (a specific stable release tag, not
      `:latest`, matching the specificity of the current
      `minio/minio:RELEASE.2025-05-24T17-08-30Z` pin) — to be confirmed at
      implementation time against Docker Hub / the project's GitHub
      Releases page.
- [ ] Exact `weed server` flag names (`-dir`, `-master.port`,
      `-volume.port`, `-filer.port`, `-s3`, `-s3.port`, `-s3.config`) need
      to be verified against the pinned image's own `weed server -h`
      output before merging the Stage 1 implementation PR.
- [ ] MinIO's admin console UI has no direct SeaweedFS equivalent in
      all-in-one mode. The implementation needs to decide between exposing
      the SeaweedFS filer's browsable UI as the new "console" link, or
      dropping the console link for this stage as a documented UX
      regression versus MinIO.
- [ ] Whether SeaweedFS's S3 gateway returns the exact same S3 standard
      error codes (`NoSuchKey`, `BucketAlreadyOwnedByYou`) that
      `minio_backend.py` currently matches against via the MinIO SDK's
      `S3Error.code` — the one place in the codebase where exact
      error-code fidelity (not just data-plane compatibility) matters. If
      not, a small compatibility shim is needed inside the existing
      `except S3Error` blocks.
- [ ] Exact name for the new sandbox flag — `--object-store` is proposed
      (matching the existing `--workflow` flag's naming style), but open to
      bikeshedding at implementation time.
- [ ] Exact soak-window length before flipping the sandbox default; two
      minor releases with default=`minio` is proposed based on the
      project's observed ~1-2 week release cadence and the weekly
      floor-bump job's cadence, but could be shortened or lengthened based
      on release-cadence changes.

## Rollout strategy

The migration ships as dual-backend support with a release-gated default
flip, not an in-place swap — `michelangelo-examples` pins `michelangelo` via
floor constraints (`>=`, not exact versions), so every release published
during the rollout window must independently keep working for it. Ordering:
**sandbox/control-plane dual-support first → Python code changes second →
`michelangelo-examples` last**, riding its existing automation.

### 1. Sandbox and control-plane dual-support

Add a new `weed-server.yaml` + `seaweedfs-s3-config.yaml` manifest pair
**alongside** the existing, untouched `minio.yaml` (no in-place edit), and
add an `--object-store {minio,seaweedfs}` flag to `ma sandbox create` /
`ma sandbox sync`, defaulting to `minio` — following the same pattern as
the existing `--workflow {cadence,temporal}` flag. Regardless of which
backend is selected, the `Service` object keeps the name `minio` and both
existing NodePorts (9090, 9091), so every other manifest that references
the object store by Service DNS name needs no change.

No control-plane code change is needed: `helm/michelangelo/values.yaml`'s
`objectStorage.endpoint` is already backend-agnostic, so "dual support"
there is satisfied by a documentation update alone (adding SeaweedFS as a
documented endpoint example next to the existing MinIO/S3/GCS ones).

**Blast radius:** local dev / CI k3d sandbox only, opt-in via flag. Zero
production impact.

### 2. Python code changes (additive only)

Introduce a generically-named `S3StorageBackend` alias for
`MinioStorageBackend`, while keeping the existing
`michelangelo.lib.artifact_manager.minio_backend.MinioStorageBackend`
import path and constructor signature permanently functional — this is the
one hard backward-compatibility contract the whole rollout depends on, since
a floor-pinned consumer can resolve to any release in the window. Also add
the small `NoSuchKey` / `BucketAlreadyOwnedByYou` error-code regression test
against the Stage-1 sandbox described in Open questions; if SeaweedFS's
codes differ from MinIO's, patch the two `except S3Error` blocks in
`minio_backend.py` to translate them. Dropping the `minio` PyPI SDK
dependency (e.g. a `boto3` rewrite) is explicitly out of scope unless that
error-code check fails — the SDK already works as a generic S3 client
against any compatible server.

### 3. Release staging and default flip

Ship dual-support (§1) in the next minor release with the sandbox default
unchanged (`minio`), announced in `CHANGELOG.md` as an opt-in preview. Soak
for a minimum of two minor releases with default still `minio`, giving
`michelangelo-examples`' existing weekly `bump-michelangelo-pin.yml`
floor-bump job (gated on its own `test.yaml` CI) at least two independent
chances to exercise dual-support code against real pipelines. Only then
flip the sandbox default to `seaweedfs`, called out explicitly in
`CHANGELOG.md` since sandbox users relying on the implicit MinIO default
would otherwise be silently switched. MinIO is never force-removed as a
selectable option — there is no forcing function to delete it, and keeping
it costs nothing.

Before publishing any release that falls within this rollout window, gate
it by running `michelangelo-examples`' own `test.yaml` against a
release-candidate build (reusing that existing workflow against a
pre-publish wheel, not a new CI system) — closing the gap where an RC could
otherwise only be caught after a real release already went out.

### 4. `michelangelo-examples`, last, with zero required code changes

`michelangelo-examples`' own `_backend.py::resolve_storage_backend()`
already treats `MinioStorageBackend` as a generic S3-endpoint backend, and
depends only on the import path and constructor signature that §2
guarantees stay stable. Its weekly floor-bump automation is therefore the
integration test for this rollout, not new automation, and needs zero
required code changes throughout. Two optional, one-time manual cleanup
PRs remain for later, once the floor moves past the relevant release: (a)
switching `_backend.py`'s import to the `S3StorageBackend` name, and (b)
dropping its own now-redundant direct `minio>=7.2,<8` dependency.

**Rollback:** every stage above is independently revertible — dual-support
via the flag default, the Python alias via reverting the single addition,
and the default flip via a follow-up release reverting `CHANGELOG.md`'s
announced default back to `minio`.

## References

- [michelangelo-ai/michelangelo#672](https://github.com/michelangelo-ai/michelangelo/issues/672) — tracking issue for the MinIO archival and candidate replacement discussion.
