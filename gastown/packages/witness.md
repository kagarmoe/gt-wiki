---
title: internal/witness
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
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
sources:
  - /home/kimberly/repos/gastown/internal/witness/manager.go
  - /home/kimberly/repos/gastown/internal/witness/handlers.go
  - /home/kimberly/repos/gastown/internal/witness/protocol.go
  - /home/kimberly/repos/gastown/internal/witness/mountain.go
  - /home/kimberly/repos/gastown/internal/witness/patrol_receipts.go
  - /home/kimberly/repos/gastown/internal/witness/spawn_count.go
  - /home/kimberly/repos/gastown/internal/witness/dedup.go
tags: [package, agent-runtime, witness, polecat-monitor, per-rig, patrol, protocol, mountain-eater]
---

# internal/witness

Polecat-monitoring runtime for the Witness — Gas Town's per-rig
health watcher. The package owns the Witness tmux session lifecycle,
the protocol-message vocabulary the Witness uses to hear about
polecat events, the zombie/stalled/orphan detection engines, the
Mountain-Eater convoy failure-tracker, the spawn-count circuit
breaker that prevents respawn storms, the patrol-receipt
serialization, and the message deduplicator that keeps restart-time
re-reads idempotent.

**Go package path:** `github.com/steveyegge/gastown/internal/witness`
**File count:** 7 non-test `.go` files — `manager.go`, `handlers.go`
(the bulk: 2705 lines), `protocol.go`, `mountain.go`,
`patrol_receipts.go`, `spawn_count.go`, `dedup.go`.
**Role:** [`witness` role](../roles/witness.md) — domain persona
**CLI command:** [`gt witness`](../commands/witness.md)
**Imports (notable):** [`internal/beads`](beads.md),
[`internal/channelevents`](channelevents.md),
[`internal/config`](config.md), `internal/constants`,
`internal/git`, [`internal/lock`](lock.md),
[`internal/mail`](mail.md), [`internal/mayor`](mayor.md),
[`internal/nudge`](nudge.md), [`internal/polecat`](polecat.md),
`internal/rig`, [`internal/runtime`](../packages/go-packages.md),
[`internal/session`](session.md), [`internal/style`](style.md),
`internal/tmux`, [`internal/util`](util.md),
[`internal/workspace`](workspace.md).
**Imported by (notable):** [`gt witness`](../commands/witness.md)
(`/home/kimberly/repos/gastown/internal/cmd/witness.go`), the
`mol-witness-patrol` molecule (indirectly, via `gt polecat`
subcommands), and any daemon path that needs the `PatrolVerdict` /
`PatrolReceipt` shapes for reporting.

## What it actually does

Most of this package is called by the `mol-witness-patrol` molecule
rather than by an always-on Go loop. The Witness agent (Claude in
a tmux session) is what drives the detection passes; the Go code
here is the library that molecule calls into.

### Manager — `manager.go`

- `NewManager(r *rig.Rig) *Manager` (`manager.go:39-43`) — bound to
  the rig.
- `Manager.IsRunning() (bool, error)` (`manager.go:50-54`) — wraps
  `tmux.CheckSessionHealth`. Returns true only if
  `tmux.SessionHealthy` — zombie sessions are NOT reported running.
- `Manager.IsHealthy(maxInactivity) tmux.ZombieStatus`
  (`manager.go:61-64`) — the stricter variant that also flags hung
  sessions.
- `Manager.SessionName() string` (`manager.go:67-69`) — returns
  `session.WitnessSessionName(session.PrefixFor(m.rig.Name))`.
- `Manager.witnessDir() string` (`manager.go:90-102`) — prefers
  `<rig>/witness/rig`, falls back to `<rig>/witness`, then `<rig>/`.
