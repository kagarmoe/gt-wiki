---
title: polecat-lifecycle (workflow)
type: workflow
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/internal/cmd/sling.go
  - /home/kimberly/repos/gastown/internal/polecat/manager.go
  - /home/kimberly/repos/gastown/internal/polecat/session_manager.go
  - /home/kimberly/repos/gastown/internal/polecat/types.go
  - /home/kimberly/repos/gastown/internal/polecat/heartbeat.go
  - /home/kimberly/repos/gastown/internal/cmd/done.go
  - /home/kimberly/repos/gastown/internal/witness/handlers.go
tags: [workflow, polecat, lifecycle, sling, done, witness, refinery, nuke, persistent-identity, ephemeral-session]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# polecat-lifecycle

The end-to-end workflow for a single polecat assignment:
from `gt sling` dispatching a bead to a polecat, through
name allocation, worktree creation, session spawn, hook,
work, merge-request submission, `gt done`, Witness
observation, Refinery merge, cleanup wisp, and the polecat
transitioning back to idle (persistent-identity model) or
being nuked entirely. This is the canonical "happy path"
for how a bead becomes merged code via a polecat.

The workflow spans three files primarily:
`internal/cmd/sling.go` (dispatch entry point),
`internal/polecat/manager.go` + `session_manager.go`
(allocation, worktree, session), and
`internal/cmd/done.go` + `internal/witness/handlers.go`
(completion, cleanup).

Polecats in Gas Town have **persistent identity and
ephemeral sessions**: the polecat's agent bead, themed
name, and sandbox worktree survive across assignments;
only the tmux session and work-bead hook are
per-assignment. This workflow therefore has two
termination shapes: **go idle** (persistent model,
default) and **nuke** (full teardown when requested).

## Trigger

Any of:

- **`gt sling <bead> <rig>`** — the normal entry point.
  Source: `/home/kimberly/repos/gastown/internal/cmd/sling.go:196`
  (`runSling`).
- **`gt sling <bead> <rig>/polecats/<name>`** — explicit
  polecat targeting (usually for `--force` reassignment).
- **`gt convoy launch` / feed** — convoy dispatch slings
  multiple beads to per-rig polecats; each dispatch goes
  through the same sling path.
- **`gt formula run`** — a formula-based dispatch pours a
  molecule and slings each leg to a polecat.
- **Batch sling** — `gt sling <bead1> <bead2> ... <rig>`
  (`sling.go:313-341`) dispatches multiple beads in one
  command, each to its own polecat.

In all cases, the actual polecat spawn happens inside
`resolveTarget` (`sling.go:666`), which may allocate a
new polecat, reuse an idle one, or route the bead to an
existing working polecat depending on rig state and CLI
flags.

## Actors involved

- **Operator** or upstream agent (Mayor, Deacon) —
  triggers `gt sling`.
- **[`internal/polecat` package](../packages/polecat.md)** —
  name pool, manager, session manager.
- **[polecat role](../roles/polecat.md)** — the agent
  identity that does the work.
- **[`internal/rig` package](../packages/rig.md)** —
  provides the rig context
  (`polecat.NewManager(r, g, t)`).
- **[witness role](../roles/witness.md) + [package](../packages/witness.md)**
  — watches the polecat, detects stalls, handles
  `POLECAT_DONE` events, creates cleanup wisps.
- **[refinery role](../roles/refinery.md) + [package](../packages/refinery.md)**
  — consumes merge requests from the polecat, runs
  gates, squash-merges, closes the source bead.
- **[`internal/beads`](../packages/beads.md)** — agent
  bead CRUD, work bead hooking.
- **[mayor role](../roles/mayor.md)** — notified of
  polecat completion so it can sling the next bead
  (slot-open notification).
- **[convoy package](../packages/convoy.md)** (only for
  convoy-fed work) — the close-hook that dispatches the
  next convoy issue.

## Prerequisites

Before this workflow can start:

- The rig must exist (`gt rig add` or `gt rig quick-add`
  has been run — see
  [`gt rig`](../commands/rig.md)).
- The Dolt server must be reachable. Polecat spawn
  pre-flight checks are in
  `polecat/manager.go:227-306` (`CheckDoltHealth`,
  `CheckDoltServerCapacity`).
