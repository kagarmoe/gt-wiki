---
title: internal/polecat
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [incomplete]
sources:
  - /home/kimberly/repos/gastown/internal/polecat/manager.go
  - /home/kimberly/repos/gastown/internal/polecat/session_manager.go
  - /home/kimberly/repos/gastown/internal/polecat/types.go
  - /home/kimberly/repos/gastown/internal/polecat/heartbeat.go
  - /home/kimberly/repos/gastown/internal/polecat/namepool.go
tags: [package, agent-runtime, polecat, worktree, persistent-identity, namepool, heartbeat, dolt-retry, witness, sling]
---

# internal/polecat

The largest agent-runtime package in Gas Town: persistent-identity
worker management built on git worktrees, themed name pools with
overflow, lifecycle states (working/idle/done/stuck/zombie), agent
beads with reset-on-reuse semantics, Dolt retry helpers,
session-heartbeat v2 with agent-reported state, polecat tmux
session lifecycle including respawn-pane and stale detection,
work-bead hooking and unassignment, ZFC-style worktree repair, and
stale-polecat detection by commit-distance from main.

**Go package path:** `github.com/steveyegge/gastown/internal/polecat`
**File count:** 5 non-test go files (`manager.go` is ~2400 lines,
`session_manager.go` is ~880 lines, `namepool.go` is ~770 lines,
`types.go` and `heartbeat.go` are small), 7 test files.
**Role:** [`polecat` role](../roles/polecat.md) — domain persona
**CLI command:** [`gt polecat`](../commands/polecat.md) (alias
`gt polecats`)
**Imports (notable):** `github.com/gofrs/flock`,
`github.com/google/uuid`, [`internal/beads`](beads.md),
[`internal/config`](config.md), `internal/constants`,
[`internal/doltserver`](go-packages.md),
[`internal/git`](go-packages.md), [`internal/lock`](lock.md),
[`internal/rig`](go-packages.md), `internal/runtime`,
[`internal/session`](session.md), `internal/style`,
[`internal/telemetry`](telemetry.md), `internal/templates`,
`internal/tmux`, [`internal/util`](util.md),
[`internal/workspace`](workspace.md).
**Imported by (notable):** the entire `gt polecat` CLI surface in
`/home/kimberly/repos/gastown/internal/cmd/polecat*.go`,
[`gt sling`](../commands/sling.md) and
[`gt convoy`](../commands/convoy.md) (which spawn polecats),
[`gt done`](../commands/done.md) (the polecat-side completion
primitive), the witness watchdog, and various rig-management
commands.

## What it actually does

Five files. The bulk is `manager.go`, which owns the polecat as a
persistent identity. `session_manager.go` is the tmux side.
`types.go` and `heartbeat.go` define the data shapes. `namepool.go`
is the themed name allocator.

### types.go — lifecycle states

Source: `/home/kimberly/repos/gastown/internal/polecat/types.go`.

The five-state lifecycle is the canonical declaration:

- `StateWorking` — session active, doing assigned work.
- `StateIdle` — work completed, session killed, sandbox preserved
  for reuse. The persistent-polecat model (gt-4ac).
- `StateDone` — `gt done` has been called; should be transient.
  A polecat stuck in StateDone is a **zombie**.
- `StateStuck` — agent self-reported stuck.
- `StateZombie` — detected condition: tmux session exists but no
  matching worktree directory. Set by callers, not stored long.

The doc comment on `State` (`types.go:6-26`) is explicit that
"stalled" and "zombie" are detected conditions, not stored states.
The Witness detects them through monitoring.

`Polecat` struct (`types.go:67-91`) — `Name`, `Rig`, `State`,
`ClonePath`, `Branch`, `Issue`, `CreatedAt`, `UpdatedAt`. JSON-
tagged.

`CleanupStatus` (`types.go:111-128`) — `clean`, `has_uncommitted`,
`has_stash`, `has_unpushed`, `unknown`. With predicates `IsSafe`,
`RequiresRecovery`, `CanForceRemove`. `CanForceRemove` excludes
stashes intentionally — stashes represent intentional WIP and
must not be silently nuked even with --force.

### heartbeat.go — session heartbeat v2