- `Manager.Start(foreground, agentOverride, envOverrides) error`
  (`manager.go:110-278`) — the 170-line start path. Refuses
  foreground (deprecated — patrol logic moved to
  `mol-witness-patrol`). Detects zombie sessions and mitigates the
  TOCTOU race: on seeing a dead agent, record the session's
  creation time, wait `constants.ZombieKillGracePeriod`, then
  re-verify before killing so a slow-starting agent isn't
  destroyed (`manager.go:127-147`). Resolves role config, loads
  `runtime.EnsureSettingsForRole`, builds the startup command via
  `buildWitnessStartCommand` (handles built-in Claude start
  commands, non-Claude agents, and custom TOML patterns with
  template expansion), generates a UUID run ID, creates the
  session with `NewSessionWithCommand`, writes `AgentEnv` env vars
  + role-config env (skipping keys already set to preserve the
  qualified `GT_ROLE` per gh#2492) + CLI overrides, applies theme,
  waits for Claude, accepts startup dialogs, starts the nudge
  poller, tracks session PID, and records GASTA instantiation.
- `Manager.Stop() error` (`manager.go:369-381`) — kills the tmux
  session. Returns `ErrNotRunning` if none.
- `Manager.roleConfig() (*beads.RoleConfig, error)`
  (`manager.go:280-293`) — pulls the role definition from
  `config.LoadRoleDefinition` and projects it into the shape the
  start path needs.
- `buildWitnessStartCommand(rigPath, rigName, townRoot,
  sessionName, agentOverride, roleConfig)`
  (`manager.go:314-357`) — resolves whether to use the built-in
  Claude command, a non-Claude agent (which forces the standard
  config path), or a custom TOML `start_command` with template
  expansion. Handles the `env -u CLAUDECODE NODE_OPTIONS=''`
  wrapping for Claude invocations.
- `isBuiltinClaudeStartCommand(cmd) bool` (`manager.go:362-365`) —
  true for `exec claude --dangerously-skip-permissions`, the stock
  TOML default.
- Sentinel errors: `ErrNotRunning`, `ErrAlreadyRunning`
  (`manager.go:28-30`).

### Protocol — `protocol.go`

The vocabulary for Witness inbox routing. Protocol messages are
special-shaped lines the Witness receives via mail or bead
descriptions — the Witness matches them against regex patterns
and dispatches to a handler.

- Pattern regexes (`protocol.go:14-50`): `POLECAT_DONE`,
  `LIFECYCLE:Shutdown`, `HELP:`, `MERGED`, `MERGE_FAILED`,
  `MERGE_READY`, `HANDOFF`, `SWARM_START`, `DISPATCH_ATTEMPT`,
  `DISPATCH_OK`, `DISPATCH_FAIL`, `IDLE_PASSIVATED`.
- `ProtocolType` enum (`protocol.go:52-69`) with matching constants.
- `AgentState` alias for `beads.AgentState` (`protocol.go:74-85`) —
  re-exports: `AgentStateRunning`, `AgentStateIdle`,
  `AgentStateDone`, `AgentStateStuck`, `AgentStateEscalated`,
  `AgentStateSpawning`, `AgentStateWorking`, `AgentStateNuked`.
  The canonical definitions live in `internal/beads`.
- `ExitType` enum (`protocol.go:91-98`) — `COMPLETED`,
  `ESCALATED`, `DEFERRED`, `PHASE_COMPLETE`. Stored on agent
  beads' `exit_type` field so the Witness can discover completions
  from beads (not just from `POLECAT_DONE` mail).
- `PolecatDonePayload`, `SwarmStartPayload`, classification helpers,
  and additional parser functions (`protocol.go:100+`).

### Handlers — `handlers.go`

The bulk of the Witness logic. Each protocol handler returns a
`HandlerResult` (`handlers.go:93-104`) with `MessageID`,
`ProtocolType`, `Handled`, `Action`, `CleanupStatus`, `WispCreated`,
`MailSent`. Most handlers are driven from the `mol-witness-patrol`
molecule via `gt polecat` subcommands.

Notable entry points (line numbers from ripgrep):

- `HandlePolecatDone(bd, workDir, rigName, msg, router)`
  (`handlers.go:118+`) — the primary completion handler. Branches
  on whether the polecat had a pending MR: if so,
  `handlePolecatDonePendingMR` notifies the Refinery via
  `notifyRefineryMergeReady`; otherwise
  `handlePolecatDoneNoMR` just records the result.
- `HandlePolecatDoneFromBead(...)` (`handlers.go:182+`) — the
  bead-driven variant discovered during a sweep rather than from a
  POLECAT_DONE mail.
- `TransitionPolecatToIdle(workDir, agentBeadID)`
  (`handlers.go:257+`) — state transition helper.
- `HandleLifecycleShutdown` (`handlers.go:335+`),
  `HandleHelp` (`handlers.go:361+`), `HandleMerged`
  (`handlers.go:387+`), `HandleMergeFailed` (`handlers.go:441+`),
  `HandleSwarmStart` (`handlers.go:473+`) — one per protocol type.
- `createCleanupWisp`, `createSwarmWisp` (`handlers.go:501-567`)
  — spawn a wisp bead (ephemeral issue) to track cleanup work.
  See [`wisp` concept](../concepts/wisp.md).
- `findCleanupWisp`, `findAnyCleanupWisp`, `findAllCleanupWisps`
  (`handlers.go:569+, 2609+, 2633+`) — lookup variants.
- `EscalateRecoveryNeeded(workDir, rigName, payload)`
  (`handlers.go:753+`) — uses
  [`internal/mayor`](mayor.md) to route a recovery escalation
  upward.
- `UpdateCleanupWispState(bd, workDir, wispID, newState)`
  (`handlers.go:766+`) — drives the cleanup wisp's state machine.
- `NukePolecat(bd, workDir, rigName, polecatName)`
  (`handlers.go:830+`) — the hard cleanup primitive the Witness
  invokes via `gt polecat nuke`.
- `AutoNukeIfClean(workDir, rigName, polecatName)`
  (`handlers.go:889+`) — safer variant that checks commit
  verification on main before nuking via `_verifyCommitOnMain`
  (`handlers.go:911+`).

#### Zombie detection

- `ZombieClassification` (`handlers.go:976-998`) — typed strings
  classifying why a zombie is a zombie.
- `ZombieClassification.ImpliesActiveWork() bool`
  (`handlers.go:999+`) — projects classification to a "was doing
  real work" boolean.
- `ZombieResult` (`handlers.go:1010+`) — per-polecat detection
  outcome with `PolecatName`, `AgentState`, `Classification`,
  `HookBead`, `BeadRecovered`, `WasActive`, `Action`, `Error`.
- `DetectZombiePolecatsResult` (`handlers.go:1023+`) — the sweep
  result with a `Zombies` slice and a `ConvoyFailures` slice.
- `DetectZombiePolecats(bd, workDir, rigName, router)`
  (`handlers.go:1059+`) — the primary detection engine. Examines
  each polecat in the rig, distinguishes live-session zombies
  (`detectZombieLiveSession`, `handlers.go:1162+`) from dead-session
  zombies (`detectZombieDeadSession`, `handlers.go:1282+`), and
  produces a classified list. Then hands off to
  `trackConvoyFailures` (see `mountain.go`) to record the failure
  against any tracking convoy.
- `isZombieState`, `handleZombieRestart`
  (`handlers.go:1412+, 1431+`) — the restart logic for zombies
  with recoverable hook beads, gated by the spawn-count circuit
  breaker (see `spawn_count.go`).

#### Stalled polecat detection

- `SpawnGracePeriod = 5 * time.Minute` (`handlers.go:1525`).
- `StalledResult`, `DetectStalledPolecatsResult`
  (`handlers.go:1528+, 1538+`) — parallel shapes to zombie
  detection for a different failure mode.
- `DetectStalledPolecats(workDir, rigName)` (`handlers.go:1558+`)
  — detects polecats that are alive-but-stuck, not crashed.

#### Completion discovery

- `CompletionDiscovery`, `DiscoverCompletionsResult`
  (`handlers.go:1661+, 1677+`).
- `DiscoverCompletions(bd, workDir, rigName, router)`
  (`handlers.go:1700+`) — sweeps beads for polecats whose
  `exit_type` field signals completion even though the
  POLECAT_DONE mail was missed or dropped. The "bead is the source
  of truth" fallback.

#### Orphan detection

- `OrphanedBeadResult`, `DetectOrphanedBeadsResult`
  (`handlers.go:2159+, 2167+`) — beads stuck in `hooked` state
  with no live polecat.
- `DetectOrphanedBeads(bd, workDir, rigName, router)`
  (`handlers.go:2179+`) — the sweep that resets them via
  `resetAbandonedBead`.
- `OrphanedMoleculeResult`, `DetectOrphanedMoleculesResult`
  (`handlers.go:2287+, 2298+`) — molecules whose children are all
  closed but the parent is still open.
- `DetectOrphanedMolecules(bd, workDir, rigName, router)`
  (`handlers.go:2316+`) — closes parent + descendants via
  `closeMoleculeWithDescendants` (`handlers.go:2463+`).

#### Utilities

- `DoneIntent`, `extractDoneIntent(labels)` (`handlers.go:2542+`)
  — parses `done:...` labels into structured intent.
- `sessionRecreated(t, sessionName, detectedAt)`
  (`handlers.go:2591+`) — TOCTOU helper for the "did the session
  we saw as dead actually come back?" check.
- `hasPendingMR(bd, workDir, _, polecatName, agentBeadID)`
  (`handlers.go:2666+`).

### Mountain-Eater — `mountain.go`

Failure tracking for convoys with the `mountain` label. When a
polecat exits without completing its hooked bead, this module
checks whether the issue is in a mountain convoy, increments a
failure counter label, and auto-skips the issue after
`MountainMaxFailures = 3` attempts.

- `MountainMaxFailures = 3` (`mountain.go:22`).
- `ConvoyFailureResult` (`mountain.go:25-33`) — per-issue result.
- `trackConvoyFailures(bd, workDir, result)` (`mountain.go:39-74`)
  — called from `DetectZombiePolecats`. Iterates zombies that had
  a hook bead but didn't complete (`ZombieBeadClosedStillRunning`
  is excluded — that's a completed bead, not a failure), calls
  `TrackConvoyFailure` per bead, logs, and appends to
  `result.ConvoyFailures`.
- `TrackConvoyFailure(bd, workDir, hookBead)` (`mountain.go:80+`)
  — the per-issue workhorse. Returns nil if the issue has no
  tracking convoy.

### Spawn-count circuit breaker — `spawn_count.go`

The Witness's primary defense against respawn storms ("clown show
#22"): after `MaxBeadRespawns` attempts on the same bead, the
Witness stops re-dispatching and escalates to the Mayor.

- `beadRespawnRecord`, `beadRespawnState` (`spawn_count.go:22-32`)
  — the on-disk shape at
  `<townRoot>/witness/bead-respawn-counts.json`.
- `loadBeadRespawnState`, `saveBeadRespawnState`
  (`spawn_count.go:38-64`) — file I/O with 0600 mode.
- `ShouldBlockRespawn(workDir, beadID) bool`
  (`spawn_count.go:71-93`) — the circuit-breaker check. Reads
  `MaxBeadRespawnsV` from operational config, acquires an
  in-process mutex + a cross-process flock, loads state, and
  returns true if the count is at/over the limit.
- `RecordBeadRespawn(workDir, beadID) int`
  (`spawn_count.go:102-127`) — increments and returns the new
  count. Save errors are non-fatal — tracking failure must not
  block the respawn.
- `ResetBeadRespawnCount(workDir, beadID)`
  (`spawn_count.go:131-149`) — used by `gt sling respawn-reset`
  after operator investigation.

Serialization: `respawnMu` (in-process sync.Mutex) plus
`lock.FlockAcquire` on a sibling `.flock` file (cross-process).
Both are needed because multiple Witness instances may race.

### Patrol receipts — `patrol_receipts.go`

Machine-readable projection of zombie detection results for
daemon consumers.

- `PatrolVerdict` enum (`patrol_receipts.go:6-11`) — `stale` or
  `orphan`.
- `PatrolReceiptEvidence` struct (`patrol_receipts.go:14-20`) —
  evidence fields including the typed `ZombieClassification`.
- `PatrolReceipt` struct (`patrol_receipts.go:23-29`) — the
  wire shape.
- `receiptVerdictForZombie(z) PatrolVerdict`
  (`patrol_receipts.go:34-46`) — projects zombie result to verdict
  via classification (gt-tsut), with a `WasActive` fallback for
  forward-compat.
- `BuildPatrolReceipt(rigName, z)` (`patrol_receipts.go:49-72`).
- `BuildPatrolReceipts(rigName, result)` (`patrol_receipts.go:75-85`).

### Deduplication — `dedup.go`

- `MessageDeduplicator` (`dedup.go:11-14`) — thread-safe
  in-memory set of processed message IDs.
- `NewMessageDeduplicator()` (`dedup.go:19-23`).
- `MessageDeduplicator.AlreadyProcessed(messageID) bool`
  (`dedup.go:28-42`) — atomic check-and-set. Prevents
  double-processing across patrol goroutines within one Witness
  session. Empty IDs are allowed (cannot be deduped).
- `Size() int` (`dedup.go:44-48`).

The comment at `dedup.go:17-18` captures the failure mode:
restart-time re-reads could create duplicate cleanup wisps — this
keeps them idempotent WITHIN a session. Cross-session idempotency
is the caller's responsibility.

### Notable design choices

- **ZFC throughout.** No tmux state file; queue state derived from
  tmux session health. The spawn-count state is a genuine file on
  disk because the count must survive restarts — but it's not a
  source of truth for running state, just for "have we tried this
  already".
- **The Witness is a `mol-witness-patrol` driver.** The Go code
  here is a library for that molecule; the tmux session is the
  Claude (or compatible agent) that runs the molecule. The
  vestigial `--foreground` flag on
  [`gt witness`](../commands/witness.md) exists because an earlier
  design had a Go-side patrol loop here — it was removed in favour
  of the molecule-driven approach.
- **Two detection engines for one problem.** Zombie detection
  (`DetectZombiePolecats`) handles crashed/dead polecats; stalled
  detection (`DetectStalledPolecats`) handles alive-but-stuck ones.
  Both feed through the same recovery machinery but they look for
  different symptoms.
- **Bead is the source of truth for completion.**
  `DiscoverCompletions` sweeps beads for `exit_type` fields set by
  `gt done` rather than relying on POLECAT_DONE mail, so mail loss
  doesn't leave stale polecats.
- **Spawn-count circuit breaker with flock.** Cross-process
  serialisation on the count state file means a Witness restart
  mid-respawn does not reset the count — the escalation threshold
  is honoured even under instability.
- **Mountain-Eater is opt-in per convoy.** Without the `mountain`
  label on the convoy, failures just log a warning; with it, the
  per-issue counter accumulates and auto-skip fires at 3. See
  `docs/design/convoy/mountain-eater.md` section 5.

## Related wiki pages

- [`witness` role](../roles/witness.md) — domain persona.
- [`gt witness`](../commands/witness.md) — CLI surface.
- [`wisp` concept](../concepts/wisp.md) — cleanup wisps are the
  primary artifact the Witness creates.
- [`convoy` concept](../concepts/convoy.md) — what the Mountain-
  Eater protects.
- [`polecat` package](polecat.md) — subject of observation.
- [`polecat` role](../roles/polecat.md) — the domain persona the
  Witness watches over.
- [`refinery` package](refinery.md) — sibling per-rig agent.
- [`mayor` package](mayor.md) — escalation target for
  unrecoverable situations.
- [`deacon` package](deacon.md) — "The Deacon monitors all
  Witnesses."
- [`internal/beads`](beads.md) — `AgentState`, `AgentFields`,
  `RoleConfig`.
- [`internal/mail`](mail.md), [`internal/nudge`](nudge.md) —
  protocol delivery.
- [`internal/lock`](lock.md) — flock for spawn-count state.
- [`internal/config`](config.md) — operational config for
  `MaxBeadRespawnsV`, threshold tuning.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Session lifecycle cleanup:** Like other agent runtime packages,
  this package uses `_ =` for tmux session teardown, lock release,
  and file cleanup operations in error paths. These are typically
  cleanup-on-failure code where the primary error has already been
  captured. **Present** — most suppressions are in defer/cleanup
  contexts where POSIX semantics provide a safety net.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `gt` | `mail send` | `mayor/ -s <subject> -m <body>` | runtime | `handlers.go:733` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| spawn count state | JSON data | Spawn count persistence | `spawn_count.go:63` |

## Notes / open questions

- **`handlers.go` is 2700+ lines** and is by far the largest file
  in Layer (f). A split along detection-axis (zombie / stalled /
  orphan / mountain) would make the package more navigable.
- **The `WispCreated` / `MailSent` fields on `HandlerResult`** are
  inconsistent in naming — one describes a bead that now exists,
  the other describes an action that was taken (and is also
  marked deprecated in favour of nudge). A future cleanup could
  rename to `Action` verbs.
- **Three protocol-message sources exist simultaneously.** Mail
  (POLECAT_DONE etc.), bead fields (`exit_type`), and patrol
  sweeps (DetectZombiePolecats). When these disagree, the
  Witness's behaviour depends on which sweep it's running at the
  moment.
- **The Mountain-Eater auto-skip threshold is hard-coded** at 3.
  A per-convoy override would be natural but not yet implemented.
- **`buildWitnessStartCommand` has four code paths** for resolving
  the start command (built-in Claude / non-Claude / custom TOML /
  default). A simplification is plausible but the current code
  has explicit historical reasons (gh#2492).
- **The patrol receipt type is designed for machine consumers**
  but no daemon path currently emits them in a persistent way.
  A future drift check against the daemon patrol ingest would
  confirm whether they're actually read.