- The bead being slung must exist and not be blocked or
  already hooked to another polecat (unless `--force`).
- If a new polecat is being allocated, the rig's name
  pool must have a slot available
  (`polecat/namepool.go:269-292`, `Allocate`).

## States

A polecat moves through the following states. Each state
corresponds to a `polecat.State` value declared in
`/home/kimberly/repos/gastown/internal/polecat/types.go:6-26`
plus file:line refs for the transition points.

### State 0: (not yet existing) → name allocation

**Source:** `sling.go:runSling` →
`resolveTarget(sling.go:666)` →
`polecat.AllocateAndAdd`
(`polecat/manager.go:632-693`).

When a sling requests dispatch to a rig and no existing
polecat is available:

1. The sling path calls `resolveTarget`
   (`sling.go:666-677`) with the target rig. Sling checks
   the early guard at `sling.go:216-223` — **polecats
   cannot sling** (`GT_ROLE=polecat` blocks `runSling`
   with "polecats cannot sling (use gt done for handoff)").
2. `resolveTarget` decides whether to reuse an idle
   polecat, start fresh, or route to an existing worker.
3. For fresh allocation: `Manager.AllocateAndAdd(opts)`
   (`polecat/manager.go:632-693`) calls
   `NamePool.Allocate()` (`namepool.go:269-292`) to pick
   the first available themed name (e.g., `furiosa`,
   `obsidian`, based on the rig's theme — see
   `BuiltinThemes` at `namepool.go:45-82`).
4. If the name pool is exhausted, an overflow name is
   synthesized (`<MaxSize+N>`).

The polecat does not yet have a worktree, agent bead, or
tmux session. Just a name.

### State 1: name allocated → worktree + agent bead created

**Source:** `Manager.addWithOptionsLocked`
(`polecat/manager.go:695-1024`, ~330 lines).

The heavyweight creation path:

1. **Polecat directory** — `<rig>/polecats/<name>/` is
   created.
2. **Worktree creation** — `git worktree add` branches off
   a new branch from the rig's default branch. Polecats
   operate in worktrees, not clones, so they share the
   `.repo.git/` bare repo with the Refinery and can push
   branches visible to it without round-tripping the
   remote.
3. **Runtime settings** — the polecat's `.claude/` /
   `.opencode/` settings are scaffolded (`hooks.InstallForRole`).
4. **Beads redirect** — if the rig has a
   redirect-backed `.beads/`, the polecat's worktree gets
   the same redirect via `setupSharedBeads`
   (`polecat/manager.go:2225-2249`).
5. **Mail dir** — the polecat's inbox directory is
   created under town beads mail storage.
