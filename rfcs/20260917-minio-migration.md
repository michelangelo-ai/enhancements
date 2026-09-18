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

## Rollout strategy

The migration is staged so that no single PR is a big-bang cutover, and
each stage is independently shippable and revertible.

### Stage 1 — Sandbox deployment swap (lowest risk, highest signal)

Swap `python/michelangelo/cli/sandbox/resources/minio.yaml`'s Pod spec to
run SeaweedFS's `weed server` all-in-one mode instead of `minio server`,
adding one new `ConfigMap` for SeaweedFS's S3 identity config. The
`minio` Service name, both NodePorts, and the `minioadmin`/`minioadmin`
demo credentials stay byte-for-byte identical, so every other manifest that
references the object store by Service DNS name (bucket-setup job,
credentials secrets, other sandbox components) needs no change.

Representative diff:

```diff
     - name: minio
-      image: minio/minio:RELEASE.2025-05-24T17-08-30Z
+      image: chrislusf/seaweedfs:<pinned-stable-tag>
       imagePullPolicy: IfNotPresent
       command:
-        - /bin/bash
+        - /bin/sh
         - -c
       args:
-        - minio server /data --console-address :9090 --address :9091
+        - >
+          weed server -dir=/data
+          -master.port=9333 -volume.port=8080 -filer.port=8888
+          -s3 -s3.port=9091 -s3.config=/etc/seaweedfs/s3_config.json
       volumeMounts:
         - mountPath: /data
           name: data-volume
+        - mountPath: /etc/seaweedfs
+          name: s3-config-volume
   volumes:
     - name: data-volume
       hostPath:
         path: /shared/minio-data
         type: DirectoryOrCreate
+    - name: s3-config-volume
+      configMap:
+        name: seaweedfs-s3-config
```

The Service block (name, ports, selector) is unchanged.

**Blast radius:** local dev / CI k3d sandbox only. Zero production impact —
the Helm chart's `objectStorage` section is untouched.

**Rollback:** the sandbox's data volume is already treated as ephemeral by
every existing teardown path; revert the manifest to the prior commit and
re-apply.

### Stage 2 — Client SDK compatibility verification (code change only if needed)

Add a small regression test, run against the Stage-1 sandbox, exercising
the two S3-error-code-dependent branches in `minio_backend.py`:
`stat_object` on a missing key (expects `S3Error.code == "NoSuchKey"`) and
`make_bucket` on an already-existing bucket (expects
`S3Error.code == "BucketAlreadyOwnedByYou"`). If SeaweedFS returns these
exact standard codes (expected, since both are S3-standard, not MinIO
inventions), this stage ships as a test-only PR that becomes a permanent
regression guard. If the codes differ, this stage additionally patches the
two `except S3Error` blocks in `minio_backend.py` to translate the actual
returned code.

**Blast radius:** `minio_backend.py` (only if codes differ) plus one new
test file. **Rollback:** revert the single file/test.

### Stage 3 — Helm chart / production config

Update `helm/michelangelo/values.yaml`'s `objectStorage` and
`logPersistence` comments/examples to add SeaweedFS alongside the existing
MinIO/S3/GCS examples. No schema or key changes — the section is already
server-agnostic. No forced production migration: an operator running their
own MinIO in production today is unaffected; this stage only documents
SeaweedFS as an additional supported option.

**Blast radius:** documentation/comments in one file, zero functional
change to any rendered manifest. **Rollback:** trivial comment revert.

### Stage 4 — Docs updates

Update any docs page that names MinIO as *the* object store (rather than
*an* object store) with a short callout referencing issue #672's archival
timeline and pointing to this migration as the recommended path forward.

**Blast radius:** docs/strings only. **Rollback:** trivial revert.

## References

- [michelangelo-ai/michelangelo#672](https://github.com/michelangelo-ai/michelangelo/issues/672) — tracking issue for the MinIO archival and candidate replacement discussion.
