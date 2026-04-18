---
title: internal/scheduler
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/scheduler/capacity/config.go
  - /home/kimberly/repos/gastown/internal/scheduler/capacity/dispatch.go
  - /home/kimberly/repos/gastown/internal/scheduler/capacity/pipeline.go
  - /home/kimberly/repos/gastown/internal/scheduler/capacity/state.go
tags: [package, scheduler, capacity, dispatch, polecat, pure-functions]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/scheduler

The capacity-controlled polecat dispatch scheduler. Despite the name,
**`internal/scheduler/` has no Go files of its own** — the entire
implementation lives in the single subpackage
`internal/scheduler/capacity/`. This page documents that subpackage as
the scheduler family, since there's nothing else under the `scheduler`
namespace.

**Go package path:** `github.com/steveyegge/gastown/internal/scheduler/capacity`
**File count:** 4 go files (`config.go`, `dispatch.go`, `pipeline.go`,
`state.go`), 3 test files.
**Imports (notable):** stdlib only (`encoding/json`, `fmt`, `os`,
`path/filepath`, `strings`, `time`). Zero gastown-internal imports.
This is deliberate — the package holds the **pure, testable parts** of
the scheduler (types, plan computation, state I/O), leaving the impure
orchestration (bead queries, polecat spawning, rig resolution) in
`cmd/` where it can import beads/sling/rig without creating cycles.
The package comment at `capacity/config.go:1-3` states this split
explicitly.
**Imported by (notable):** the daemon's dispatch-cycle loop and the
[`gt scheduler`](../commands/scheduler.md) command family (status,
pause, resume, dry-run).

## What it actually does

### Scheduler configuration (`config.go`)

`SchedulerConfig` (`config.go:16-31`) is the town-wide scheduler
config, serialized as JSON. The configuration is **host-wide, not
per-rig**, because capacity control guards shared host resources (API
rate limits, memory, CPU).

Fields:

- `MaxPolecats *int` — max concurrent polecats across all rigs.
  Pointer-typed so "absent" is distinguishable from "zero."
- `BatchSize *int` — beads dispatched per heartbeat tick. Limits the
  spawn rate per 3-minute cycle.
- `SpawnDelay string` — Go duration string between spawns, to avoid
  Dolt lock contention.

The `MaxPolecats` value drives the whole scheduler mode
(`config.go:15-21, 69-73`):

- `-1` (default): direct dispatch. `gt sling` runs like it used to,
  with near-zero scheduler overhead. The bead skips the queue.
- `0`: direct dispatch, same as `-1`. Explicit "I thought about it and
  disabled the scheduler."
- `N > 0`: **deferred dispatch**. `gt sling` writes a labeled sling
  context bead, the daemon's scheduler cycle picks it up, and
  dispatch happens from the loop instead of the caller.

Accessors `GetMaxPolecats`, `GetBatchSize`, `GetSpawnDelay`,
`IsDeferred` (`config.go:46-73`) all accept a `nil` receiver and
return defaults. `ParseDurationOrDefault` (`config.go:76-85`) is a
tolerant helper — a malformed duration string falls back to the
caller-supplied fallback rather than erroring.

### Dispatch cycle (`dispatch.go`)

`DispatchCycle` (`dispatch.go:20-43`) is a generic capacity-controlled
orchestrator. **All domain logic is injected via callbacks** —
`AvailableCapacity`, `QueryPending`, `Execute`, `OnSuccess`,
`OnFailure` — so the core loop is pure and testable. The daemon wires
in the real bead queries and sling-dispatch implementation; tests wire
in fakes.

Two entry points:

- `Plan() (DispatchPlan, error)` (`dispatch.go:54-66`) — compute what
  would be dispatched, without executing. Used for `gt scheduler
  --dry-run`. Calls `AvailableCapacity`, then `QueryPending`, then
  delegates to `PlanDispatch` (the pure function in `pipeline.go`).
- `Run() (DispatchReport, error)` (`dispatch.go:72-126`) — execute one
  full cycle. For each item in the plan, call `Execute`; on success,
  retry `OnSuccess` up to `onSuccessRetries = 2` times
  (`dispatch.go:69, 95-104`); on `OnSuccess` failure after retries,
  count the dispatch as a failure (not a success) and invoke
  `OnFailure` with a wrapped `ErrOnSuccessFailed` (`dispatch.go:105-113`).
  Between successful spawns, sleep for `SpawnDelay` (skipping the
  delay after the final item, `:120-122`).

The `ErrOnSuccessFailed` (`dispatch.go:9-16`) type exists specifically
so the failure handler can distinguish "polecat launched, context
close failed" from "polecat never launched at all." The former
usually means the sling context bead wasn't cleaned up and will be
re-dispatched next cycle unless something else intervenes.

### Pipeline types and pure functions (`pipeline.go`)

This file holds the types the scheduler operates on and the one big
pure function, `PlanDispatch`.

**Types:**

- `PendingBead` (`pipeline.go:5-14`) — the scheduler's view of a
  dispatchable work item: sling context bead ID, work bead ID, title,
  target rig, description, labels, and parsed context fields.
- `SlingContextFields` (`pipeline.go:17-38`) — the JSON-serialized
  body of a sling context bead. Captures everything the daemon needs
  to reconstruct a `gt sling` call from cold: formula name, args,
  vars, merge mode, convoy, base branch, account, agent, mode flags,
  dispatch-failure counter, and last-failure timestamp. See also
  `ReconstructFromContext` (`pipeline.go:177-196`) which rebuilds
  `DispatchParams` from these fields.
