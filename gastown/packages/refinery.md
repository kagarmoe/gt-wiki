---
title: internal/refinery
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
sources:
  - /home/kimberly/repos/gastown/internal/refinery/manager.go
  - /home/kimberly/repos/gastown/internal/refinery/engineer.go
  - /home/kimberly/repos/gastown/internal/refinery/batch.go
  - /home/kimberly/repos/gastown/internal/refinery/score.go
  - /home/kimberly/repos/gastown/internal/refinery/types.go
tags: [package, agent-runtime, refinery, merge-queue, per-rig, batch, bisect]
---

# internal/refinery

Merge-queue processing for the Refinery — Gas Town's per-rig merge
queue agent. The package owns the Refinery tmux session lifecycle,
the merge-request types and their state machine, the priority
scoring function, the merge-queue loader that derives queue state
from beads (ZFC), the worker-claim protocol that lets multiple
Refinery workers cooperate, the actual merge engine (`Engineer`)
that runs quality gates and performs squash merges, and the
batch-then-bisect merger for throughput.

**Go package path:** `github.com/steveyegge/gastown/internal/refinery`
**File count:** 5 non-test `.go` files — `manager.go`, `engineer.go`
(the bulk: 2057 lines), `batch.go`, `score.go`, `types.go`.
**Role:** [`refinery` role](../roles/refinery.md) — domain persona
**CLI command:** [`gt refinery`](../commands/refinery.md)
**Imports (notable):** [`internal/beads`](beads.md),
[`internal/config`](config.md), `internal/constants`,
[`internal/crew`](crew.md), [`internal/events`](events.md),
`internal/git`, [`internal/mail`](mail.md),
[`internal/nudge`](nudge.md), `internal/rig`,
`internal/runtime`, [`internal/session`](session.md),
[`internal/style`](style.md), `internal/tmux`,
[`internal/util`](util.md).
**Imported by (notable):** [`gt refinery`](../commands/refinery.md)
(`/home/kimberly/repos/gastown/internal/cmd/refinery.go`), the
daemon's health-patrol paths, and any code that needs the
`MRStatus` / `MRPhase` / `FailureType` types as a shared vocabulary.

## What it actually does

ZFC-compliant: there is no state file. Running state is derived
from the rig's tmux session (`manager.go:74-79`); queue state is
derived from beads merge-request issues (`manager.go:329-381`);
claims live on the bead, not in package memory.

### Types — `types.go`

- `MergeRequest` struct (`types.go:14-44`) — the in-memory projection
  of a merge-request bead for display and transient processing.
  Fields: `ID`, `Branch`, `Worker`, `IssueID`, `SwarmID`,
  `TargetBranch`, `CreatedAt`, `Status`, `CloseReason`, `Error`.
- `MRStatus` (`types.go:48-59`) — the beads-aligned status:
  `MROpen`, `MRInProgress`, `MRClosed`. Uses bead statuses for
  consistency with issue tracking.
- `MRPhase` (`types.go:64-90`) — the v2 fine-grained state machine
  stored in checkpoint/claim fields: `ready`, `claimed`, `preparing`,
  `prepared`, `merging`, `merged`, `rejected`, `failed`. The beads
  `status` field stays at open/in_progress/closed for compatibility.
- `ValidPhaseTransitions` (`types.go:93-101`) — the allowed phase
  transitions, enforced by `ValidatePhaseTransition`
  (`types.go:104-118`). Terminal states: `merged`, `rejected`.
- `CloseReason` (`types.go:124-138`) — `merged`, `rejected`,
  `conflict`, `superseded`.
- `ValidateTransition` (`types.go:166-194`) — the coarser status-
  level validator: `open ↔ in_progress` round trips, terminal
  `closed` is immutable.
- `MergeRequest` methods `SetStatus`, `Close`, `Reopen`, `Claim`,
  `IsClosed`, `IsOpen`, `IsInProgress` (`types.go:198-249,303-311`).
