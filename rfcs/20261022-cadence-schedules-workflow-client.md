# RFC-20261022-cadence-schedules: Run recurring triggers as Cadence Schedules

- **Status:** Draft
- **Author(s):** @yangfan0328
- **Created:** 2026-10-22
- **Internal ERD:** n/a

---

## Problem statement

A recurring `TriggerRun` (a cron trigger today, an interval trigger once one is wired) is executed by the workflow engine behind the controller's `WorkflowClient` interface. The two providers behave differently:

- **Temporal** runs the trigger as a Temporal Schedule. Pause, resume and in-place updates (cron, notifications, input) are schedule operations, so `spec.action = PAUSE` produces a real `PAUSED` state and `RESUME` picks the schedule back up.
- **Cadence** runs the trigger with the legacy `CronSchedule` start option. A cron workflow cannot be paused, and its cron expression cannot change without terminating and restarting it. The Cadence adapter's `PauseTrigger`, `UnpauseTrigger` and `UpdateTrigger` return "not supported", so the same `TriggerRun` spec that pauses cleanly on Temporal errors on Cadence, and a cron edit silently does nothing until the trigger is recreated.

The controller already treats the two engines as one abstraction. The gap is entirely inside the Cadence adapter, and it became closable when `go.uber.org/cadence` v1.4.0 shipped a `ScheduleClient` (Create / Describe / Update / Delete / Pause / Unpause / Backfill / List) backed by server-side Cadence Schedules.

## Motivation

- `PAUSED` is part of the `TriggerRun` API contract, not a provider feature. Operators running Cadence get an error where Temporal users get a state transition.
- Every planned trigger feature builds on the same path: interval triggers need a first-run alignment time, auto-flip needs in-place updates, and catch-up on resume needs a policy knob. Doing these on the legacy cron option would mean terminate-and-restart on every change.
- The Cadence adapter is the one shipped by the Helm chart's bundled subchart, so the default install is the one with the gap.

## Goals

- The Cadence adapter implements every recurring-trigger operation on the interface (`StartWorkflow` with `CronSchedule`, `PauseTrigger`, `UnpauseTrigger`, `UpdateTrigger`, `DeleteTrigger`) using Cadence Schedules, with the same observable `TriggerRun` state machine as the Temporal adapter.
- The engine-neutral `StartWorkflowOptions` carries the schedule semantics both engines share (overlap policy, catch-up policy, window, catch-up start, start-paused) so the controller never branches on the engine.
- Triggers started with the legacy cron option before the change keep running, report status and can be killed; nothing requires a bulk migration.
- The chart and operator docs state the server requirement plainly, so a stale Cadence deployment fails at schedule creation with a clear error rather than behaving differently.

## Non-goals

- Exposing overlap or catch-up policy on the `TriggerRun` proto. The interface carries them; the user-facing surface stays as it is until there is a concrete ask.
- Replacing the one-shot backfill runner with `ScheduleClient.Backfill`. Schedule backfill re-fires the schedule's action at historical times, while the backfill runner enumerates occurrences itself and tags the runs; the two have different semantics and the runner stays.
- Jitter, `PauseOnFailure`, and other schedule features with no counterpart in the `TriggerRun` spec.
- Any change to the Temporal provider's behaviour beyond honouring the new interface fields.

## High-level architecture

```
TriggerRun controller ── WorkflowClient (engine-neutral) ──┬── Temporal adapter ── Temporal Schedule
                                                           └── Cadence adapter  ── Cadence Schedule  (new)
                                                                                 └─ legacy cron workflow (read/kill only)
```

**Schedule per trigger.** `StartWorkflow` with a non-empty `CronSchedule` creates a Cadence Schedule instead of a cron workflow. The schedule ID is the trigger's workflow ID with a `-schedule` suffix, mirroring the Temporal adapter, so a schedule can never collide with a legacy cron workflow that still holds the bare ID. The schedule's action is a `StartWorkflow` of the same trigger workflow type, on the same task list, with the same encoded input the cron workflow used to receive, and a workflow-ID prefix equal to the bare trigger ID. Cadence names each fired run `<prefix>-<occurrence time>`, exactly as Temporal does.

**Status and kill paths are unchanged.** The controller already looks up the trigger's open execution by exact workflow ID and treats "no open execution" as "the schedule is running". Fired runs never match the exact ID, so the Temporal-shaped status logic applies to Cadence as-is. `DeleteTrigger` deletes the schedule and then, if a run ID was recorded, terminates that workflow; this is what makes a legacy cron workflow killable through the same call.

**Pause / resume / update.** `PauseTrigger` and `UnpauseTrigger` map to schedule pause and unpause. `UpdateTrigger` mutates the schedule's cron expression and action input in place. Cadence's `ScheduleUpdate` carries no pause state, so a request that changes both the spec and the pause state is applied as two calls, pause first (so a paused-and-updated trigger never fires between the calls), unpause last. The adapter documents that this is not atomic.

**Non-idempotent create.** Schedule `Create` fails with "already exists" if a retry repeats a start whose first response was lost, or a reconciliation repeats a start it never recorded. The adapter follows the client's guidance: on a create error it `Describe`s the schedule and adopts it if it exists (pausing it when the start asked for a paused schedule), and only surfaces the error when the schedule is really absent.

**One-off starts.** For non-recurring starts the adapter returns the already-open execution when the engine reports the workflow ID as already started, which is the Temporal client's default behaviour; before, a repeated start on Cadence failed the trigger.

**Catch-up on create.** A start that asks for catch-up from a past time is implemented with `ScheduleClient.Backfill` over `[CatchUpFrom, min(now, window end)]` with the `Buffer` overlap policy, so replayed occurrences queue rather than drop. A backfill failure after a successful create is returned as a typed error alongside the execution, so the controller can mark the trigger as running but report the catch-up failure, as it already does for Temporal.

