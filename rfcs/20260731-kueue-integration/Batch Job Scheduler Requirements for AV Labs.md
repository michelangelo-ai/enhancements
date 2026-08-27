# Batch Job Scheduling Requirements

Author: [Sidharth Padhee](mailto:sidharth.padhee@uber.com)

## Context

AV Labs runs RayClusters and RayJobs on OKE (Oracle Kubernetes Engine) using the default kube-scheduler ( No scheduling at the RayCluster level ). This works at a small scale but breaks down as the team scales up GPU workloads. Kueue is an open-source Kubernetes-native job admission system that solves these problems. 

---

## Problems Without a Batch Job Scheduler

|  | Problem | Impact | Priority | POC |
| :---- | :---- | :---- | :---- | :---- |
| **1** | **No job-level admission control** — kube-scheduler sees pods, not jobs. It has no concept of "this RayCluster needs a head \+ N workers as a unit." Pods get scheduled one at a time with no gate. | Jobs pile onto the cluster even when resources are exhausted, leading to cascading partial-starts and OOM kills. | P0 |  |
| **2** | **No gang scheduling** — kube-scheduler cannot guarantee all pods of a distributed job start together. A RayCluster might get its head node scheduled but workers are stuck in Pending. | Deadlocks and wasted GPU-hours: the head node sits idle consuming a GPU while workers wait for capacity that may never free up. | P0 |  |
| **3** | **No queue visibility** — without a queueing layer there is no way to see how many jobs are waiting, what their estimated start time is, or why a job hasn't started. | Users re-submit jobs thinking they failed, creating duplicates. No capacity planning data. Debugging "why is my job pending?" requires kubectl expertise. | P1 |  |
| **4** | **No hardware-aware routing** — kube-scheduler uses node affinity/selectors, but the user must manually configure these per job. There is no abstraction for "run on A100s, fall back to L40S." | Users misconfigure node selectors, jobs land on a wrong Compute cluster, or capacity on one GPU type sits idle while another is overloaded. | P1 |  |
| **5** | **No multi-tenant quota management** — Kubernetes ResourceQuotas are static, namespace-scoped, and binary (hard reject). There is no borrowing, lending, or fair sharing between teams. | One team can monopolize the GPU pool. Other teams' jobs queue indefinitely with no recourse. As AV Labs onboards more teams, this becomes a blocker. | P2 |  |
| **6** | **No priority or preemption across jobs** — kube-scheduler has PriorityClasses for pods, but no mechanism to preempt an entire low-priority job to make room for a high-priority one. | A long-running dev experiment blocks a time-sensitive production training run. No way to express "this job matters more" at the job level. | P2 |  |

---

## How Kueue Solves These Problems

| Problem | Kueue Solution |
| :---- | :---- |
| **No admission control** | Kueue holds jobs in a suspended state via LocalQueues. Jobs only unsuspend (create pods) when Kueue confirms the ClusterQueue has sufficient quota. No pods are created until admission succeeds. |
| **No gang scheduling** | Kueue evaluates the full resource footprint of a job (all pod sets: head \+ workers) before admitting. If the total doesn't fit, the job stays queued — no partial starts. Combined with `waitForPodsReady` and configurable timeouts, failed admissions are automatically requeued. |
| **No quota management** | ClusterQueues define per-resource `nominalQuota` (guaranteed), `borrowingLimit`, and `lendingLimit`. Queues in the same Cohort share unused capacity elastically. When a queue needs its guaranteed share back, Kueue reclaims borrowed resources via preemption. |
| **No priority/preemption** | Kueue supports WorkloadPriorityClasses and preemption policies (within a queue and across a cohort). Higher-priority jobs preempt lower-priority ones with minimal victim selection. Preemption fencing prevents unwanted cross-team evictions. |
| **No hardware routing** | ResourceFlavors (e.g., `a100-80gb`, `l40s`, `cpu-only`) define named hardware pools with node selectors and tolerations. Kueue auto-injects the correct node labels into admitted jobs. Users just pick a flavor; Kueue handles placement. |
| **No queue visibility** | Kueue exposes queue depth, pending workloads, and admission status via CRD status fields and `kueuectl`. Teams can see their position in the queue and why a job is waiting. Kueue creates a Workload CR for each RayCluster which has the status fields. We can watch the Kueue Workload CR to understand errors and expose them to MA RayCluster.  |
| **Crashlooping jobs hold GPUs** | Kueue’s waitForPodsReady feature (opt-in, must be explicitly configured) evicts and requeues jobs whose pods fail to reach or maintain Ready status. Covers both ImagePullBackOff (never Ready) and CrashLoopBackOff (loses Ready). On timeout, Kueue suspends the job (pods deleted, GPUs released), returns quota to the ClusterQueue, and requeues with exponential backoff. The timeout is configurable — set it to a few minutes to release GPUs fast. Per-queue quotas further confine the blast radius to the offending team’s share. |

---

## Glossary

| Term | Definition |
| :---- | :---- |
| **RayCluster / RayJob** | Kubernetes CRDs from KubeRay that define distributed Ray workloads (head node \+ worker pods). |
| **OKE** | Oracle Kubernetes Engine — the managed K8s service AV Labs runs RayClusters / RayJobs on. |
| **kube-scheduler** | The default Kubernetes pod scheduler. Assigns pods to nodes but has no job-level awareness. |
| **Kueue** | A Kubernetes-native job admission and queueing controller (kubernetes-sigs). Decides *when* jobs start, not *where* pods go. |
| **ClusterQueue** | Kueue CRD defining a cluster-wide resource pool with quotas and policies. |
| **LocalQueue** | Kueue CRD (namespaced) where users submit jobs. Points to a ClusterQueue. |
| **Cohort** | A group of ClusterQueues that can borrow/lend unused capacity to each other. |
| **ResourceFlavor** | Kueue CRD representing a named hardware class (e.g., A100 GPUs) with associated node selectors. |
| **AdmissionCheck** | A pluggable gate that must pass before Kueue admits a job (e.g., "GPU nodes are provisioned"). |
| **Gang scheduling** | All-or-nothing scheduling: either every pod of a job can run, or none start. |