- `FailureType` enum (`types.go:251-278`) — categorises merge
  failures so callers can branch on `conflict` vs `tests_fail` vs
  `build_fail` vs `flaky_test` vs `push_fail` vs `fetch_fail` vs
  `checkout_fail`.
- `FailureType.FailureLabel()` (`types.go:280-291`) — maps failure
  types to beads labels: `needs-rebase`, `needs-fix`, `needs-retry`.
- `FailureType.ShouldAssignToWorker()` (`types.go:294-301`) — true
  for conflicts, test failures, build failures, flaky-test failures.
- `QueueItem` (`types.go:141-145`) — position + MR + age, for
  display.
- `DefaultClaimTTLMinutes = 30` (`types.go:121`).

### Manager — `manager.go`

- `NewManager(r *rig.Rig) *Manager` (`manager.go:50-56`) — bound to
  the rig. Single constructor.
- `Manager.SessionName() string` (`manager.go:65-67`) — returns
  `session.RefinerySessionName(session.PrefixFor(m.rig.Name))`.
- `Manager.IsRunning() (bool, error)` (`manager.go:74-79`) — wraps
  `tmux.CheckSessionHealth`. Returns true only if session is
  `tmux.SessionHealthy` — zombie sessions (tmux alive, agent dead)
  are NOT reported running.
- `Manager.IsHealthy(maxInactivity) tmux.ZombieStatus`
  (`manager.go:86-89`) — the stricter check that also flags hung
  sessions where Claude is alive but hasn't produced output.