**Server requirement.** Cadence Schedules need Cadence server v1.4.1 or later with the scheduler worker enabled for the domain (dynamic config `worker.enableScheduler`). The Helm chart's Cadence subchart is bumped to the first release shipping v1.4.1 and the chart's dynamic-config defaults turn the key on; the operator guides state the requirement. A Cadence that does not meet it returns an error from schedule creation, which the controller surfaces on the `TriggerRun`.

## APIs and CRDs

No CRD or proto changes. The Go workflow-client interface (`go/base/workflowclient/interface`) gains:

```go
type StartWorkflowOptions struct {
    // existing: ID, TaskList, timeouts, CronSchedule, StartPaused,
    // ScheduleStartAt, ScheduleEndAt, CatchUpFrom
    OverlapPolicy OverlapPolicy // zero value = SkipNew
    CatchUpPolicy CatchUpPolicy // zero value = Skip
}

type OverlapPolicy int32 // Unspecified, SkipNew, Buffer, Concurrent, CancelPrevious, TerminatePrevious
type CatchUpPolicy int32 // Unspecified, Skip, One, All
```

- The Cadence adapter maps both enums onto the schedule policies one-to-one.
- The Temporal adapter maps `OverlapPolicy` onto Temporal's overlap enum instead of hard-coding `SKIP`, and rejects `CatchUpPolicy` values it cannot honour (`One`, `All`) with an explicit error rather than silently ignoring them.
- `PauseTrigger`, `UnpauseTrigger` and `UpdateTrigger` keep their signatures; their documented contract now states that a legacy Cadence cron workflow must be recreated to gain these operations, and that the Cadence pause state inside `UpdateTrigger` is applied as a separate call.

Helm: the Cadence subchart dependency moves from 1.1.0 (server v1.3.6) to 1.6.7 (server v1.4.1), and `cadence.dynamicConfig.values` gains `worker.enableScheduler: true`.

## Alternatives considered

### Alternative A: keep the legacy cron option and emulate pause with signals
Pause would be a signal the trigger workflow honours by skipping ticks; update would be a signal carrying the new spec.
**Pros:** no server version requirement; no new client surface.
**Cons:** the cron expression itself cannot change (it is a start option), so cron edits still need terminate-and-restart; pause state lives in workflow history and is lost on a restart; every planned feature (interval alignment, auto-flip) needs its own signal. The Temporal adapter already took the schedule route, so the two providers would keep diverging.
**Why not chosen:** it solves one symptom and leaves the structural gap.

### Alternative B: schedule ID equal to the bare workflow ID
**Pros:** one identifier per trigger.
**Cons:** a legacy cron workflow started before the change holds that ID as an open execution; the status path could not tell "schedule running" from "legacy cron workflow running", and a schedule's fired-run prefix would collide with the legacy ID. The Temporal adapter already uses the suffix.
**Why not chosen:** the suffix is what makes the legacy fallback work by construction.

### Alternative C: make pause-plus-update atomic by recreating the schedule
Delete and re-create the schedule with the new spec and state in one logical step.
**Pros:** no intermediate state.
**Cons:** recreation loses the schedule's run history and counters, and a failure between delete and create leaves the trigger with no schedule at all, which is worse than a brief paused window.
**Why not chosen:** two ordered calls with pause-first is strictly safer.

## Open questions

- [ ] Should the `TriggerRun` proto eventually expose `overlap_policy` (the one plausible ask is `Buffer`, "do not drop a tick while the previous fan-out is slow")? This RFC keeps it interface-only.
- [ ] Should `RESUME` carry a catch-up policy (`One` / `All`) for Cadence, where the engine supports it, with Temporal rejecting it? Today resume never replays on either engine.
- [ ] Is a one-time migration tool for legacy cron workflows worth adding, or is "recreate the trigger" acceptable for the small number of long-lived Cadence installs?

## Rollout strategy

1. **Code:** one implementation PR in the core repo adds the interface fields, the schedule-based Cadence adapter with the legacy read fallback, the Temporal mapping, unit tests for both adapters, and the chart/doc changes. No feature flag: a cron trigger created after the upgrade is a schedule; one created before keeps running as a cron workflow until it is recreated.
2. **Operators:** upgrade the bundled Cadence subchart (or an external Cadence) to server v1.4.1 or later and enable `worker.enableScheduler` for the domain before upgrading the controller. The Helm change is a breaking configuration change and ships with a migration note.
3. **Verification:** the existing TriggerRun lifecycle integration tests (create, pause, resume, kill, cascade delete, one-shot backfill) run against a Cadence sandbox on the schedule path, in addition to the Temporal run they already have.
4. **Rollback:** reverting the controller returns new triggers to the cron option; schedules created in the meantime keep firing until deleted, so a rollback should be paired with deleting or pausing the schedules of affected triggers.
5. **Follow-ups that build on this path:** the interval trigger (first-run alignment through the schedule's start time) and auto-flip (in-place spec updates).

## References

- Cadence Schedules announcement: https://cadenceworkflow.io/blog/2026/06/23/cadence-schedules
- `go.uber.org/cadence` v1.4.0 release notes (ScheduleClient; server v1.4.1 or later): https://github.com/cadence-workflow/cadence-go-client/releases/tag/v1.4.0
- Cadence server v1.4.1 release: https://github.com/cadence-workflow/cadence/releases/tag/v1.4.1
- Cadence Helm chart releases (first chart shipping server v1.4.1): https://github.com/cadence-workflow/cadence-charts/releases
- Core repo: `go/base/workflowclient` (interface and both adapters), `go/components/triggerrun` (controller)