6. **Agent bead registration** —
   `createAgentBeadWithRetry` (`manager.go:308-326`)
   calls `beads.CreateOrReopenAgentBead` with
   `RoleType=polecat`, `Rig=<rig>`, `AgentState=idle`,
   `HookBead=""`. The CV chain for this polecat starts
   accumulating here. `CreateOrReopen` handles the
   "re-spawning with the same name" case (GH#332) —
   the same themed name can be reused after a nuke.
7. **Name pool persist** — the name pool state file is
   updated to reflect the allocation
   (`namepool.go` via `saveState`).

At this point the polecat exists as: a directory, a
worktree branch, an agent bead (state = idle), and a
slot in the name pool. Still no tmux session, no work
assigned.

### State 2: worktree ready → session spawned + issue hooked

**Source:** `SessionManager.Start(polecat, opts)`
(`polecat/session_manager.go:222-543`, ~320 lines).

The sling path invokes `SessionManager.Start` after the
polecat has been allocated. Session startup does:

1. **Session name** — `SessionName(polecat)`
   (`session_manager.go:111-122`) builds the tmux session
   name via `session.PolecatSessionName(prefix, polecat)`.
   Validation at `session_manager.go:127-145` checks for
   the double-prefix bug.
2. **Issue validation** — `validateIssue(issueID, workDir)`
   (`session_manager.go:778-817`) confirms the bead
   exists and isn't tombstoned. This is a pre-flight
   against the sling: if the bead is invalid, no session
   gets started.
3. **Issue hooking** — `hookIssue(issueID, agentID, workDir)`
   (`session_manager.go:874+`) writes the "this polecat
   now owns this issue" relationship: the bead's
   `assignee` field is set, and the polecat's agent bead
   gets `HookBead = issueID`. This is the authoritative
   "I am working on X" record.
4. **Beacon config** — constructs the agent beacon (env
   vars, startup command) that tells the agent how to
   prime itself.
5. **`GT_*` env vars set** — `GT_ROLE=polecat`,
   `GT_RIG=<rig>`, `GT_POLECAT=<name>`, `GT_ROOT=<town>`,
   `BEADS_ACTOR=<rig>/polecats/<name>`. See
   [identity concept](../concepts/identity.md).
6. **`session.StartSession`** — actual tmux session
   spawn happens via the `internal/session` package,
   which creates the pane, sets the cwd to the polecat's
   worktree, and launches the agent binary with the
   prime command.
7. **Nudge delivery verification** —
   `verifyStartupNudgeDelivery`
   (`session_manager.go:819-872`) confirms the initial
   nudge to the session was received, so the agent
   actually started priming.
8. **Agent state → working** — the agent bead's
   `AgentState` is updated to `working` once prime
   completes successfully.

The polecat is now in `StateWorking` (`types.go:9-10`):
tmux session active, worktree mounted, issue hooked,
agent primed. The agent runs its loop.

### State 3: working → build, commit, push

**Source:** whatever the agent does in its session.
From the workflow's perspective, the polecat:

- Reads the hooked bead's description via `bd show` or
  `gt current`.
- Edits files in its worktree.
- Commits changes on its polecat-specific branch (the
  branch was created when the worktree was added).
- Touches its heartbeat periodically via
  `TouchSessionHeartbeat` / `TouchSessionHeartbeatWithState`
  (`polecat/heartbeat.go:71-97`). The heartbeat file at
  `<townRoot>/.runtime/heartbeats/<sessionName>.json`
  records `{timestamp, state, context, bead}`. Heartbeat
  v2 (gt-3vr5) lets the agent self-report `working`,
  `idle`, `exiting`, `stuck`.
- Pushes the branch to the shared bare repo (the
  Refinery can see it because all worktrees share
  `.repo.git`).

During this phase the **Witness** is observing: reading
heartbeats, checking session state, watching for stalls.
`SessionHeartbeatStaleThreshold = 3 * time.Minute`
(`heartbeat.go:13`) is the Witness's freshness budget.
If heartbeats stop coming, the Witness may intervene.
See [witness role](../roles/witness.md).

If the agent realizes it cannot proceed, it calls
`gt heartbeat --state stuck` which writes a heartbeat
with `state="stuck"`. The Witness escalates.

### State 4: work done → `gt done` called

**Source:** `internal/cmd/done.go:runDone` (`:91+`).

When the polecat has committed and pushed its work, it
calls `gt done`. This command is polecat-only — the
early guard at `done.go:91-99` refuses to run for any
other role:

> `gt done is for polecats only (you are <actor>)`

The done flow:

1. **Actor check** (`done.go:91-99`) — refuses non-polecats.
2. **Cwd normalization** (`done.go:181`) — allow running
   from a subdirectory of the worktree.
3. **Commit guard** (`done.go:275-313`) — refuses to
   proceed if there are uncommitted changes with a
   clear error message. "The agent should have committed
   before running gt done."
4. **Checkpoint recovery** (`done.go:369-378`) — if a
   previous `gt done` was interrupted, resume from the
   checkpoint.
5. **Heartbeat → exiting** (`done.go:386-387`) —
   `TouchSessionHeartbeatWithState(townRoot, sessionName,
   HeartbeatExiting, "gt done", issueID)`. This tells
   the Witness the polecat is in the `gt done` flow,
   which means "trust the agent until it's actually
   done" — don't treat heartbeat pauses as stalls.
6. **Merge request submission** — builds a merge-request
   wisp in the beads database with the polecat's branch,
   target branch, hook bead, and agent metadata. The
   wisp lands in the refinery's queue.
7. **Agent state → done** (transient) — the polecat's
   agent bead transitions to `StateDone` (`types.go:11-12`).
   This is explicitly a **transient** state; a polecat
   stuck in `StateDone` for a long time is a **zombie**
   (`types.go:13-14`).
8. **Persistent polecat transition** — with the
   persistent polecat model (gt-hdf8, comment at
   `done.go:108`), sessions stay alive after `gt done`:
   the polecat transitions to `StateIdle` instead of
   being killed. The polecat's sandbox worktree is
   preserved for reuse; the session is torn down but the
   polecat remains allocated.
9. **Send completion signal** — writes a `POLECAT_DONE`
   event to the witness's channel and a mail message.

### State 5: Witness receives `POLECAT_DONE`

**Source:** `internal/witness/handlers.go:handlePolecatDonePendingMR`
(`:264-282`) and
`handlePolecatDoneNoMR` (`:307-314`).

The Witness has two paths depending on whether a merge
request was created:

**With a pending MR** (`handlers.go:264-282`):

1. **Create cleanup wisp** — `createCleanupWisp(bd,
   workDir, polecatName, issueID, branch)`
   (`handlers.go:500+`). The cleanup wisp is an
   ephemeral bead tracking "this polecat has work pending
   refinery review; clean up after merge." See
   [wisp concept](../concepts/wisp.md).
2. **Update wisp state** —
   `UpdateCleanupWispState(bd, workDir, wispID,
   "merge-requested")`.
3. **Notify Refinery** —
   `notifyRefineryMergeReady(workDir, rigName, result)`
   (`handlers.go:289-303`). Emits a `MERGE_READY`
   channel event so the refinery's await-event loop
   unblocks instantly, and a tmux nudge as belt-and-
   suspenders.
4. **Action:** deferred cleanup; polecat stays around
   until the MR merges.

**With no pending MR** (`handlers.go:307-314`):

1. **No nuke** — the persistent polecat model (gt-4ac,
   comment at `handlers.go:308-310`): polecats go idle
   after completion, sandbox is preserved. The polecat
   already set its own state to `idle` in `gt done`.
2. **Acknowledge** — action string: "polecat
   <name> completed (exit=<exit>, no MR) — now idle,
   sandbox preserved".

In both branches, **Mayor is notified that a slot is
open** (`handlers.go:245-247`,
`notifyMayorSlotOpen(workDir, rigName, polecatName, ...)`),
so the Mayor can sling the next bead to this polecat.

### State 6: Refinery merges the MR

**Source:** `internal/refinery/engineer.go` — the
refinery's normal merge pipeline.

When the `MERGE_READY` event fires (or the refinery's
next patrol cycle runs):

1. Refinery picks up the oldest unclaimed MR wisp from
   the queue.
2. Runs gates (tests, lint, custom checks per the rig's
   `merge_queue` config).
3. Squash-merges the polecat's branch into the default
   branch.
4. Closes the MR wisp with `reason="merged"`.
5. Force-closes the original hook bead with "Merged in
   <MR-ID>".
6. Fires close-event hooks, which among other things
   invoke [`convoy.CheckConvoysForIssue`](../packages/convoy.md)
   if the closed bead was convoy-tracked (see the
   [convoy-launch workflow](convoy-launch.md)).

The polecat's work is now on the default branch.

### State 7: Cleanup → idle OR nuke

**Source:** both `polecat/manager.go:Remove*` and the
witness's post-merge cleanup path.

**Default (persistent polecat model, gt-4ac):**

- The polecat's cleanup wisp is closed (`state
  = "cleaned"`).
- The polecat's agent bead stays in `StateIdle`; worktree
  is preserved.
- Name pool slot remains occupied.
- **Ready for reuse** via `ReuseIdlePolecat`
  (`manager.go:1520-1664`) on the next sling.

**Explicit nuke path** (when requested via
`gt polecat nuke` or when witness determines cleanup
is needed):

- `Manager.RemoveWithOptions(name, force, nuclear,
  selfNuke)` (`manager.go:1026-1199`):
  - **Reset agent bead FIRST** (`manager.go:1102-1105`
    comment): a concurrent sling could try to allocate
    the same name — by resetting first, the concurrent
    sling sees a clean bead and
    `CreateOrReopenAgentBead` handles the re-spawn case.
  - **Unassign work beads** (gt-e4u1): any bead still
    pointing at this polecat gets unhooked.
  - **Shell-cwd safety check** (`manager.go:1085-1088`
    comment; gh#942): refuse to nuke if the operator's
    shell is `cd`'d into the worktree, unless
    `selfNuke=true`. When a polecat calls `gt done`
    it's inside its worktree by design — `selfNuke=true`
    handles that case.
  - **Worktree removal** — `repoGit.WorktreeRemove`.
  - **Name pool release** — `NamePool.Release(name)`
    (`namepool.go:294-304`).

At the end of a full nuke, the polecat slot is empty and
the name is available for reuse. The agent bead is
**reset, not closed** — the same name can come back into
service without Dolt's dead-row constraint getting in
the way.

### State 8 (recovery only): zombie detection

**Source:** witness patrol, polecat staleness scan.

If a polecat gets stuck in `StateDone` (the transient
state) long enough to count as a zombie, or if its tmux
session exists but the worktree does not
(`StateZombie`, `types.go:13-14`), the Witness detects
and recovers:

- Zombie session but no worktree → kill session, release
  name.
- Stale commit distance from main (`DetectStalePolecats`,
  `manager.go:2315-2387`) → mark stale, possibly nuke.
- Heartbeat stopped → Witness intervenes via its patrol
  loop (see [`internal/witness`](../packages/witness.md)).

These are detected conditions, not stored states — the
Witness's single inference (`is the heartbeat fresh?`)
drives most recovery paths.

## Failure modes

- **Dolt unavailable at spawn time.** `CheckDoltHealth`
  (`manager.go:227-279`) and `CheckDoltServerCapacity`
  (`manager.go:281-306`) fail fast before any expensive
  operation. Returns `ErrDoltUnhealthy` /
  `ErrDoltAtCapacity`.
- **Transient Dolt errors during spawn.** The retry
  helper (`manager.go:36-97`) distinguishes transient
  errors (backoff + retry) from config errors
  (`isDoltConfigError`) that will fail identically every
  time. Without this split, a missing Dolt database
  would hang spawn for ~3 minutes per attempt (gt-2ra).
- **Name pool exhausted.** `NamePool.Allocate()` falls
  back to an overflow name. The rig can still spawn, but
  the names become non-themed.
- **Bead already hooked.** Sling refuses unless
  `--force`, which sends a shutdown message to the old
  polecat (`sling.go:710-745`), waits for cleanup, and
  re-assigns.
- **Uncommitted work in `gt done`.** Refused with a
  clear error (`done.go:275-313`): "The agent should
  have committed before running gt done."
- **Mid-flight session crash.** The Witness detects a
  stale heartbeat (`SessionHeartbeatStaleThreshold = 3
  minutes`, `heartbeat.go:13`) and triggers recovery.
  If the polecat was in `HeartbeatExiting` state
  (i.e., inside `gt done`), the Witness trusts the
  agent longer to avoid racing its own cleanup.
- **`gt done` crash after checkpoint.** The next `gt
  done` reads the checkpoint file
  (`done.go:369-378`) and resumes from where it left
  off.
- **MR gate failure.** The refinery's gates reject the
  MR; the wisp gets an error state. The polecat stays
  idle; the operator must intervene.
- **Cross-rig sling.** `checkCrossRigGuard`
  (`sling.go:697-701`) rejects slings where the bead's
  rig prefix doesn't match the target polecat's rig,
  unless `--force` or it's a self-sling. Polecats can't
  fix code owned by another rig.
- **Shell-cwd in worktree during nuke.** Refused unless
  `selfNuke=true`. Operator gets a clear error pointing
  at the offending shell.
- **Stalled polecat.** `DetectStalePolecats`
  (`manager.go:2315-2387`) walks every polecat and
  measures commit distance from main plus active
  session plus uncommitted work. Stale polecats can be
  cleaned up with `gt polecat stale`.

## Relationships

This workflow is the atomic "do one bead" unit. Larger
workflows compose many polecat lifecycles:

- **[convoy-launch workflow](convoy-launch.md)** —
  launches a convoy that slings many beads, each going
  through its own polecat lifecycle. The convoy's feed
  loop triggers the next sling after each merge.
- **[molecule execution](../concepts/molecule.md)** — a
  polecat can have an attached molecule on its hook
  bead, in which case the work the polecat does is
  molecule-step-driven rather than single-bead.
- **[wisp lifecycle](../concepts/wisp.md)** — merge
  requests, cleanup wisps, molecule step wisps, and
  rate-limit notices created during a polecat's life are
  all wisps subject to TTL and promotion.

## Related wiki pages

- [polecat role](../roles/polecat.md) — what a polecat
  IS in Gas Town.
- [`internal/polecat` package](../packages/polecat.md) —
  the implementation.
- [`gt sling`](../commands/sling.md) — the dispatch
  entry point.
- [`gt done`](../commands/done.md) — the
  polecat-only completion command.
- [`gt polecat`](../commands/polecat.md) — the polecat
  management CLI (list, nuke, stale, repair, etc.).
- [witness role](../roles/witness.md) — the watchdog.
- [`internal/witness` package](../packages/witness.md) —
  `handlePolecatDonePendingMR`, `handlePolecatDoneNoMR`,
  `createCleanupWisp`.
- [refinery role](../roles/refinery.md) — consumes the
  merge-request wisps.
- [`internal/refinery` package](../packages/refinery.md)
  — engineer and manager.
- [mayor role](../roles/mayor.md) — receives slot-open
  notifications after polecat completion.
- [convoy-launch workflow](convoy-launch.md) — the
  larger workflow that this lifecycle is a step inside.
- [wisp concept](../concepts/wisp.md) — merge requests
  and cleanup wisps.
- [molecule concept](../concepts/molecule.md) — polecats
  often host molecules.
- [identity concept](../concepts/identity.md) —
  `GT_ROLE=polecat`, `GT_RIG`, `GT_POLECAT` are the
  spawn-time env vars.
- [rig concept](../concepts/rig.md) — polecats live
  inside a rig.

## Notes / open questions

- **The persistent polecat model changes the lifecycle's
  default end state.** Pre-gt-4ac, `gt done` led to an
  immediate session kill + worktree removal. Post-gt-4ac,
  the default is `StateIdle` with worktree preserved.
  Nuke is now an explicit action, not the automatic end
  of every lifecycle.
- **`StateDone` is transient and has no durable
  representation.** A polecat observed in `StateDone`
  for any meaningful duration is a zombie. The
  heartbeat-exiting state is the "we're in `gt done`
  right now, please trust us" signal.
- **Cleanup wisps are the audit channel.** The
  Witness's `createCleanupWisp` writes an ephemeral
  bead tracking the cleanup lifecycle, which the
  reaper eventually compacts (unless someone commented
  on it, which triggers `HasComments` promotion — see
  [wisp concept](../concepts/wisp.md)).
- **`validateSessionName` only warns on the
  double-prefix bug.** `session_manager.go:127-145`
  logs but does not reject; malformed sessions are
  still tracked so they can be cleaned up. Worth a
  doctor check.
- **`polecatSlot` is positional by directory order.**
  If the filesystem returns entries in a different
  order across runs (different filesystems do this),
  a polecat's port-offset slot can change. Not a
  correctness problem but worth noting if any cache
  keys by it.
- **`ReuseIdlePolecat` is the branch-only fast path.**
  When an idle polecat exists and a new bead is slung
  to the same rig, sling prefers reuse: create a new
  branch in the existing worktree, reset the agent
  bead to `working` with a new hook, and start a new
  session. No worktree recreation, no name pool
  churn. See `manager.go:1520-1664`.
- **Mail and channel events both exist for
  polecat completion.** The mail is the
  survive-session-death channel; the channel event is
  the instant-wakeup channel. Both are sent so the
  Witness and Refinery hear about completion whether
  they're in patrol loop, await-event, or mid-shell.
- **The workflow does not address `--ralph` loop mode.**
  Ralph mode (acceptance-criteria-driven loops) is a
  per-step execution shape; it runs inside State 3
  rather than changing the overall lifecycle. A
  separate workflow page may be warranted.
- **No explicit "stuck" → recovery path is enumerated
  here.** `gt heartbeat --state stuck` transitions the
  polecat to a stuck state; the Witness escalates; the
  operator or Deacon intervenes. The recovery path is
  outside this workflow's scope.
