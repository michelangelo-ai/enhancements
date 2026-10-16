# RFC-20260804-kueue-job-queue: Kueue-backed job queue and gang admission for Ray and Spark workloads

- **Status:** Draft
- **Author(s):** @apurvapatkeshwar
- **Created:** 2026-08-04
- **Internal ERD:** N/A (external contribution)

---

## Problem statement

Michelangelo's batch job scheduler admits every job immediately. The shipped `JobQueue` implementation (`go/components/jobs/scheduler/scheduler.go`) is a FIFO channel queue drained by a goroutine, and the shipped `AssignmentStrategy` (`ClusterOnlyAssignmentStrategy` in `go/components/jobs/scheduler/framework/cluster_only_assignment_engine.go`) picks the first available cluster, or the one named by the `michelangelo/cluster-affinity` label (`ClusterAffinityLabelKey` in `go/components/jobs/common/constants/constants.go`; the engine's doc comment still cites the older `resourcepool.michelangelo/cluster` name). There is no quota, no fair sharing between projects, no queueing when a cluster is full, and no priority or preemption (the `SchedulingSpec.preemptible` field in `proto/api/v2/job.proto` exists but nothing enforces it; priority is an open TODO, issue #929).

Two concrete symptoms:

1. On a shared compute cluster, every submitted job races for capacity. A large training job from one project can starve everyone else, and there is no mechanism to say "team A gets 40 GPUs, team B gets 24, and unused quota is borrowable". Operators coming from Kubeflow or KubeRay environments where Kueue is standard hit this in their first week.

2. Multi-pod jobs are not admitted atomically. The Ray plugin says it plainly: `python/michelangelo/uniflow/plugins/ray/task.star` inflates `RAY_NUM_REDIS_GET_RETRIES` to give workers roughly 30 minutes to find the head node "because the Job Controller doesn't support gang scheduling of nodes". That 30 minutes is a hard ceiling, not patience: the same constant (`DEFAULT_CREATE_CLUSTER_TIMEOUT_SECONDS = 60 * 30`) is passed to `ray.create_cluster` as the readiness sensor's expiration, and on expiry the workflow terminates the cluster and fails the task (`go/worker/plugins/ray/starlark_module.go`). On a contended cluster this means partially-started Ray clusters holding GPUs for up to half an hour while waiting for pods that cannot schedule — capacity is wasted exactly when it is scarcest, two half-admitted jobs can hold each other's capacity hostage, and after 30 minutes both surface to their users as failures.

The scheduler was explicitly designed for a pluggable backend. The operator guide (`docs/operator-guides/jobs/extend-michelangelo-batch-job-scheduler-system.md`) documents "Custom Backend (e.g., Kueue, Volcano)" as extension Option 2 and even sketches a `KueueScheduler` implementing `JobQueue`. What is missing is a supported, in-tree implementation and the config surface to select it.

There is prior art: PR michelangelo-ai/michelangelo#1129 ("Integration Kueue into compute cluster scheduler for ray job") demonstrated end to end that a Kueue-labeled RayCluster gets suspended, admitted against a ClusterQueue, and started, all with Kueue disabled by default. It stalled in review with two unresolved design points (label propagation and queue topology, addressed below) and was scoped to Ray plus a sandbox demo. This RFC generalizes that work into a first-class backend.

## Motivation

Any organization running Michelangelo on shared clusters needs three things the platform does not provide today: quota enforcement per team or project, fair sharing of idle capacity, and all-or-nothing admission for multi-pod jobs. These are table stakes in the ecosystems adopters migrate from: Kueue ships a TrainJob integration for Kubeflow Trainer (enabled in Kueue's configuration), KubeRay documents native Kueue integration for RayJob and RayCluster, and the managed platforms ship equivalent guardrails. An evaluator comparing Michelangelo against "Kubeflow + Kueue + Ray assembled by hand" currently finds a scheduling story that amounts to first-come-first-served.

Kueue is the right substrate to compose rather than compete with. It is the Kubernetes-native job queueing project, it operates at the admission layer (deciding when a job may start and what quota it consumes) without replacing the pod scheduler, and upstream Kubernetes is absorbing gang scheduling concepts along the same lines: KEP-4671 (workload-aware scheduling) shipped alpha in Kubernetes v1.35 with the `Workload` API, and KEP-5832's Workload-template/PodGroup split was folded into it for v1.36 — early-stage, but a clear signal of where the ecosystem is converging. Building quota, fair sharing, and gang admission natively inside Michelangelo would duplicate a large, actively developed project and would put the code on the wrong side of that convergence.

Solving this now also derisks the roadmap: the Planned "resource pool selection" work concerns placement (which cluster), while this proposal concerns admission (when to start, against whose quota). Keeping those layers separate today means they compose cleanly later.

## Goals

- An opt-in `KueueJobQueue` backend implementing the existing `JobQueue` interface (`go/components/jobs/scheduler/scheduler.go`), selectable per deployment; the current default scheduler remains the default and is not modified.
- An additive `scheduler_type` field on `ClusterSpec` (`proto/api/v2/cluster.proto`) so operators declare, per compute cluster, whether jobs dispatched there are Kueue-managed. Unset means today's behavior.
- Gang (all-or-nothing) admission for Ray workloads via Kueue's RayCluster integration, with pod sets covering the head group and every worker group, eliminating the `RAY_NUM_REDIS_GET_RETRIES` workaround for Kueue-managed clusters. (The RayJob that submits work to the cluster is deliberately not gated — see the architecture section for why `clusterSelector`-based RayJobs stay outside Kueue.)
- A defined queue topology and config surface: one Kueue ClusterQueue per compute cluster (capacity), one LocalQueue per Michelangelo project (tenancy), with the mapping resolved by the control plane, not by user supplied labels.
- A deliberate, allowlisted label propagation path from Michelangelo job objects to the Kubernetes objects created on compute clusters, so that Kueue labels reach the workload and internal control-plane labels do not leak.
- A design that extends to Spark with no further API changes once the Planned Spark-on-Kubernetes launch path lands (`SparkJob` proto and scheduler job type already exist; the k8s mapping in `go/components/jobs/client/k8sengine/mapper.go` is not yet implemented).
- Full backward compatibility: existing deployments see zero behavior change unless they enable the backend.

## Non-goals

- Replacing or modifying cluster placement. `AssignmentStrategy` continues to choose the target compute cluster; Kueue governs admission within that cluster. This deliberately stays out of the roadmap's "resource pool selection" territory and composes with whatever lands there.
- MultiKueue (cross-cluster queueing and dispatch). Michelangelo already has federation and an assignment layer; introducing a second cross-cluster brain would create two sources of truth. Revisit later if maintainers want Kueue to own placement too.
- A bespoke gang scheduler, PodGroup implementation, or topology-aware placement engine inside Michelangelo.
- Priority and preemption semantics at the Michelangelo API level (issue
  #929). The design maps cleanly onto Kueue WorkloadPriorityClass later, and
this RFC notes the seams, but the proto work for priority is out of scope.
- Changing the Uniflow DSL or user-facing pipeline definitions. Users do not see Kueue; they see jobs that queue instead of failing or deadlocking.
- Managing Kueue itself in production (installation, upgrades). The Helm chart gains optional sandbox/e2e installation for testing, but production Kueue lifecycle belongs to the cluster operator, like the metrics stack.

## High-level architecture

The design follows the operator guide's Option 2 (custom `JobQueue` backend) and keeps the control-plane/compute-cluster split that PR #1129 validated: Kueue runs on each participating compute cluster and admits work locally; the Michelangelo control plane decides where a job goes, stamps it with the right queue label, dispatches it suspended, and reflects admission status back into job conditions.

```
Control plane cluster                          Compute cluster N
+---------------------------+                  +-----------------------------+
| Job controllers           |                  | Kueue controller            |
|  (RayCluster, SparkJob)   |                  |  ClusterQueue: cluster-N    |
|        |                  |                  |  LocalQueue per project     |
|        v                  |                  |                             |
| JobQueue.Enqueue(job)     |                  |  Workload (pod sets:        |
|        |                  |                  |   head x1, workers xM)      |
|  KueueJobQueue            |   dispatch       |        |                    |
|   1. AssignmentStrategy   |  (suspended,     |        v  admit when all    |
|      selects cluster      |   queue-name     |  suspend=false, pods start  |
|   2. resolve project ->   |   label set)     |                             |
|      LocalQueue           | ---------------> |  RayCluster / RayJob        |
|   3. mark job Kueue-      |                  |                             |
|      managed              |  status sync     |                             |
|   4. update Assignment    | <--------------- |  admission / eviction       |
|      Info + condition     |  (FederatedClient)  reflected in conditions    |
+---------------------------+                  +-----------------------------+
```

Walking through the lifecycle:

1. **Enqueue.** A job controller calls `JobQueue.Enqueue` exactly as today. When the Kueue backend is active, `KueueJobQueue` (new package `go/components/jobs/scheduler/kueue`) satisfies the interface. It is wired through an fx factory in `go/components/jobs/scheduler/module.go` that selects the implementation from config, following the factory pattern the operator guide already documents; the default branch returns the existing `Scheduler` untouched.

2. **Placement.** `KueueJobQueue` runs the same `framework.AssignmentStrategy.Select` call the default scheduler runs (interface at `go/components/jobs/scheduler/framework/interface.go`). Admission and placement stay separate: the strategy picks the cluster, and the backend then consults that cluster's `ClusterSpec.scheduler_type`. If the target cluster is not Kueue-managed, the job proceeds exactly as today, which allows mixed fleets during migration.

3. **Queue resolution.** For a Kueue-managed target, the backend resolves the job's project to a LocalQueue name. The topology, following maintainer guidance on PR #1129: one ClusterQueue per compute cluster representing that cluster's schedulable capacity (or a carve-out of it), and one LocalQueue per Michelangelo project pointing at the ClusterQueue. Fair sharing and borrowing between projects then fall out of Kueue cohorts and quota configuration rather than new Michelangelo code. One deliberate consequence of current reality: the k8s engine dispatches every Ray object into the compute cluster's `default` namespace (`RayLocalNamespace` in `go/components/jobs/client/k8sengine/constants.go`; the project namespace exists on the control-plane CRD but is not propagated to compute clusters), and Kueue requires the LocalQueue to live in the workload's own namespace — so in this design all per-project LocalQueues live in that same dispatch namespace, distinguished by name (`ma-<project>`) — LocalQueues are namespaced objects selected by name through the queue-name label, and nothing in Kueue restricts a namespace to one of them. That trades away namespace-selector isolation on the compute cluster; materializing per-project namespaces there is a real tenancy improvement but a separate scope (mapper, RBAC, cleanup), noted in open questions rather than smuggled in here. The default name mapping is convention based (`ma-<project>`); an explicit override map is available in config for operators with existing Kueue estates.

4. **Dispatch, suspended.** The job is marked scheduled (AssignmentInfo and the scheduled condition, as today) and dispatched through the existing `FederatedClient` path. Two changes happen at the k8s engine layer (`go/components/jobs/client/k8sengine/`):
- The generated object carries `kueue.x-k8s.io/queue-name: <localqueue>`.
- Label propagation goes through a new `mapLabels` allowlist in `mapper.go`. To be precise about the starting point: today the k8s engine propagates no labels at all from Michelangelo job objects to the KubeRay objects it creates, so `mapLabels` introduces label propagation rather than restricting an existing flow. The allowlist shape is what makes that introduction safe, and it is the direction the first unresolved review comment on #1129 asked for: control-plane labels such as the `michelangelo/cluster-affinity` affinity label must not leak onto compute-cluster objects, and arbitrary user labels must not be able to smuggle a different queue-name. The queue label is set by the control plane after resolution, never passed through from user input.

Because the RayCluster is labeled for Kueue and the compute cluster's Kueue runs the `ray.io/raycluster` integration, Kueue itself creates the Workload, holds the object suspended (`spec.suspend=true`), and flips suspend only when the full Workload is admitted. Michelangelo does not hand-craft Workload objects for Ray; it delegates to Kueue's maintained integration.

One deliberate asymmetry: **only the RayCluster is queue-labeled, never the RayJob.** Michelangelo's Ray path creates a RayCluster and then a RayJob that targets it via `spec.clusterSelector`, and Kueue deliberately ignores such RayJobs entirely — since kueue#7218, its webhook and reconciler skip any RayJob with `clusterSelector` set, on the reasoning that the pre-existing cluster is where the resources live and was already admitted. Gating the RayCluster therefore gates everything real; the RayJob's submitter pod rides along. Two operational consequences worth stating: the compute cluster's Kueue should run with `manageJobsWithoutQueueName: false` (the default) so unlabeled objects are never swept into management; and for quota sizing, the log-collector sidecars injected into head and worker pods (100m CPU / 128Mi each) are counted inside the gated pod sets, while the RayJob's small submitter pod (sized to KubeRay's defaults by `buildSubmitterPodTemplate`) runs outside Kueue quota — operators sizing `nominalQuota` should know both.

5. **Gang admission.** Kueue's RayCluster integration constructs the Workload with one pod set for the head group and one per worker group, with counts, and admission is all-or-nothing against ClusterQueue quota: either the whole Ray cluster fits and starts, or nothing starts and the Workload waits in queue. (Kueue caps a Workload at 18 pod sets, so at most 17 worker groups per RayCluster; Michelangelo's Ray paths generate exactly one worker group today, so the limit is nowhere near binding — noted only to show it was checked.) For runtime stragglers (quota admitted but pods unschedulable due to fragmentation), the design relies on Kueue's `waitForPodsReady` all-or-nothing configuration, which evicts and requeues a workload whose pods do not all become ready within a deadline — with the honest caveat that `waitForPodsReady` is Kueue-installation-wide configuration, so enabling it affects every Kueue-managed workload on that compute cluster, not only Michelangelo's; the operator guide must say so (see open questions). Together these remove the deadlock and capacity-waste modes described in the problem statement. On Kueue-managed clusters the Ray plugin's inflated `RAY_NUM_REDIS_GET_RETRIES` becomes unnecessary; the default stays for non-Kueue clusters.

6. **Status reflection.** The existing federated status sync is extended so that queueing is visible from the control plane. The sync maps only the KubeRay `State` plus a derived reason today, so the `Queued` condition is synthesized control-plane-side from the suspended state (the existing `Enqueued` condition in `go/components/jobs/common/constants/constants.go` is the exact pattern to follow): while the workload awaits quota, the job surfaces `Queued` with the Kueue-reported reason (for example `QuotaReserved` pending or `Inadmissible` with message); on admission the normal running conditions take over. Eviction before the cluster ever became ready (preemption while pending, `waitForPodsReady` timeout) returns the job to `Queued` rather than failing — Kueue re-suspends the object and the sensor keeps waiting. Eviction of a cluster that already went ready and has a Ray job running on it is different, and this RFC does not pretend otherwise: the workers die, the running task fails, and recovery rides on Uniflow's existing task-level retry semantics (a fresh submission, from scratch, re-queued like any new job). Transparent mid-run requeue would need checkpoint-aware resubmission and is out of scope.

**Two prerequisites in the current Ray path.** Neither is optional; both are sequenced into Phase 1.

- *The control plane must stop killing suspended clusters.* Today a Kueue-suspended RayCluster is actively destroyed by Michelangelo: the k8s engine maps KubeRay's `suspended` state to `RAY_CLUSTER_STATE_UNKNOWN` (`ray.go`), KubeRay reports `HeadPodReady=False` because the pods are intentionally absent, and the ray cluster controller treats UNKNOWN-plus-terminal-pod-errors as FAILED and terminates the cluster (`go/components/ray/cluster/controller.go`). Open PR #1700 ("Support the KubeRay suspended cluster state in the MA API") documents this exact failure observed live on a Kueue-gated cluster and introduces a non-terminal `RAY_CLUSTER_STATE_SUSPENDED`. This RFC treats that PR as its first prerequisite and the anchor for the `Queued` condition above.
- *Queueing must escape the 30-minute creation budget.* The entire create-and-wait path shares one 1800-second budget: `task.star` passes `DEFAULT_CREATE_CLUSTER_TIMEOUT_SECONDS` to `ray.create_cluster`, the Go plugin sets it as the readiness sensor's `ExpirationInterval`, and on expiry the plugin terminates the cluster and fails the task. Left alone, a job legitimately queued behind quota for 31 minutes fails — which would contradict this RFC's central promise. The design therefore splits the budget by phase: while the cluster is `SUSPENDED` (awaiting admission), the sensor does not consume the creation budget — the admission wait is governed by Kueue-native mechanisms (and an optional operator-set queue timeout), and is surfaced through the `Queued` condition; the existing 1800-second budget applies only from admission (suspend flipped false) to readiness, which is the phase it was always sized for. This touches the sensor activity and the Ray plugin, and is called out as its own PR in Phase 1 rather than a footnote.

**Spark.** The `SparkJob` proto, job type (`go/components/jobs/common/types/types.go`), and controller scaffolding exist, but the k8s mapping is explicitly not implemented yet (`mapper.go` returns "spark job mapping not implemented") and Spark-on-K8s launch is roadmap-Planned. Since this RFC was first sketched, the mechanism question has largely answered itself: Kueue v0.17 added a first-class integration for the Kubeflow Spark Operator's `SparkApplication` CRD (behind the `SparkApplicationIntegration` feature gate) — suspend-based and gang-admitted like the Ray integrations, requiring spark-operator v2.4.0+ and with dynamic allocation explicitly rejected by webhook for managed applications. So the default answer is: when the Spark launch path lands, a Kueue-managed cluster admits Spark work through that integration, with the pod-group integration or an explicit two-pod-set Workload as fallbacks only if the launch path ends up not using the operator CRD. Two alignment tasks come with that and should be named now: the vendored Spark operator Go types under `go/thirdparty/k8s-crds/apis/sparkoperator.k8s.io/` date from the pre-Kubeflow (GoogleCloudPlatform) operator and predate the suspend field, and the operator version on compute clusters must be v2.4.0+ — both belong to whoever builds the Spark mapping. The `scheduler_type` field, queue resolution, label allowlist, and status reflection are all job-type agnostic, so no API changes are needed to extend coverage.

**Relation to PR #1129.** That PR proved the mechanism (label a RayCluster, Kueue suspends and admits it, quota is tracked) with a deliberately minimal surface: a single global `KUEUE_QUEUE_NAME` env var read by the Ray plugin, Helm plumbing, and sandbox demo commands. The two review comments it drew are both structural and both adopted here: label propagation happens through an explicit allowlist (the direction its review asked for), and the queue topology is explicit (ClusterQueue per compute cluster, LocalQueue per project) instead of one ambiguous global queue name. The other generalizations: queue labeling moves from the Python plugin layer into the control plane so it is uniform across job types and not user-forgeable, opt-in moves from an env var to a typed per-cluster proto field, and admission status becomes visible in job conditions. The sandbox `--install-kueue` work in #1129 is directly reusable for the e2e story, and landing it (rebased) as the first PR of this series would be an ideal outcome for that stalled effort.

## APIs and CRDs

**Additive proto field.** One new enum and field on `ClusterSpec` in `proto/api/v2/cluster.proto` (field 4 and 6 are reserved; 8 is the next free number):

```proto
// SchedulerType selects the admission backend for jobs dispatched
// to this cluster.
enum SchedulerType {
  // Invalid/unset: default scheduler behavior (immediate admission).
  // (INVALID = 0 follows the zero-value convention used across
  // proto/api/v2 enums.)
  SCHEDULER_TYPE_INVALID = 0;

  // Default internal queue with immediate admission.
  SCHEDULER_TYPE_DEFAULT = 1;

  // Kueue-managed admission: jobs dispatched to this cluster are
  // gated by Kueue quota and admitted all-or-nothing.
  SCHEDULER_TYPE_KUEUE = 2;
}

message ClusterSpec {
  // ... existing fields 1-3, 5, 7 unchanged; 4 and 6 reserved ...

  // Admission backend for this cluster. Optional; unspecified
  // preserves existing behavior.
  SchedulerType scheduler_type = 8;
}
```

This is strictly additive: no field renames, removals, or semantic changes to existing fields, per the project's SemVer policy for `proto/api/v2`.

**Control plane configuration** (YAML config consumed by the scheduler module; names bikesheddable):

```yaml
jobs:
  scheduler:
    backend: kueue            # "default" | "kueue"; default: "default"
    kueue:
      # Convention for resolving a project's LocalQueue on the target
      # cluster. "{project}" is substituted with the project name.
      localQueueTemplate: "ma-{project}"
      # Explicit overrides win over the template.
      localQueueOverrides:
        some-project: "custom-queue"
```

**Helm values** (nested under `controllermgr` like the rest of the controller-manager domain config, and explicit about which level each knob configures, addressing the #1129 review comment about queue ambiguity):

```yaml
controllermgr:
  scheduler:
    backend: kueue
    kueue:
      localQueueTemplate: "ma-{project}"

sandbox:
  installKueue: true   # e2e/sandbox only; production Kueue is operator-managed
```

**Kueue objects** (operator-managed on each Kueue-enabled compute cluster; shipped as documented examples, not created by Michelangelo in the first phase):

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cluster-compute-0
spec:
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
      flavors:
        - name: default-flavor
          resources:
            - name: "cpu"
              nominalQuota: 512
            - name: "memory"
              nominalQuota: 2Ti
            - name: "nvidia.com/gpu"
              nominalQuota: 64
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  name: ma-team-a
  # All Ray objects are dispatched into the compute cluster's
  # "default" namespace today (RayLocalNamespace), and a LocalQueue
  # must live in the workload's namespace, so per-project queues are
  # distinguished by name, not namespace.
  namespace: default
spec:
  clusterQueue: cluster-compute-0
```

**Labels on dispatched objects** (set by the control plane through the `mapLabels` allowlist, not user input):

```yaml
metadata:
  labels:
    kueue.x-k8s.io/queue-name: ma-team-a
```

No new Michelangelo CRDs are introduced. The Kueue Workload objects are created and owned by Kueue's own integrations. No gRPC or REST surface changes beyond the proto field above; the Python SDK is unaffected.

## Alternatives considered

### Alternative A: Volcano

**Pros:** Mature CNCF project; gang scheduling via PodGroup is its original core competency; queue and fair-share support; existing Spark operator integration; the operator guide names it alongside Kueue as an example backend. **Cons:** Volcano runs as an additional scheduler alongside kube-scheduler — workloads opt in via `schedulerName: volcano` (its docs describe it as supplementing Kubernetes scheduling; it can optionally be configured as the sole scheduler, but that is not the typical pattern). Either way it is a second scheduler binary for adopters to operate, with upgrades and scheduling behavior to manage next to kube-scheduler, where Kueue composes at the admission layer without touching pod scheduling. Its queueing model is scheduler-internal rather than expressed as declarative quota APIs. The broader ecosystem signal points the other way: Kubeflow's trainer and KubeRay both document Kueue as the queueing layer, and upstream Kubernetes is absorbing gang scheduling primitives compatible with Kueue's model (KEP-4671, alpha in v1.35). **Why not chosen:** Composition beats replacement for a control plane that targets adopters' existing clusters. Nothing in this design blocks a future Volcano backend behind the same `JobQueue` interface and `scheduler_type` enum if demand appears; the enum is open for extension.

### Alternative B: Apache YuniKorn

**Pros:** Strong Spark-on-K8s heritage; hierarchical queues and fair sharing; app-level (gang) scheduling. **Cons:** Also a scheduler replacement with its own placement engine, so it overlaps not just admission but also the roadmap's planned resource pool selection work; smaller footprint in the ML-platform ecosystem this project competes in; Ray integration is not first class. **Why not chosen:** Same composition argument as Volcano, plus weaker alignment with the Ray-first workload mix.

### Alternative C: scheduler-plugins coscheduling

**Pros:** Lightweight; gang scheduling via PodGroup CRD with the standard scheduler framework; no quota system to operate. **Cons:** Gang only. It solves the deadlock symptom but provides no quota, no fair sharing, no queueing, and no visibility into why a job waits, which is the larger half of the problem statement. Requires installing an out-of-tree scheduler profile on every compute cluster anyway. **Why not chosen:** Does not close the multi-tenancy gap; adopting it now would mean a second migration later.

### Alternative D: Build queueing and gang admission into Michelangelo

**Pros:** No external dependency; full control over semantics; could be tailored to Michelangelo's federation model. **Cons:** Quota, fair sharing, borrowing, preemption, and all-or-nothing admission with requeue is a multi-year project that Kueue has already built and continues to evolve with dedicated maintainers; an in-house version would start behind and fall further behind, and adopters would have to learn a bespoke system instead of one they may already operate. **Why not chosen:** The scheduler code itself anticipates external backends (the operator guide's Option 2), and the internal FIFO queue comment already says "We should replace this with priority queues when supporting job priority", pointing away from growing bespoke scheduling logic.

### Alternative E: Kueue on the control plane cluster instead of per compute cluster

**Pros:** Single Kueue installation; the control plane could gate dispatch centrally; no per-cluster Kueue lifecycle. **Cons:** Control-plane Kueue cannot see compute-cluster pod scheduling, so `waitForPodsReady` and flavor-level accuracy are lost; quota would be tracked against objects that do not run where Kueue runs; it drifts toward reimplementing MultiKueue semantics by hand. **Why not chosen:** Per-cluster Kueue matches how Kueue is designed to run, matches the topology PR #1129 validated, and keeps admission observing the same cluster it admits into. Cross-cluster queueing is explicitly deferred (see non-goals and open questions).

## Open questions

- [ ] Minimum supported Kueue version. Current Kueue releases serve `v1beta2` (first exposed in v0.15.0, with v1beta1 deprecated and scheduled out in subsequent minors; latest release at time of writing is v0.19.x). Proposal: target `v1beta2` and set the floor at the oldest release that both serves it and carries the `clusterSelector` skip semantics this design relies on — v0.15 — raising it if the Spark phase needs the SparkApplication integration (v0.17). Worth noting for reviewers of #1129: its sandbox configs declared `v1beta2` objects while installing Kueue v0.9.1, which never served that API — one more reason to pin the floor explicitly in the operator guide. Maintainer preference?
- [ ] Should Michelangelo manage LocalQueue lifecycle (create `ma-<project>` in the dispatch namespace when the target cluster is Kueue-managed), or are queues strictly operator-managed with Michelangelo only validating existence? First phase proposes operator-managed plus a clear dispatch-time error condition; auto-provisioning is a natural phase 2.
- [ ] Per-project namespaces on compute clusters. This design accepts today's reality that all Ray objects dispatch into `default` (`RayLocalNamespace`), so LocalQueues are name-scoped rather than namespace-scoped and namespace-selector isolation is unavailable. Materializing per-project namespaces on compute clusters would strengthen tenancy (and this topology would inherit it transparently — each LocalQueue simply moves into its project's namespace), but it touches the mapper, RBAC, and cleanup, and deserves its own proposal. Should this RFC's phase 2 reserve it, or is it separate work?
- [ ] Where should the project-to-LocalQueue mapping live long term: control plane config (proposed), or a field on the project/job protos? Config avoids API surface until the model is proven, but a proto field would make the mapping visible to the UI.
- [ ] Spark: the default mechanism is now Kueue's SparkApplication integration (see High-level architecture), but the choice interlocks with whichever operator and CRD version the Planned Spark-on-K8s launch path adopts — the vendored pre-Kubeflow operator types need refreshing regardless. This should be settled together with whoever builds the Spark mapping in `k8sengine`.
- [ ] Interplay with `SchedulingSpec.preemptible` and the priority TODO (#929): mapping preemptible to a low Kueue WorkloadPriorityClass is the obvious first step, but that deserves its own small design note once this backend exists. Should this RFC reserve the mapping or stay silent?
- [ ] `waitForPodsReady` is Kueue-installation-wide configuration (a per-ClusterQueue override is only an open upstream request, kueue#7542), so enabling it affects every Kueue-managed workload on that compute cluster, not just Michelangelo's. Should it be a documented operator setting only, or also surfaced (read-only) in cluster status so the control plane can explain requeue events precisely.
- [ ] Whether the maintainers want the stalled PR #1129, rebased, to land as the sandbox/e2e foundation of this series (my preference, with its author credited and involved), or prefer the sandbox work folded into the new PR series. Coordination has started on that PR's thread (the 2026-08-03 comment proposes exactly this split and flags the risk of two parallel Kueue implementations).

## Rollout strategy

Everything is opt-in at every layer, and the default path is untouched: with `backend: default` (or no config) the factory returns the existing `Scheduler` and no Kueue code runs. The `scheduler_type` proto field is additive, and a Kueue-enabled control plane still schedules normally onto clusters whose `scheduler_type` is unset, so fleets can migrate one cluster at a time.

**Phase 1 (alpha).** Land in this order, each PR independently reviewable:
1. The suspended-state prerequisite: land (or rebase and land) PR #1700's non-terminal `RAY_CLUSTER_STATE_SUSPENDED` so the control plane stops destroying Kueue-suspended clusters. Independently a bug fix; nothing else in this series works without it.
2. Additive `scheduler_type` proto field plus kubeproto conversion, no behavior.
3. `mapLabels` allowlist in `k8sengine/mapper.go` and `ray.go` with unit tests (independently useful: it introduces the deliberate, allowlisted label propagation the #1129 review asked for — today no labels propagate at all — without the wholesale-copy leak that review flagged).
4. Creation-budget split in the Ray plugin and sensor: suspended/queued time no longer consumes the 1800-second readiness budget (see architecture prerequisites).
5. `KueueJobQueue` package with the fx factory wiring, behind `backend: kueue`, with unit tests against a fake client; `Queued` condition reflection in status sync.
6. Sandbox and e2e: rebase or absorb #1129's `--install-kueue` sandbox work; k3d e2e with two compute clusters, one Kueue-managed and one not, proving mixed-fleet behavior, gang admission (a RayCluster that fits admits fully; one that exceeds quota starts zero pods), and requeue on `waitForPodsReady` timeout. Alpha stability per the project's per-component versioning policy: may change without notice.

**Phase 2 (beta).** Operator guide rewritten from "extension point" to "supported backend" with the ClusterQueue/LocalQueue topology, sizing, and migration examples; Helm values stabilized; `Queued` condition surfaced in the UI job detail; soak on real multi-tenant usage from at least one non-sandbox deployment. Migration notes required for any config change.

**Phase 3 (GA).** Declare the config surface and proto field stable after a full minor release of beta soak. Candidate follow-ups, each separately proposed: LocalQueue auto-provisioning, priority mapping (#929), Spark gang admission alongside the Spark launch path.

**Rollback.** At any phase, setting `backend: default` (or clearing a cluster's `scheduler_type`) restores current behavior for new jobs immediately. Jobs already queued in Kueue at rollback time either admit normally or can be released by removing the queue label on the compute cluster; the runbook documents both. No data migration exists in either direction, and the proto field can simply remain set but inert.

**Failure modes and mitigations** (tested in Phase 1 e2e where feasible):

- Kueue not installed or its webhooks down on a `SCHEDULER_TYPE_KUEUE` cluster: dispatch fails or objects hang unsuspended-but-unmanaged. The backend validates at dispatch that the target LocalQueue exists and sets a clear failure condition ("kueue_queue_not_found") instead of dispatching into a black hole.
- Workload inadmissible forever (quota smaller than the job): surfaced via the `Queued` condition message and a scheduler metric (`kueue_pending_workloads` style gauge tagged by cluster and project); optional per-job queue timeout is deliberately left to Kueue-native mechanisms rather than a new Michelangelo knob.
- Eviction of a workload (preemption or `waitForPodsReady`): before the cluster ever became ready, the job returns to `Queued` rather than `Failed` — Kueue re-suspends and requeues. After the cluster was ready and a Ray job was running, eviction kills workers and the task fails; recovery is Uniflow's existing task-level retry (a fresh submission that queues like any new job), not a transparent requeue. Both behaviors are stated in the status-reflection design and tested in e2e.
- Version skew between the pinned Kueue client API and the cluster's Kueue: detected at cluster registration (API discovery) and reported as a cluster condition rather than per-job errors.
- Label tampering: users cannot select or forge queues because the queue label is resolved and applied by the control plane after `mapLabels` filtering; user-supplied `kueue.x-k8s.io/*` labels are dropped.

**Testing plan summary.** Unit tests for queue resolution, label allowlisting, and factory selection; envtest-level tests for condition reflection; k3d e2e as above (mixed fleet, gang admit, gang reject, requeue, rollback to default backend); an upgrade test asserting byte-identical dispatch for the default backend before and after the change.

## References

- Prior art: michelangelo-ai/michelangelo PR #1129, "Integration Kueue into compute cluster scheduler for ray job" (validated the suspend/admit mechanism end to end; this RFC adopts both of its open review points; coordination underway on its thread since 2026-08-03).
- Prerequisite: michelangelo-ai/michelangelo PR #1700, "Support the KubeRay suspended cluster state in the MA API" (non-terminal `RAY_CLUSTER_STATE_SUSPENDED`; documents the control plane destroying Kueue-suspended clusters today).
- Extension seam and backend sketch: `docs/operator-guides/jobs/extend-michelangelo-batch-job-scheduler-system.md` (Option 2, "Custom JobQueue Implementation", names Kueue and Volcano; this design deliberately diverges from its hand-crafted-Workload sketch by delegating Workload creation to Kueue's own integrations).
- Code anchors: `go/components/jobs/scheduler/scheduler.go` (JobQueue), `go/components/jobs/scheduler/framework/interface.go` (AssignmentStrategy), `go/components/jobs/scheduler/framework/cluster_only_assignment_engine.go` (current default), `go/components/jobs/common/scheduler/queue.go` (FIFO queue and its priority-queue comment), `go/components/jobs/common/constants/constants.go` (`ClusterAffinityLabelKey`, `EnqueuedCondition`), `go/components/jobs/client/k8sengine/{mapper.go,ray.go,constants.go}` (dispatch mapping, `RayLocalNamespace`, Spark not yet implemented), `go/components/ray/cluster/controller.go` (terminal-state reconciliation), `go/worker/plugins/ray/starlark_module.go` (creation-budget sensor), `python/michelangelo/uniflow/plugins/ray/task.star` (gang-scheduling gap workaround, 30-minute budget), `proto/api/v2/cluster.proto` (ClusterSpec), `proto/api/v2/job.proto` (SchedulingSpec.preemptible, TODO #929).
- Kueue: https://kueue.sigs.k8s.io/ (ClusterQueue, LocalQueue, Workload, waitForPodsReady all-or-nothing, RayCluster/RayJob integrations, SparkApplication integration).
- Kueue `clusterSelector` skip semantics: kubernetes-sigs/kueue#7218 (RayJobs with `spec.clusterSelector` are ignored by Kueue).
- KubeRay + Kueue: https://docs.ray.io/en/latest/cluster/kubernetes/k8s-ecosystem/kueue.html
- Kubernetes KEP-4671, workload-aware scheduling (alpha in v1.35; KEP-5832's Workload/PodGroup split folded in for v1.36): https://github.com/kubernetes/enhancements/issues/4671
- Related in-repo issues: #929 (priority information TODO in job.proto).