- `Manager.Status() (*tmux.SessionInfo, error)` (`manager.go:93-106`).
- `Manager.Start(foreground bool, agentOverride string) error`
  (`manager.go:113-271`) — the 150-line start path. Refuses
  foreground mode (deprecated). Detects and kills zombie sessions.
  Computes `refineryRigDir = <rig>/refinery/rig` with auto-repair
  fallback via `.repo.git` worktree add (see
  `repairRefineryWorktree`) and a last-resort fall-through to
  `<rig>/mayor/rig`. Loads runtime settings via
  `runtime.EnsureSettingsForRole`, builds a startup prompt with a
  patrol beacon, generates a UUID run ID for GASTA tracing,
  creates the tmux session with the command directly
  (`NewSessionWithCommand` — avoids the send-keys race from
  gh#280), sets `GT_REFINERY=1` and other env vars, applies theme,
  accepts startup dialogs, waits for the runtime to be ready,
  starts the nudge poller (`nudge.StartPoller`), runs the startup
  fallback chain, tracks the session PID, and records the agent
  instantiation event for GASTA.
- `Manager.repairRefineryWorktree(dir) error` (`manager.go:278-308`)
  — rebuilds a missing `refinery/rig` worktree from the shared
  bare `.repo.git` when the original has been pruned or deleted.
  Self-heals instead of failing startup.
- `Manager.Stop() error` (`manager.go:312-324`) — kills the tmux
  session. Returns `ErrNotRunning` if there wasn't one.
- `Manager.Queue() ([]QueueItem, error)` (`manager.go:329-381`) —
  queries beads for open merge-request issues via
  `beads.New(m.rig.BeadsPath()).ListMergeRequests`, filters by
  `gt:merge-request` label, keeps only `status=="open"`, filters by
  rig name (wisps are shared across all rigs per gh#2718), scores
  each issue via `calculateIssueScore`, sorts highest-first, and
  returns position-numbered `QueueItem`s. The Queue method is the
  one the `gt refinery queue` / `gt refinery status` CLI paths use
  and is the canonical answer to "what's the queue right now".
- `Manager.calculateIssueScore(issue, now) float64`
  (`manager.go:395-424`) — wraps the stateless `ScoreMRWithDefaults`
  from `score.go`.
- `Manager.issueToMR(issue) *MergeRequest` (`manager.go:427-462`) —
  converts a beads issue to an `MergeRequest`, pulling branch,
  worker, source issue, target, and created-at from
  `beads.ParseMRFields`. Falls back to the rig's default branch.
- `Manager.FindMR(idOrBranch)` (`manager.go:505-530`) — matches by
  ID, branch, `polecat/`-prefixed branch, or ID prefix.
- `Manager.RejectMR(idOrBranch, reason, notify)`
  (`manager.go:550-580`) — closes the MR bead with reason
  `"rejected: <reason>"`, updates in-memory state, optionally
  nudges the polecat worker.
- `Manager.PostMerge(idOrBranch) (*PostMergeResult, error)`
  (`manager.go:594-642`) — closes the MR bead with reason
  `"merged"` and force-closes the source issue with
  `"Merged in <MR-ID>"`. Uses `ForceCloseWithReason` so attached
  molecule steps don't block the close. Branch deletion is the
  caller's responsibility (Manager has no git access).
- `Manager.notifyWorkerRejected(mr, reason)` (`manager.go:645-657`)
  — nudges (not mails) the polecat worker with a rejection summary.
- `Manager.Retry`, `Manager.RegisterMR` (`manager.go:535-545`) —
  deprecated no-ops; the Engineer handles retry autonomously and
  MRs register via beads creation.
- Sentinel errors: `ErrNotRunning`, `ErrAlreadyRunning`,
  `ErrNoQueue`, `ErrMRNotFound`, `ErrMRNotFailed`
  (`manager.go:31-36, 492-495`).

### Engineer — `engineer.go`

The actual merge engine. Where `Manager` owns the tmux session and
surface queue view, `Engineer` owns the merge pipeline: claiming,
gate-running, squash-merging, pushing, post-merge cleanup, and
escalation. Engineer holds the rig, beads client, git client,
merge-queue config, output writer, mail router, and the
`mergeSlotEnsureExists` / `mergeSlotAcquire` / `mergeSlotRelease`
closures (`engineer.go:253-266`).

- `NewEngineer(r) *Engineer` (`engineer.go:269-301`) — determines
  the git working directory (`refinery/rig` then falling back to
  `mayor/rig`), wires up beads/git clients, sets default merge
  slot retries (10) and backoff (500 ms).
- `LoadConfig()` (`engineer.go:310-437`) — reads `config.json`
  and applies the `merge_queue` section over defaults. Handles
  duration parsing for `poll_interval`, `stale_claim_timeout`,
  per-gate timeouts. Validates phase values for gates.
- `MergeQueueConfig` struct (`engineer.go:103-176`) — the big
  config shape: `Enabled`, `OnConflict`, `RunTests`, `TestCommand`,
  `DeleteMergedBranches`, `RetryFlakyTests`, `PollInterval`,
  `MaxConcurrent`, `StaleClaimTimeout`, `Gates` map,
  `GatesParallel`, `StaleClaimWarningAfter`,
  `StaleClaimCriticalAfter`, `MaxRetryCount`, `AutoPush`,
  `MergeStrategy` (`direct` or `pr`), `RequireReview`, `Batch`.
- `GateConfig` struct (`engineer.go:74-87`) with `GatePhase`
  (`GatePhasePreMerge`, `GatePhasePostSquash`, at
  `engineer.go:62-72`). Post-squash gates run after the squash but
  before push, so a failure resets the merge.
- `ProcessResult` struct (`engineer.go:453-463`) — per-MR outcome
  including `SlotTimeout`, `BranchNotFound`, `NoMerge`,
  `NeedsApproval` flags for post-processing dispatch.
- `doMerge(ctx, branch, target, sourceIssue, skipGates...)`
  (`engineer.go:466+`) — the core merge engine: honours the
  `no_merge` flag (gh#2778), verifies the local branch exists,
  checks out target, performs the squash merge, runs gates in
  two phases, acquires the main-push slot via beads-backed
  coordinated lock, pushes, and reports results.
- `doMergePR(ctx, branch, target)` (`engineer.go:710+`) — the
  `merge_strategy=pr` variant; uses `gh pr merge` to respect
  branch protection and honour GitHub review requirements.
- `acquireMainPushSlot(ctx)` (`engineer.go:784+`) — beads-backed
  distributed lock so concurrent Refinery workers can't race on
  pushing to main. Uses `mergeSlotSeq` + nanosecond timestamps for
  a unique holder ID (Windows timer resolution workaround at
  `engineer.go:246-249`).
- `runGate`, `runGates`, `runGatesForPhase`
  (`engineer.go:901-1042`) — the gate execution loop with
  optional parallelism and per-gate timeouts.
- `syncCrewWorkspaces()` (`engineer.go:1044+`) — pushes merged
  changes into crew workspaces via the
  [`crew` package](crew.md).
- `ProcessMRInfo(ctx, mr)` (`engineer.go:1075+`) — the public
  per-MR entry. Used by `HandleMRInfoSuccess` /
  `HandleMRInfoFailure` (`engineer.go:1107+, 1225+`) to branch
  on result type and update bead state, escalate, re-slinging,
  etc.
- `createConflictResolutionTaskForMR` (`engineer.go:1342+`) —
  spawns a synthesis task bead to handle a conflicting MR; the
  synthesis bead is the mechanism by which a FRESH polecat
  re-implements the work (see the Long help quoted on
  [`gt refinery`](../commands/refinery.md)).
- `ClaimMR`, `ReleaseMR` (`engineer.go:1780-1802`) — parallel-worker
  coordination. `ClaimMR` writes the worker ID to the bead's
  claim field; claims older than `StaleClaimTimeout` (default
  30 min) are eligible for re-claim per `isClaimStale`
  (`engineer.go:47-56`).
- `ListReadyMRs`, `ListBlockedMRs`, `ListAllOpenMRs`
  (`engineer.go:1534-1692`) — the bead-sourced queue views used by
  `gt refinery ready`, `gt refinery blocked`.
- `ListQueueAnomalies(now)` (`engineer.go:1693+`) — detects
  `stale-claim` and `orphaned-branch` problems using the
  configured warning/critical thresholds. Returned via
  `MRAnomaly` (`engineer.go:230-237`).
- `postMergeConvoyCheck(mr)`, `notifyDeaconConvoyFeeding(mr)`,
  `checkAndCloseCompletedConvoys(townRoot, townBeads)`,
  `notifyConvoyCompletion`, `landConvoySwarm`
  (`engineer.go:1804-2052`) — convoy-awareness: when a merge
  completes, the Engineer checks whether it closed the last
  tracked issue in a convoy and drives the
  [`convoy-launch` workflow](../workflows/convoy-launch.md)'s
  landing phase.
- `pruneStaleRemoteRefs()` (`engineer.go:2053+`) — housekeeping.
- `IsBeadOpen(beadID)` (`engineer.go:1450+`) — helper for the
  blocked-MR filter.

### Batcher — `batch.go`

Batch-then-bisect merge queue: instead of merging one MR at a
time, the Engineer can assemble a batch, stack them on target,
run gates once, and on failure use bisection to find the
culprit(s).

- `BatchConfig` struct (`batch.go:10-26`) — `MaxBatchSize` (default
  5), `BatchWaitTime` (default 30s), `RetryBatchOnFlaky` (default
  true).
- `BatchResult` struct (`batch.go:37-53`) — `Merged`, `Culprits`,
  `Conflicts`, `MergeCommit`, `Error`.
- `Engineer.AssembleBatch(readyMRs, config)` (`batch.go:58-92+`) —
  picks up to `MaxBatchSize` MRs, excluding any whose
  `BlockedBy` target isn't already in the batch.
- `Engineer.BuildRebaseStack(ctx, batch, target)`
  (`batch.go:97+`) — rebases each MR on top of the last, producing
  a stacked linear history. Returns the stacked set and any MRs
  that could not be stacked (conflicts).
- `Engineer.ProcessBatch(ctx, batch, target, batchCfg)`
  (`batch.go:199+`) — the batch merge entry point: stack, gate,
  verify-and-push, and on gate failure, bisect.
- `Engineer.bisectBatch(ctx, batch, target)` (`batch.go:422+`) —
  binary-search bisection: split the batch in half, merge and test
  each half, recurse into the failing half. Identifies culprit(s).
- `Engineer.bisectRight(ctx, knownGood, right, target)`
  (`batch.go:488+`) — the right-half recursion helper.
- `Engineer.resetAndRebuildStack(mrs, target)` (`batch.go:544+`)
  — resets the working tree and rebuilds the stack after a
  bisection step.

### Scorer — `score.go`

- `ScoreConfig` struct (`score.go:13-41`) — tunable weights:
  `BaseScore` (1000), `ConvoyAgeWeight` (10 pts/hr),
  `PriorityWeight` (100 * (4-priority)), `RetryPenalty`
  (50/retry), `MRAgeWeight` (1 pt/hr), `MaxRetryPenalty` (300).
- `DefaultScoreConfig()` (`score.go:44-53`).
- `ScoreInput` struct (`score.go:55+`) — decouples the score
  function from any specific MR struct.
- `ScoreMRWithDefaults(input) float64` — called from
  `Manager.calculateIssueScore` for the queue ordering.

### Notable design choices

- **Three different beads access patterns coexist.** `Manager`
  uses `beads.New` directly for the queue view; `Engineer` also
  uses `beads.New` but through a cached `beadsClient` for
  pipeline operations and the merge-slot protocol; `gt refinery
  ready/blocked/unclaimed` has the CLI path using the Engineer
  directly for its read-only list views. See
  [`gt refinery`](../commands/refinery.md) notes for why all
  three exist.
- **Wisps live in the same table as MRs.** Merge requests ARE
  wisps — they're stored in the `wisps` table and queried via
  `beads.ListMergeRequests`. The `Rig` filter at `manager.go:351-
  355` exists because "wisps are shared across all rigs
  (gh#2718)": the rig has to filter by its own name even though
  the query looks like it's scoped. See
  [`wisp` concept](../concepts/wisp.md).
- **ZFC throughout.** No state file. Running state from tmux,
  queue state from beads, claims on bead fields, anomalies derived
  from timestamps.
- **MR phases vs status.** The beads `status` field stays at
  open/in_progress/closed for compatibility with the rest of the
  tracker, while the fine-grained `MRPhase` lives in
  checkpoint/claim fields. Two state machines on one bead.
- **Main-push slot protocol.** Concurrent Refinery workers never
  push to main simultaneously — the beads-backed
  `MergeSlotAcquire` / `MergeSlotRelease` serializes pushes. A
  holder ID combines nanosecond timestamp + atomic counter for
  uniqueness on Windows timers (`engineer.go:246-249`).
- **Batch-then-bisect is opt-in.** When `Batch.MaxBatchSize <= 1`
  (the default off state), the Engineer falls through to the
  sequential single-MR path via `processSingleMR`.
- **Convoy completion is a side effect of merging.** The Engineer
  closes the last tracked issue of a convoy, detects convoy
  completion, and fires the landing flow without the convoy
  needing its own polling loop. See
  [`convoy-launch` workflow](../workflows/convoy-launch.md).

## Related wiki pages

- [`refinery` role](../roles/refinery.md) — domain persona.
- [`gt refinery`](../commands/refinery.md) — CLI surface, 11
  subcommands.
- [`wisp` concept](../concepts/wisp.md) — why MRs are wisps.
- [`convoy` concept](../concepts/convoy.md) — cross-rig work units
  this package's post-merge path closes.
- [`convoy-launch` workflow](../workflows/convoy-launch.md) — the
  launch → watch → land flow the Engineer participates in.
- [`internal/beads`](beads.md) — `Beads.ListMergeRequests`,
  `ParseMRFields`, `ForceCloseWithReason`, `MergeSlotAcquire`.
- [`internal/config`](config.md) — role-agent resolution and
  runtime settings.
- [`internal/session`](session.md) — `RefinerySessionName`,
  `PrefixFor`, `BuildStartupPrompt`.
- [`internal/crew`](crew.md) — crew workspace sync after merges.
- [`internal/events`](events.md) — refinery result events.
- [`internal/mail`](mail.md), [`internal/nudge`](nudge.md) —
  worker notifications on rejection.
- [`witness` package](witness.md) — sibling per-rig agent.
- [`polecat` package](polecat.md) — where work originates.
- [`mayor` package](mayor.md) — escalation target for exhausted
  retries.

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Failed merge leaves partial state:** The refinery's merge
  workflow involves multiple git operations (checkout, merge, push).
  With 164 silent error suppressions across the package, many
  intermediate cleanup steps discard errors. The most critical:
  merge state files and branch references may persist after a
  failed batch merge, requiring manual cleanup.

### Silent suppression (what errors are swallowed?)
- **Extensive `_ =` pattern across merge operations:** The refinery
  has 164 instances of `_ =` across 5 non-test files. Most are
  cleanup operations (removing temp files, closing connections)
  where failure is non-fatal. However, the volume means that
  silent failures in the merge pipeline may accumulate without
  any diagnostic trail. **Absent** — no aggregate logging of
  cleanup failures across a merge batch.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `sh` | `-c` | `<testCommand>` | rig config | `engineer.go:871` |
| `sh` | `-c` | `<gate.Cmd>` | rig config | `engineer.go:921` |
| `gt` | `nudge` | `mayor/ <nudgeMsg>` | runtime | `engineer.go:1210` |
| `gt` | `nudge` | `mayor/ <mayorMsg>` | runtime | `engineer.go:1258` |
| `gt` | `nudge` | `<target> <nudgeMsg>` | runtime | `engineer.go:1280` |
| `gt` | `nudge` | `mayor/ <mayorMsg>` | runtime | `engineer.go:1292` |
| `gt` | `nudge` | `deacon <nudgeMsg>` | runtime | `engineer.go:1851` |
| `bd` | `list` | `--type=convoy --status=open --json` | hardcoded | `engineer.go:1875` |
| `bd` | `dep list` | `<convoyID> --direction=down --type=tracks --json` | runtime | `engineer.go:1901` |
| `bd` | `show` | `<depID> --json` | runtime | `engineer.go:1932` |
| `bd` | `close` | `<convoyID> -r <reason>` | runtime | `engineer.go:1968` |
| `gt` | `mail send` | `<addr> -s <subj> -m <body>` | runtime | `engineer.go:1996` |
| `gt` | `swarm land` | `<moleculeID>` | runtime | `engineer.go:2035` |
| `gt` | `nudge` | `<target> <nudgeMsg>` | runtime | `manager.go:651` |

## Notes / open questions

- **Foreground mode is hard-rejected here.** `Manager.Start`
  refuses `foreground=true`. The `--foreground` flag on
  `gt refinery start` is still live in the CLI, but it hits this
  rejection path before doing anything useful. See
  [`gt refinery`](../commands/refinery.md) notes.
- **`mergeSlotSeq uint64` is a package-level counter** and therefore
  resets to zero on process restart. Combined with nanosecond
  timestamps, collisions are astronomically unlikely in practice,
  but the invariant depends on clocks moving forward.
- **30-minute default claim TTL is conservative** to avoid
  re-claiming MRs legitimately running long test suites, but it
  also means a crashed refinery worker's claim sits there for up
  to 30 minutes before another worker can pick it up.
- **The Engineer is 2000+ lines.** A future split along
  claim/gate/merge/convoy axes would reduce the blast radius of
  any single change; no such split is in progress today.
- **Post-squash gates reset on failure.** That reset path is not
  described in the shallow `runGatesForPhase` entry here — it's
  implicit in the merge pipeline. A drift check against
  `docs/design/merge-queue.md` (pending Phase 3) would be worth
  running.
- **`ValidPhaseTransitions` has no reverse transition out of
  `MRPhaseMerged` or `MRPhaseRejected`.** This is correct (they're
  terminal) but also means a merge that has to be rolled back
  must create a new MR, not reopen the old one.