Source: `/home/kimberly/repos/gastown/internal/polecat/heartbeat.go`.

Heartbeat v1 was timestamp-only. Heartbeat v2 (gt-3vr5) adds
agent-reported state. The Witness makes exactly one inference: "is
the heartbeat fresh?" Everything else is what the agent reported.

- `HeartbeatState` constants `HeartbeatWorking`, `HeartbeatIdle`,
  `HeartbeatExiting`, `HeartbeatStuck` (`heartbeat.go:21-29`).
- `SessionHeartbeat` struct (`heartbeat.go:33-38`) — `Timestamp`,
  `State`, `Context`, `Bead`.
- `EffectiveState()` (`heartbeat.go:42-47`) — defaults to
  `HeartbeatWorking` for v1 heartbeats without a state field.
- `IsV2()` (`heartbeat.go:52-54`) — used by the Witness to choose
  between agent-reported state and legacy timer-based detection.
- `SessionHeartbeatStaleThreshold = 3 * time.Minute`
  (`heartbeat.go:13`).
- File layout: `<townRoot>/.runtime/heartbeats/<sessionName>.json`
  (`heartbeat.go:58-65`), parallel to `.runtime/pids/`.
- `TouchSessionHeartbeat(townRoot, sessionName)`
  (`heartbeat.go:71-73`) — convenience that writes
  `state="working"`.
- `TouchSessionHeartbeatWithState(townRoot, sessionName, state,
  context, bead)` (`heartbeat.go:78-97`) — used by `gt done`
  (`state="exiting"`) and `gt heartbeat` (`state="stuck"`).
  Best-effort: errors are silently ignored because heartbeats
  must never break a `gt` command.
- `ReadSessionHeartbeat`, `IsSessionHeartbeatStale`,
  `RemoveSessionHeartbeat` (`heartbeat.go:101-133`).

### namepool.go — themed name allocator

Source: `/home/kimberly/repos/gastown/internal/polecat/namepool.go`.

Every polecat gets a name from a themed pool. Built-in themes are
`mad-max`, `minerals`, `wasteland` (`namepool.go:45-82`) — each
50 names. The default theme is `mad-max`. Themes can be selected
per-rig via settings, and custom themes can be loaded from
`<townRoot>/themes/*.txt`.

- `DefaultPoolSize = 50`, `DefaultTheme = "mad-max"`,
  `MaxThemeNames = 2000` (`namepool.go:18-30`).
- `ReservedInfraAgentNames` (`namepool.go:35-42`) — `witness`,
  `mayor`, `deacon`, `refinery`, `crew`, `polecats`. These are
  filtered out of every theme; no polecat is ever named one of
  these.
- `BuiltinThemes` map (`namepool.go:45-82`).
- `NamePool` struct (`namepool.go:92-122`) — bound to a rig path
  and a state file at `<rig>/.runtime/namepool-state.json`.
  `InUse` is intentionally NOT persisted — it's discovered via
  `Reconcile` from the filesystem (ZFC).
- `NewNamePool(rigPath, rigName)` (`namepool.go:125-134`) and
  `NewNamePoolWithConfig` (`namepool.go:137-154`).
- `Allocate() (string, error)` (`namepool.go:269-292`) — picks
  the first not-in-use name from the resolved name list; on
  pool exhaustion, allocates an overflow name `<MaxSize+N>`.
- `Release(name)` (`namepool.go:294-304`) — frees the slot.
- `Reconcile(existingPolecats)` (`namepool.go:353-369`) —
  rewrites `InUse` from a list of currently-existing polecat
  names. The Manager calls this on construction to drop stale
  entries.
- Theme selection: `ThemeForRig(rigName)`,
  `ThemeForRigAvoiding(...)` (`namepool.go:431-484`) — auto-
  selects a theme based on rig name characteristics, with
  collision avoidance across rigs.
- Custom theme support: `ParseThemeFile`, `ResolveThemeNames`,
  `SaveCustomTheme`, `AppendToCustomTheme`, `DeleteCustomTheme`
  (`namepool.go:572-712`).
- `ValidatePoolName(name)` (`namepool.go:555-570`) — name
  validation rules.