- `DispatchPlan` (`pipeline.go:44-48`) — output of `PlanDispatch`:
  which beads to dispatch, how many were skipped, and a `Reason`
  string (`"capacity" | "batch" | "ready" | "none"`).
- `DispatchReport` (`dispatch.go:46-51`) — output of `Run`:
  dispatched, failed, skipped counts plus the reason.
- `DispatchParams` (`pipeline.go:159-174`) — the scheduler's view of
  what the dispatcher needs, a cleaned-up mirror of `cmd.SlingParams`.

**Constants:**

- `LabelSlingContext = "gt:sling-context"` (`pipeline.go:41`) — the
  label that identifies sling context beads in the bead store.
- `FailureRetry` / `FailureQuarantine` (`pipeline.go:51-58`) — the two
  failure actions.

**Pure functions:**

- `PlanDispatch(availableCapacity, batchSize, ready) DispatchPlan`
  (`pipeline.go:89-123`) — the scheduler's core decision logic. Takes
  the smallest of `capacity`, `batchSize`, and `len(ready)` as the
  dispatch count, picks a `Reason` string that tells the user *which*
  constraint was binding, and returns the plan. Readability over
  cleverness — the code explicitly distinguishes the three possible
  reasons with separate conditionals (`:110-116`).
- `AllReady` / `BlockerAware(readyIDs)` (`pipeline.go:67-83`) — the
  two built-in readiness filters. `BlockerAware` filters to beads
  whose work bead has no unresolved blockers, threading in a set of
  ready work-bead IDs from outside.
- `NoRetryPolicy()` / `CircuitBreakerPolicy(maxFailures)`
  (`pipeline.go:126-141`) — the two built-in failure policies.
- `FilterCircuitBroken(beads, maxFailures)` (`pipeline.go:144-156`) —
  removes beads that have exceeded the failure threshold, returning
  the filtered list and the count of removed beads.

### Runtime state (`state.go`)

`SchedulerState` (`state.go:13-19`) is the daemon's operational state:
whether the scheduler is paused, who paused it, when, and when the
last dispatch ran. It's stored at
`<townRoot>/.runtime/scheduler-state.json` (`state.go:22-24`) and
follows the same pattern as `deacon/redispatch-state.json`.

- `LoadState(townRoot)` (`state.go:34-59`) — load JSON. Handles the
  legacy `queue-state.json` filename for a migration path
  (`:40-47`). A truly missing file is not an error; it returns
  `&SchedulerState{}` meaning "not paused, never dispatched."
- `SaveState(townRoot, state)` (`state.go:64-98`) — atomic write via
  temp file + rename. Needed because `gt scheduler pause` and the
  dispatch loop's `RecordDispatch` can race on the same file.
- `SetPaused(by)` / `SetResumed()` (`state.go:101-112`) — mutate the
  pause fields with RFC3339 UTC timestamps.
- `RecordDispatch(count)` (`state.go:115-118`) — stamp the last
  dispatch time and count.

## Subdirectory structure

```
internal/scheduler/           <-- 0 Go files directly
  capacity/                   <-- the real package
    config.go
    dispatch.go
    pipeline.go
    state.go
    (+ tests)
```

The `scheduler` parent directory exists to suggest that more
scheduling packages might be added later (priority queues, fairness
policies, etc.) but today there's only `capacity/`. Anywhere in this
wiki that refers to "the scheduler package" means
`internal/scheduler/capacity`.

## Docs claim

The package comment (`config.go:1-3`) states the split clearly: "types
and pure functions for the capacity-controlled dispatch scheduler. The
impure orchestration (dispatch loop, enqueue, epic/convoy resolution)
stays in cmd but uses types and pure functions from this package."
This matches the implementation exactly.

## Drift

- **Naming mismatch between state and legacy state.** The migration
  path reads `queue-state.json` but the new file is
  `scheduler-state.json` (`state.go:22, 27`). The legacy name implies
  an older "queue" abstraction that was renamed; any docs still
  referring to a "queue state" file may be describing the old
  filename.
- **`ParseDurationOrDefault` swallows parse errors silently**
  (`config.go:82-84`). A typo in `spawn_delay` (e.g., `"500mss"`)
  falls back to zero without a warning. `gt scheduler status` or
  `gt doctor` would be the right place to surface that drift.

## Notes / open questions

- The dispatch loop's `onSuccessRetries = 2` (`dispatch.go:69`) hard
  codes "try three times total" with linear backoff (500ms * attempt,
  `:102`). If `OnSuccess` is closing a sling context bead and the
  bead store is flaky, a 1.5-second total retry window may not be
  enough — this is a future-tuning knob.
- `DispatchReport.Failed` double-counts the `OnSuccessFailed` case:
  dispatch succeeded but we treat it as a failure to avoid
  re-dispatch. The `Reason` field on the report doesn't reflect this
  — it's whatever `PlanDispatch` decided, not what actually happened
  at runtime.
- The pipeline's `Reason` strings (`"capacity" | "batch" | "ready" |
  "none"`) are not exported as constants — they're loose string
  literals. A consumer that wants to branch on the reason has to
  match them as strings.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [gt scheduler](../commands/scheduler.md) — the user-facing command
  surface (status, pause, resume, dry-run).
- [go-packages inventory](../inventory/go-packages.md).