### manager.go — persistent-identity polecat lifecycle

Source: `/home/kimberly/repos/gastown/internal/polecat/manager.go`.

The largest file in the package and one of the largest in the
`internal/` tree. Owns:

**Dolt retry helpers** (`manager.go:36-97`):

- `doltMaxRetries = 10`, `doltBaseBackoff = 500ms`,
  `doltBackoffMax = 30s` constants.
- `doltBackoff(attempt)` — exponential backoff with ±25% jitter,
  capped at `doltBackoffMax`.
- `isDoltOptimisticLockError(err)` — pattern-matches transient
  Dolt write conflicts that should be retried (optimistic lock,
  serialization failure, lock wait timeout, etc.).
- `isDoltConfigError(err)` — pattern-matches **non-transient**
  errors (not initialized, no such table, identity mismatch,
  Unknown database) that must NOT be retried because retrying
  wastes ~3 minutes on every spawn (gt-2ra: polecat spawn hang
  when Dolt DB not initialized).

**Sentinel errors** (`manager.go:100-108`):
`ErrPolecatExists`, `ErrPolecatNotFound`, `ErrHasChanges`,
`ErrHasUncommittedWork`, `ErrShellInWorktree`, `ErrDoltUnhealthy`,
`ErrDoltAtCapacity`. Plus `UncommittedWorkError` struct
(`manager.go:111-122`) wrapping a `git.UncommittedWorkStatus`.

**Manager type** (`manager.go:125-182`) — bound to a `*rig.Rig`,
`*git.Git`, `*beads.Beads`, `*NamePool`, and `*tmux.Tmux`.
Constructor `NewManager(r, g, t)` reads rig settings to pick a
theme, builds a name pool, and crucially uses
`beads.ResolveBeadsDir` to figure out where bd commands should
run (handles both local-beads and shared-beads modes).

**Key public methods** (selection — there are ~50 in total):

- `CheckDoltHealth()` (`manager.go:227-279`) and
  `CheckDoltServerCapacity()` (`manager.go:281-306`) — pre-flight
  checks before any Dolt-touching operation.
- `createAgentBeadWithRetry(agentID, fields)` and
  `SetAgentStateWithRetry(name, state)`
  (`manager.go:308-361`) — retry-wrapped writes.
- `Add(name) (*Polecat, error)` (`manager.go:624-630`) and
  `AllocateAndAdd(opts) (string, *Polecat, error)`
  (`manager.go:632-693`) — name allocation + worktree creation.
- `addWithOptionsLocked` / `AddWithOptions`
  (`manager.go:695-1024`) — the actual creation. ~330 lines of
  worktree fan-out, branch creation, runtime settings,
  shared-beads setup, mail dir creation, agent bead
  registration.
- `Remove(name, force)` and `RemoveWithOptions(name, force,
  nuclear, selfNuke)` (`manager.go:1026-1199`) — the canonical
  destruction path. Resets the agent bead BEFORE filesystem ops
  (per gt-14b8o, to avoid Dolt close/reopen cycles), unassigns
  any work beads still pointing at this polecat (gt-e4u1),
  refuses to nuke if the user's shell cwd is inside the
  worktree (unless `selfNuke=true`, which is set by the
  polecat's own `gt done` flow), removes the worktree via
  `repoGit.WorktreeRemove`, releases the name back to the pool.
- `RepairWorktree` / `RepairWorktreeWithOptions`
  (`manager.go:1332-1518`) — ZFC worktree repair.
- `ReuseIdlePolecat(name, opts)` (`manager.go:1520-1664`) — the
  persistent-polecat cycle: take an idle polecat, give it new
  work, transition to working without recreating the worktree.
- `ReconcilePool()` (`manager.go:1666-1731`) and
  `ReconcilePoolWith` (`manager.go:1732-1775`) — name pool
  reconciliation against filesystem state.
- `cleanupOrphanPolecatState()` (`manager.go:1826-1873`) —
  scrubs orphaned state files.
- `List()`, `FindIdlePolecat()`, `Get(name)`
  (`manager.go:1880-1945`) — read paths.
- `SetAgentState`, `SetState`, `AssignIssue`, `ClearIssue`
  (`manager.go:1947-2054`) — state transitions.
- `loadFromBeads(name)` (`manager.go:2100-2200`) — the
  beads-backed `Get` path; reconstructs polecat state from the
  agent bead and filesystem.
- `polecatSessionState(name)` (`manager.go:2202-2214`) — running
  + stale predicate.
- `setupSharedBeads(clonePath)` (`manager.go:2225-2249`) —
  delegates to `beads.SetupRedirect`.
- `CleanupStaleBranches()` (`manager.go:2251-2313`) — orphan
  branch GC.
- `DetectStalePolecats(threshold) ([]*StalenessInfo, error)`
  (`manager.go:2315-2387`) — finds polecats more than
  `threshold` commits behind the rig's default branch with no
  active session and no uncommitted work. Used by
  `gt polecat stale`.
- `assessStaleness` (`manager.go:2389-end`) — the staleness
  rules.

### session_manager.go — tmux session lifecycle for polecats

Source:
`/home/kimberly/repos/gastown/internal/polecat/session_manager.go`.

- `SessionManager` struct (`session_manager.go:43-47`) — bound
  to `*tmux.Tmux` and `*rig.Rig`.
- `SessionStartOptions` (`session_manager.go:57-79`) —
  `WorkDir`, `Issue`, `Command`, `Account`, `RuntimeConfigDir`,
  `Agent` (the per-session agent override; sets `GT_AGENT` in
  the tmux session env so `IsAgentAlive` reads the correct
  process names).
- `SessionInfo` (`session_manager.go:82-106`) — the JSON shape.
- `SessionName(polecat)` (`session_manager.go:111-122`) —
  delegates to `session.PolecatSessionName(prefix, polecat)`
  and validates against the double-prefix bug
  (`session_manager.go:127-145` — example bad name:
  `gt-gastown_manager-gastown_manager-142`). Logs but does not
  fail, so malformed sessions can still be tracked and cleaned.
- `polecatSlot(polecat)` (`session_manager.go:202-219`) — unique
  integer slot index for port offsetting and resource isolation
  when multiple polecats run in parallel (gh#954).
- `Start(polecat, opts)` (`session_manager.go:222-543`) — ~320
  lines. The polecat-side of session creation, including issue
  hooking, beacon construction, and `session.StartSession`
  invocation.
- `Stop(polecat, force)` (`session_manager.go:549-577`).
- `IsRunning`, `Status`, `List`, `ListPolecats`, `Attach`,
  `Capture`, `CaptureSession`, `Inject`, `StopAll`
  (`session_manager.go:578-769`) — the rest of the session
  surface.
- `verifyStartupNudgeDelivery` (`session_manager.go:819-872`) —
  liveness check for the initial nudge.
- `hookIssue(issueID, agentID, workDir)`
  (`session_manager.go:874-end`) — the bead hook that records
  "this polecat now owns this issue."
- `validateIssue(issueID, workDir)`
  (`session_manager.go:778-817`) — pre-flight check that the
  issue exists and isn't tombstoned.

### Notable design choices

- **Persistent identity, ephemeral session, ephemeral sandbox.**
  This is the fundamental polecat shape. The `Polecat` value
  outlives any single session. The agent bead (CV chain, work
  history) outlives the worktree. The worktree outlives the
  session. Only `Remove`/`Nuke` tears the whole thing down, and
  even then the agent bead is **reset for reuse** (not closed)
  so the same name can be re-added without colliding with
  Dolt's dead-row constraint (gt-14b8o).
- **Dolt error classification matters.** The retry loop
  distinguishes transient errors (worth retrying with backoff)
  from config errors (will fail identically every time). Without
  this split, a missing Dolt database would hang polecat spawn
  for ~3 minutes per attempt (gt-2ra).
- **ZFC-first state.** Wherever possible, the manager treats
  filesystem state as the source of truth and reconstructs
  polecat state from it (`loadFromBeads`, `ReconcilePool`,
  `cleanupOrphanPolecatState`). The name pool's `InUse` map is
  explicitly NOT persisted.
- **Reset-before-filesystem-ops in Remove.** The agent bead is
  reset BEFORE any filesystem operations because a concurrent
  sling could allocate the same name, set `hook_bead`, and then
  have it cleared by this cleanup. By resetting first,
  concurrent slings see a clean bead.
- **Shell-cwd safety check.** `RemoveWithOptions` refuses to
  nuke a polecat if the user's shell is `cd`'d into its
  worktree, unless `selfNuke=true` (gh#942). When a polecat
  calls `gt done`, it's inside its worktree by design; the
  session is going to die anyway. Otherwise the operator gets a
  clear error and a hint.
- **Heartbeat v2 is agent-honest.** The Witness only infers "is
  the heartbeat fresh?" Everything else — what state, what
  context, what bead — is what the agent reported. This is the
  ZFC principle applied to liveness.
- **Themed names with reserved-name filtering.** Reserved infra
  names (`witness`, `mayor`, `deacon`, etc.) are filtered out of
  every theme so a polecat is never named after a town-level
  agent, preventing routing confusion.
- **Best-effort heartbeat writes.** All heartbeat writes silently
  swallow errors. This is intentional: liveness signals must
  never block or fail a `gt` command.

## Related wiki pages

- [`polecat` role](../roles/polecat.md) — domain persona; what a
  polecat IS in Gas Town.
- [`gt polecat`](../commands/polecat.md) — CLI surface (16+
  subcommands across 5 sibling files).
- [`gt sling`](../commands/sling.md) — primary spawn entry
  point.
- [`gt done`](../commands/done.md) — polecat-only completion
  primitive.
- [`crew` package](crew.md) — the persistent counterpart;
  shared rig scope, very different semantics.
- [`internal/session`](session.md) — `PolecatSessionName`,
  `StartSession`, `BeaconConfig`, heartbeat substrate.
- [`internal/config`](config.md) — agent env, runtime config,
  startup command builders.
- [`internal/beads`](beads.md) — agent beads,
  `ResolveBeadsDir`, `ResetAgentBeadForReuse`,
  `CreateOrReopenAgentBead`, `SetupRedirect`.
- [`internal/doltserver`](go-packages.md) — health check used
  by `CheckDoltHealth`.
- [`internal/lock`](lock.md) — used by name pool save.
- [`internal/telemetry`](telemetry.md) —
  `RecordPolecatRemove` and friends.
- [`mayor` package](mayor.md) — the cleanup-veto checker that
  protects polecat workspaces while a Mayor is reviewing diffs
  in ACP mode.
- [`deacon` package](deacon.md) — the redispatch and stale-hook
  patrols that operate on polecat work beads.
- [`gt convoy`](../commands/convoy.md) — work batching upstream
  of sling.
- [`gt witness`](../commands/witness.md) — per-rig watchdog
  that runs the polecat lifecycle observer.

## Notes / open questions

- `manager.go` at ~2400 lines is the largest file in the
  agent-runtime layer. A future split by concern (Dolt retries,
  worktree management, name pool integration, beads
  reconciliation, staleness detection) would help readability
  without changing semantics.
- The Dolt retry constants are compiled-in but configurable via
  `operational.polecat.*` settings. The `LoadOperationalConfig`
  hook is in the deacon package; polecat doesn't currently read
  it for Dolt retry tuning, so the constants here are
  effectively the only knobs.
- `polecatSlot` is a positional integer derived from directory
  enumeration order. If the filesystem returns entries in a
  different order across runs (different filesystems do this),
  the slot can change for the same polecat. This is fine for
  resource isolation but worth noting if anyone keys a cache
  by it.
- `validateSessionName` only logs the double-prefix bug, it
  doesn't refuse the session creation. This is deliberate
  (so malformed sessions can be cleaned up) but means the
  bug can persist until something explicitly reaps it.
- The persistent-polecat model (`ReuseIdlePolecat`) shares the
  same name and worktree across many work assignments. This
  means a polecat's `CreatedAt` doesn't tell you "when did this
  agent start its current task" — for that you need the agent
  bead's hook history.
- Themed name pools have a tasteful default (`mad-max`) but
  the per-rig theme selection means rigs with different themes
  produce visually distinct polecat names. If a town has many
  rigs, the theme namespace can collide; `ThemeForRigAvoiding`
  is the avoidance hook.
