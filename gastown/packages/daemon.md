---
title: internal/daemon
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/daemon/daemon.go
  - /home/kimberly/repos/gastown/internal/daemon/types.go
  - /home/kimberly/repos/gastown/internal/daemon/pidfile.go
  - /home/kimberly/repos/gastown/internal/daemon/lifecycle.go
  - /home/kimberly/repos/gastown/internal/daemon/lifecycle_defaults.go
  - /home/kimberly/repos/gastown/internal/daemon/handler.go
  - /home/kimberly/repos/gastown/internal/daemon/pressure.go
  - /home/kimberly/repos/gastown/internal/daemon/log_rotation.go
  - /home/kimberly/repos/gastown/internal/daemon/restart_tracker.go
  - /home/kimberly/repos/gastown/internal/daemon/dolt.go
  - /home/kimberly/repos/gastown/internal/daemon/compactor_dog.go
  - /home/kimberly/repos/gastown/internal/daemon/convoy_manager.go
  - /home/kimberly/repos/gastown/internal/daemon/jsonl_git_backup.go
  - /home/kimberly/repos/gastown/internal/daemon/signals_unix.go
tags: [package, daemon, long-running, patrol, supervisor, heartbeat]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [incomplete]
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression, partial-completion]
detail_depth: {params: 1, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/daemon

Gas Town's per-town background process — a "dumb scheduler" Go binary
that runs as a single locked singleton, supervises long-running
infrastructure (Dolt server, convoy manager, feed curator, KRC pruner),
fires a recovery-focused heartbeat loop that restarts dead agents, and
dispatches a zoo of independent patrol tickers that do periodic
maintenance work (compaction, backups, log rotation, checkpoints,
zombie detection, scheduled work dispatch). The daemon is *the*
long-running process in a town: every other agent is either a Claude
tmux session supervised by the daemon, or a one-shot CLI invocation.

**Go package path:** `github.com/steveyegge/gastown/internal/daemon`
**File count:** 33 non-test Go files, 29 test files — one of the
largest packages in the codebase. `daemon.go` alone is ~100KB.
**CLI surface:** [`gt daemon`](../commands/daemon.md) (8 subcommands).
**Driven by:** [`gt up`](../commands/up.md), [`gt start`](../commands/start.md),
[`gt down`](../commands/down.md) / [`gt shutdown`](../commands/shutdown.md).
**Drives:** [Deacon](../roles/deacon.md) patrol cycles, Mayor/Witness/
Refinery restarts, Dolt server lifecycle, convoy delivery, a dozen
independent maintenance tickers.

## Design stance

From the package doc comment at `daemon.go:1-9`:

> The daemon is a simple Go process (not a Claude agent) that:
> 1. Pokes agents periodically (heartbeat)
> 2. Processes lifecycle requests (cycle, restart, shutdown)
> 3. Restarts sessions when agents request cycling
>
> The daemon is a "dumb scheduler" — all intelligence is in agents.

And from `daemon.go:48-51`:

> The daemon is the town-level background service. It ensures patrol
> agents (Deacon, Witnesses) are running and detects failures. This
> is recovery-focused: normal wake is handled by feed subscription
> (`bd activity --follow`). The daemon is the safety net for dead
> sessions, GUPP violations, and orphaned work.

This framing matters: the daemon is deliberately dumb. It does not
decide *what work to do* — that's the job of Claude agents (Deacon
handles judgment, Mayor handles escalation). The daemon's job is to
make sure those agents *exist and are running*, and to own everything
that can't be trusted to a potentially-dead Claude process: Dolt
uptime, convoy delivery, periodic filesystem backups, log rotation,
exponential backoff on crash loops.

## Process model

**Single-process singleton per town root.** The daemon is a single OS
process (no child workers, no pool) that runs `gt daemon run` as its
argv. It holds an exclusive `gofrs/flock` on
`<townRoot>/daemon/daemon.lock` for its entire lifetime (`daemon.go:341-352`).
Startup races are resolved by `flock.TryLock()` — only one daemon per
town can ever be running.

**Identity scrub.** When launched from `gt daemon start`'s child
re-exec, `gt daemon run` (in `internal/cmd/daemon.go`) clears all
`agentconfig.IdentityEnvVars` before calling `daemon.New`, so the
daemon doesn't impersonate whoever launched it. It then runs as
`BD_ACTOR=daemon` for the life of the process. See
[`gt daemon`](../commands/daemon.md) for the CLI side.

**Goroutines inside the single process:**

- Main loop (`Daemon.Run`, `daemon.go:322-707`) — select-based dispatcher.
- Feed curator goroutine (`feed.Curator`, `daemon.go:394-400`).
- Convoy manager (`ConvoyManager`, `daemon.go:424-428`) — event-driven
  delivery loop plus periodic stranded-convoy scan.
- KRC pruner (`KRCPruner`, `daemon.go:442-452`) — ephemeral data
  cleanup.
- Dolt server supervisor goroutine (embedded inside
  [`DoltServerManager`](doltserver.md)).
- Telemetry (OpenTelemetry) exporters when enabled.

All other work happens inline on the main loop goroutine via ticker
channels. Several fields on the `Daemon` struct are marked
"Only accessed from heartbeat loop goroutine — no sync needed"
(`daemon.go:78-118`), explicitly relying on single-threaded main-loop
discipline instead of mutexes.

## Entry point

`Daemon.Run` at `daemon.go:322-707` is the single entry point, invoked
from `cmd/gt/internal/cmd/daemon.go`'s hidden `gt daemon run`
subcommand via `daemon.New(daemon.DefaultConfig(townRoot))` + `d.Run()`.

`Run()` does the following on startup, in order:

1. **Acquire `daemon.lock` flock** (`daemon.go:341-352`). Non-blocking;
   if already held, error out.
2. **Pre-flight: `checkAllRigsDolt`** (`daemon.go:354-357`) — every
   rig must be on Dolt backend; startup blocks if any uses SQLite.
3. **Metadata repair** (`daemon.go:361-365`) via
   `doltserver.EnsureAllMetadata`.
4. **Write PID file** with random nonce (`daemon.go:368-370`,
   `pidfile.go:20-30`) — format is `"PID\nNONCE"`, used for ownership
   verification without `ps` string matching.
5. **Save initial state** (`daemon.go:374-381`) to
   `<townRoot>/daemon/state.json`.
6. **Install signal handlers** (`daemon.go:384-385`) for SIGINT,
   SIGTERM, SIGUSR1, SIGUSR2.
7. **Start feed curator** (`daemon.go:394-400`) for
   `bd activity --follow`-based agent wakes.
8. **Open beads stores and start convoy manager** (`daemon.go:405-428`).
9. **Wire Dolt recovery callback** (`daemon.go:434-440`) — when Dolt
   transitions unhealthy → healthy, the convoy manager runs a sweep.
10. **Start KRC pruner** (`daemon.go:442-452`).
11. **Create all the maintenance tickers** (`daemon.go:456-589`) — one
    `time.Ticker` per enabled patrol.
12. **Run initial heartbeat** (`daemon.go:596`).
13. **Enter the select loop** (`daemon.go:599-706`).

## The heartbeat loop

`Daemon.heartbeat` at `daemon.go:724-879` runs once per
`recoveryHeartbeatInterval` (default **3 minutes**, configurable via
`operational.daemon.recovery_heartbeat_interval`, see
`daemon.go:713-715`). Important: the heartbeat interval changed from the
older 5-minute `Config.HeartbeatInterval` default (`types.go:41`) —
that field is now effectively legacy; the real interval comes from
[`operational.daemon.recovery_heartbeat_interval`](config.md).

A single heartbeat performs, in order:

- **Shutdown guard** (`daemon.go:728-731`): if
  `isShutdownInProgress()` returns true, skip the entire heartbeat.
  This prevents the daemon from fighting `gt down` by immediately
  restarting agents it's trying to kill.
- **E-stop guard** (`daemon.go:737-740`): if
  `estop.IsActive(townRoot)` is true, skip agent management entirely
  (but keep the daemon alive to maintain Dolt).
- **Prefix registry reload** (`daemon.go:748-750`) — so newly-added
  rigs get correct session names.
- **Kill ghost sessions** (`daemon.go:753`) — stale `gt-*` prefixed
  sessions from prior registry state.
- **Ensure Dolt server running** (`daemon.go:757`).
- **Ensure Deacon running** — if `deacon` patrol active
  (`daemon.go:761-769`), else kill leftover deacon/boot sessions.
- **Poke Boot** (`daemon.go:774-776`) for triage.
- **Check Deacon heartbeat file** (`daemon.go:781-783`) directly —
  belt-and-suspenders fallback when Boot misses a stuck state.
- **Ensure Witnesses running** for all rigs (`daemon.go:787-793`).
- **Ensure Refineries running**, pressure-gated (`daemon.go:798-808`).
- **Ensure Mayor running** (`daemon.go:811`).
- **Handle dogs** — cleanup stuck, reap idle, dispatch plugins,
  pressure-gated (`daemon.go:815-825`, see `handler.go`).
- **Process lifecycle requests** (`daemon.go:828`) — read the deacon
  mail inbox for `LIFECYCLE:` messages (see `lifecycle.go`).
- **GUPP violations** — agents stuck on a hook (`daemon.go:833`).
- **Orphaned work** — beads assigned to dead agents (`daemon.go:836`).
- **Polecat session health check** (`daemon.go:840`).
- **Reap idle polecats** (`daemon.go:845`) — kill tmux sessions after
  `gt done` to free API slots.
- **Cleanup orphaned claude subagents** (`daemon.go:850`).
- **Prune stale tracking branches** (`daemon.go:856`).
- **Dispatch scheduled work** via `gt scheduler run`,
  pressure-gated (`daemon.go:861-865`).
- **Rotate oversized Dolt logs** (`daemon.go:869`) using copytruncate.
- **Bump state** (`LastHeartbeat`, `HeartbeatCount`) and save.

The heartbeat is deliberately idempotent and pressure-gated: any step
can be skipped if the system is under load, and every "ensure" step is
a check-then-restart pattern that does nothing if the agent is already
healthy.

## The maintenance tickers

Alongside the heartbeat timer, `Daemon.Run` creates ~10 independent
`time.Ticker`s for maintenance work that runs on its own cadence
independent of the 3-minute heartbeat. Each ticker is gated on
`isPatrolActive(name)` (combining `mayor/daemon.json` with the
town-level `settings/config.json::disabled_patrols` list —
`types.go:381-386`) and has its own handler file:

| Ticker | File | Default interval | Purpose |
|---|---|---|---|
| `doltHealth` | `dolt.go` | 30s | Fast Dolt crash detection |
| `doltRemotes` | `dolt_remotes.go` | 15m | Push Dolt DBs to git remotes |
| `doltBackup` | `dolt_backup.go` | 15m | Filesystem backup sync |
| `jsonlGitBackup` | `jsonl_git_backup.go` | 15m | Scrub + export + git push |
| `wispReaper` | `wisp_reaper.go` | 30m | Close/delete stale wisps |
| `doctorDog` | `doctor_dog.go` | 5m | Dolt health monitor |
| `compactorDog` | `compactor_dog.go` | 24h | Flatten commit history |
| `checkpointDog` | `checkpoint_dog.go` | 10m | Auto-commit WIP polecat work |
| `scheduledMaintenance` | `scheduled_maintenance.go` | varies | `gt maintain --force` in window |
| `mainBranchTest` | `main_branch_test_runner.go` | 30m | Quality gates per rig |
| `quotaDog` | `quota_dog.go` | varies | Credential rotation on rate-limit |

Every ticker case in the select loop is guarded with
`if !d.isShutdownInProgress()` (see `daemon.go:623-698`) so that an
in-progress `gt down` halts all maintenance work, not just the
heartbeat.

Defaults for the enabled set live in
`lifecycle_defaults.go:DefaultLifecycleConfig` at
`lifecycle_defaults.go:15-66`. `EnsureLifecycleConfigFile` at
`lifecycle_defaults.go:134-146` creates or patches the config file to
ensure lifecycle patrols exist — this is called both by `gt up` and
during `daemon.New` startup (`daemon.go:189-191`).

## State management

The daemon owns a suite of state files under `<townRoot>/daemon/`:

| File | Owner | Purpose |
|---|---|---|
| `daemon.lock` | `Run` (`daemon.go:341`) | flock singleton marker. Never deleted — lock is on the fd, not the path. |
| `daemon.pid` | `writePIDFile` (`pidfile.go:20-30`) | `"PID\nNONCE"` format; nonce prevents PID-reuse confusion. |
| `daemon.log` | `lumberjack.Logger` (`daemon.go:152-160`) | 100MB max, 3 backups, 7 days, compressed. Lumberjack auto-rotates. |
| `state.json` | `SaveState` (`types.go:90-99`) | `State{Running, PID, StartedAt, LastHeartbeat, HeartbeatCount}`. Atomic write. |
| `shutdown.lock` | `gt down` / `gt shutdown` | flock held by shutdown commands. `isShutdownInProgress` checks this. File is **never removed** (`daemon.go:1972-1976`) because flock semantics require the inode to persist. |
| `restart_state.json` | `RestartTracker.Load/Save` (`restart_tracker.go`) | Per-agent crash-loop counters and next-allowed-restart timestamps. |

Additional state owned on disk but typically under sibling directories:

- **`mayor/daemon.json`** — `DaemonPatrolConfig` (patrol enable/disable,
  intervals, the `Env` map propagated to spawned sessions). Managed by
  `LoadPatrolConfig` / `SavePatrolConfig` at `types.go:216-242`. See
  [`gt config`](../commands/config.md) and [`internal/config`](config.md).
- **`settings/config.json::disabled_patrols`** — town-level
  patrol disable list loaded by
  `loadDisabledPatrolsFromTownSettings` (`types.go:358-375`). Simpler
  than editing `mayor/daemon.json` for one-off disables.

**State file referenced elsewhere but NOT owned here:** the Deacon's
`heartbeat.json`, `paused.json`, `feed-stranded-state.json`,
`redispatch-state.json`, and `health-check-state.json` all live under
`<townRoot>/deacon/` and are owned by [`internal/deacon`](deacon.md),
not this package. The daemon only *reads* Deacon's heartbeat file to
decide whether the Deacon is alive. See the
[Deacon role](../roles/deacon.md) page for that inventory.

## Lifecycle: start, stop, crash recovery

**`daemon.New`** (`daemon.go:143-319`): allocates the `Daemon` struct
with `lumberjack` log writer, initializes tmux global env
(`GT_TOWN_ROOT`, and explicitly *unsets* identity vars per GH#3006),
loads patrol config + applies lifecycle defaults, boots
`DoltServerManager` if configured, propagates `GT_DOLT_PORT` /
`BEADS_DOLT_PORT` to the process env so spawned agent sessions
inherit the right Dolt connection info (GH#2412), resolves `gt`/`bd`
paths, initializes `RestartTracker`, and best-effort initializes
OpenTelemetry (`GT_OTEL_METRICS_URL`, `GT_OTEL_LOGS_URL`).

**`Daemon.Run`** — described above. The defer stack tears things
down on exit: release flock, remove PID file, save final `state.json`
via `shutdown()`.

**`Daemon.shutdown`** (`daemon.go:1911-1961`) runs on context cancel
(`cancel()` from signal handler or from `Stop()`):

- Stops feed curator.
- Stops convoy manager (closes beads stores).
- Stops KRC pruner.
- **Pushes Dolt remotes before stopping the server** — one last best-
  effort push so local-only commits aren't stranded.
- Stops Dolt server (only if we manage it — skipped in "external" mode).
- Flushes OpenTelemetry with 5s deadline.
- Marks `state.Running=false` and saves final state.

**`daemon.IsRunning`** (`daemon.go:2028-2067`) is the public API for
other packages (e.g. `gt up`, `gt down`, `gt daemon status`) to check
if a daemon is running. It uses `daemon.lock` flock as the
authoritative signal rather than `ps` command-line matching, falling
back to PID-file verification only when flock check fails. This is
the ZFC-mandated fix for gt-utuk.

**`daemon.StopDaemon`** (`daemon.go:2097-2140`) resolves the PID from
the lock file, sends a term signal, waits
`constants.ShutdownNotifyDelay`, and force-kills if still alive.
Best-effort removes the PID file on exit.

**`daemon.FindOrphanedDaemons`** / **`KillOrphanedDaemons`**
(`daemon.go:2149-2203`) detect the edge case where `daemon.lock` is
held but the PID file is stale — rare, since flock should release on
process death, but present as a safety net.

### The `shutdown.lock` sentinel (GH#2656)

`gt down` writes a flock on `<townRoot>/daemon/shutdown.lock` before
terminating any agents. The daemon's heartbeat and every ticker case
check `isShutdownInProgress` (`daemon.go:1968-2002`) and bail out
immediately if the lock is held by someone else. Without this, the
daemon would happily restart whichever Deacon/Witness/Mayor `gt down`
just killed — an endless tug-of-war during teardown.

Two flavors exist: `d.isShutdownInProgress()` (instance method) and
the exported `daemon.IsShutdownInProgress(townRoot)` (`daemon.go:2007-2026`)
used by Boot and other packages that need the same check without a
live `Daemon` struct.

**The `shutdown.lock` file is never deleted**, intentionally
(`daemon.go:1972-1976`). `flock` works on file descriptors, not paths;
removing the file while another process waits on the flock defeats
mutual exclusion.

## Supervision

"Supervision" in this package comes in two distinct flavors.

### 1. Agent supervision (built-in)

The daemon's heartbeat loop *is* a supervisor for Claude agents. Each
`ensureXxxRunning` function follows the same pattern: check if the
tmux session is alive, if not, use the `internal/<role>` package's
`Manager.Start()` method to spawn a fresh one, subject to
`RestartTracker` exponential-backoff gating.

**`RestartTracker`** at `restart_tracker.go:73-78` persists per-agent
restart timestamps with exponential backoff
(`restart_tracker.go:14-45` for defaults: 30s initial, 10m max, 2.0x,
15m crash-loop window, 5 crashes before crash-loop state, 30m
stability period). `ClearAgentBackoff` at `restart_tracker.go:258` is
the on-disk mutator that `gt daemon clear-backoff` calls; a
subsequent SIGUSR2 tells a running daemon to reload the state from
disk (`daemon.go:610-617`, `signals_unix.go:23-25`).

### 2. OS-level supervision (opt-in via `gt daemon enable-supervisor`)

`gt daemon enable-supervisor` provisions a **launchd plist on macOS or
a user systemd unit on Linux** that invokes `gt daemon run` at login /
boot. The templates live in a sibling package; this daemon package
has no OS-level supervision code itself. The distinction matters: the
daemon does not self-respawn. If the daemon process dies, nothing in
this package brings it back — the OS supervisor is responsible for
that, or the user runs `gt up`/`gt daemon start` manually.

## Signals

`signals_unix.go:10-25` defines the handled set:

- **SIGINT, SIGTERM** → graceful shutdown via `d.shutdown(state)`.
- **SIGUSR1** → lifecycle signal (`isLifecycleSignal`). Immediately
  runs `processLifecycleRequests` without waiting for the next
  heartbeat. Sent by `gt handoff` so cycle/restart/shutdown requests
  don't sit in the inbox for up to 3 minutes.
- **SIGUSR2** → reload-restart signal. Reloads `RestartTracker` from
  disk. Sent by `gt daemon clear-backoff` after it's edited the
  on-disk crash-loop state.

Windows has its own stub (`signals_windows.go`) with matching but
minimal signal handling.

## Lifecycle requests

`lifecycle.go:ProcessLifecycleRequests` (`lifecycle.go:41-104`) is the
inbox processor for agent-initiated lifecycle actions. Agents that
want to cycle, restart, or shut themselves down send a `gt mail` to
the `deacon/` inbox with subject `LIFECYCLE:` and a structured JSON
body `{"action": "cycle" | "restart" | "shutdown"}`. Each heartbeat
(or SIGUSR1) the daemon reads this inbox, parses requests, **deletes
the message before executing** (claim-then-execute), then performs
the action.

Stale requests are filtered by
`operational.daemon.max_lifecycle_message_age` (default 6h,
`lifecycle.go:36-38`). This prevents a mail written during a prior
outage from re-triggering a restart days later.

## Pressure gating

`pressure.go:checkPressure` (`pressure.go:43-91`) is a three-tier gate
used by heartbeat steps that spawn API-consuming agents (refineries,
dogs, polecats):

1. **CPU pressure** — 1-minute load average / `NumCPU` vs
   `operational.daemon.pressure_cpu_threshold`.
2. **Memory pressure** — `availableMemoryGB` vs
   `pressure_mem_threshold`.
3. **Session concurrency** — count of agent-prefixed tmux sessions vs
   `pressure_max_sessions`.

**Infrastructure agents are deliberately exempt**
(`pressure.go:38-43`): Deacon, Witness, and Mayor are monitoring /
recovery and must start even when the system is loaded. Only
refinery, dog, and polecat spawns are gated.

Platform-specific memory pressure queries live in
`pressure_darwin.go`, `pressure_linux.go`, and fallback
`pressure_other.go` / `pressure_windows.go`.

## Log rotation

**Two independent systems:**

1. **`daemon.log`** itself uses `gopkg.in/natefinch/lumberjack.v2`
   configured in `daemon.New` (`daemon.go:151-158`): 100MB max, 3
   backups, 7 days, compressed. Lumberjack handles rotation
   automatically on every write; `gt daemon rotate-logs` deliberately
   skips `daemon.log` because it's already managed.

2. **Dolt server logs** (and other long-fd'd logs) use copytruncate
   rotation in `log_rotation.go` because their child processes hold
   open file descriptors that can't be renamed out from under them.
   `RotateLogs` (`log_rotation.go:52-92`) walks
   `collectDoltLogFiles`, skips files under 100MB, and calls
   `copyTruncateRotate` per file. Runs on every heartbeat
   (`daemon.go:869`).

`log_rotation.go` also enforces a **500MB daemon disk budget** and
**7-day stale archive age** (`log_rotation.go:16-30`), deleting the
oldest `.gz` files when either threshold is exceeded.

`gt daemon rotate-logs --force` calls `ForceRotateLogs` which ignores
the 100MB threshold — useful for manual cleanup after an incident.

## Patrol types (the full zoo)

Beyond the core "keep agents running" responsibilities, the daemon
runs a catalog of scheduled maintenance tasks, each with its own Go
file and ticker. These are NOT Claude agents — they are pure Go
routines that do deterministic work. Many are named "dog" (the same
naming convention used for kennel pack workers) because they were
originally spawned as Claude-driven `dog` sessions and later
reimplemented as Go code once the workflow stabilized.

**Dolt patrols** (`dolt.go`, `dolt_remotes.go`, `dolt_backup.go`,
`jsonl_git_backup.go`, `dolt_backup.go`):

- Dolt server uptime supervision + 30s health check.
- Periodic `git push` of Dolt DBs to their configured remotes.
- Filesystem backup sync to `~/.dolt-archive/` (or configured dir).
- JSONL export + scrub + git push backup (spike-detection protected).

**Maintenance dogs** (`compactor_dog.go`, `checkpoint_dog.go`,
`doctor_dog.go`, `quota_dog.go`):

- `compactor_dog` — daily commit-history flatten for Dolt DBs when
  commit count exceeds threshold (default 2000).
- `checkpoint_dog` — every 10m, auto-commits dirty polecat worktrees
  so a session crash doesn't lose uncommitted work.
- `doctor_dog` — health monitor covering TCP, latency, db count, gc,
  zombies, backups, disk usage.
- `quota_dog` — scans for rate-limited sessions and rotates
  credentials via keychain swap.

**Scheduled operations** (`scheduled_maintenance.go`,
`main_branch_test_runner.go`):

- `scheduled_maintenance` — checks time-of-day window (default 03:00
  daily) and runs `gt maintain --force` when commit counts exceed
  threshold (default 1000). This is the FLATTEN stage of the Dolt
  lifecycle.
- `main_branch_test` — periodically runs quality gates on each rig's
  main branch to catch regressions from merges or direct pushes.

**Wisp / ephemeral cleanup** (`wisp_reaper.go`, `krc_pruner.go`):

- `wisp_reaper` — closes stale wisps (abandoned molecule steps, old
  patrol data) across all DBs. Prevents unbounded table growth.
- `krc_pruner` — automatic ephemeral (KRC) data cleanup.

**Dog management** (`handler.go`, `dog_molecule.go`):

- `handler.go` — dog lifecycle: cleanup stuck dogs, detect stale
  working dogs, reap idle dogs, dispatch plugins. See `handleDogs`
  at `handler.go:41-58`.
- `dog_molecule.go` — helpers for pouring `mol-dog-*` molecules
  (observability wisps) from daemon-owned patrol code.

**Convoy / feed** (`convoy_manager.go`, plus `internal/feed`):

- Event-driven convoy delivery with a periodic stranded-scan fallback.
- Dolt recovery callback triggers an immediate sweep after Dolt
  transitions healthy.

## Deep dives: large subsystem files

### `dolt.go` — DoltServerManager (~47KB)

Source: `/home/kimberly/repos/gastown/internal/daemon/dolt.go`.

The daemon's Dolt server lifecycle manager. `DoltServerManager`
struct (`dolt.go:116-150`) wraps a `DoltServerConfig` with mutex-
protected runtime state: `process`, `startedAt`, `lastCheck`,
backoff counters (`currentDelay`, `restartTimes`, `escalated`),
identity verification timestamp, health check warnings, and a
recovery callback (`onRecoveryFn`) invoked when Dolt transitions
unhealthy-to-healthy.

`DoltServerConfig` (`dolt.go:31-82`) exposes: `Enabled`, `External`
(daemon monitors only — does not start/stop), `Port`, `Host`,
`User`, `Password`, `DataDir`, `LogFile`, `AutoRestart`,
`RestartDelay` (5s default), `MaxRestartDelay` (5m), crash-loop
window parameters (`MaxRestartsInWindow`=5, `RestartWindow`=10m,
`HealthyResetInterval`=5m), and `HealthCheckInterval` (30s default).

The health check fires every 30s (`DefaultDoltHealthCheckInterval`,
`dolt.go:29`) independent of the 3-minute heartbeat, providing fast
crash detection. When health transitions unhealthy-to-healthy, the
recovery callback triggers a convoy manager sweep.

The manager implements exponential-backoff restarts with crash-loop
escalation: after `MaxRestartsInWindow` restarts within
`RestartWindow`, it stops retrying and escalates instead of spinning.
The backoff resets after `HealthyResetInterval` of continuous health.

### `compactor_dog.go` — Wisp compaction (~31KB)

Source: `/home/kimberly/repos/gastown/internal/daemon/compactor_dog.go`.

Runs every 24 hours (configurable via `CompactorDogConfig.IntervalStr`).
Two compaction modes (`compactorDogMode`, `compactor_dog.go:90-97`):

- **Flatten** (default): soft-resets to the root commit on main and
  commits all data as a single commit. Safe with concurrent writes.
- **Surgical**: interactive rebase via `DOLT_REBASE` that squashes
  old commits while preserving recent N as individual picks
  (`KeepRecent`, default 50). NOT safe with concurrent writes —
  retries once on graph-change errors (`surgicalMaxRetries=1`,
  `compactor_dog.go:43-44`).

Default commit threshold: 2000 (`defaultCompactorCommitThreshold`,
`compactor_dog.go:31`). This was raised from 500 to prevent the
escalation feedback loop where each compaction failure creates
beads/escalations that add more commits than the compactor can drain.

After successful compaction, runs `CALL dolt_gc()` with a 5-minute
timeout (`compactorGCTimeout`, `compactor_dog.go:36`) to reclaim
unreferenced chunks, then pushes via `DOLT_PUSH` with a 2-minute
timeout (`compactorPushTimeout`, `compactor_dog.go:38`).

ZFC exemption: executes imperatively in Go rather than via
agent-driven formula execution because operations require
`database/sql` connections, transactional state spanning multiple
queries, cleanup-on-failure error paths, concurrent write retry with
error classification, and row count integrity verification.
Observability uses `pourDogMolecule` + `closeStep`/`failStep` on the
`mol-dog-compactor` formula.

### `convoy_manager.go` — Event-driven convoy delivery (~22KB)

Source: `/home/kimberly/repos/gastown/internal/daemon/convoy_manager.go`.

`ConvoyManager` struct (`convoy_manager.go:53-115`) owns: `townRoot`,
`scanInterval` (default 30s), a context+cancel pair, a logger,
per-store `beadsdk.Storage` map (`stores` — "hq" plus per-rig names),
lazy `openStores` for Dolt-not-ready-at-startup, an `isRigParked`
predicate, a `gtPath` for subprocess calls, and several
concurrency-control fields (`started`, `recoveryMode`, `scanMu`,
`lastEventIDs`, `seeded`, `processedCloses`,
`processedLifecycleEvents`).

**Two goroutines** started by `Start()`:

1. **Event poll loop** — polls all beads stores (hq + per-rig,
   skipping parked rigs) every 5 seconds (`eventPollInterval`,
   `convoy_manager.go:24`) with exponential backoff to 60s on
   failure. The first poll cycle is warm-up only (advances
   high-water marks without processing events, preventing burst
   replay on daemon restart — `seeded` flag). Close events trigger
   `convoy.CheckConvoysForIssue`. A 1s lookback overlap
   (`eventPollLookback`) handles same-second-precision transitions.
   `processedCloses` and `processedLifecycleEvents` sync.Maps
   deduplicate across stores and poll cycles.

2. **Stranded scan loop** — every 30s, calls `gt convoy stranded
   --json` and processes each stranded convoy. A 5-minute grace
   period (`convoyGracePeriod`, `convoy_manager.go:32`) prevents
   auto-close of newly created convoys before `bd dep add` is
   visible in Dolt (GH#2303). In recovery mode (Dolt down), scan
   interval drops to 5s for fast retry.

### `jsonl_git_backup.go` — JSONL backup pipeline (~29KB)

Source: `/home/kimberly/repos/gastown/internal/daemon/jsonl_git_backup.go`.

Runs every 15 minutes (configurable). The pipeline:

1. **Export** each configured database to JSONL via `bd sql --json`
   against the Dolt server (`jsonlExportTimeout`=60s).
2. **Scrub** ephemeral data: the `scrubWhereClause`
   (`jsonl_git_backup.go:52-64`) filters out ephemeral beads,
   tombstones, infrastructure types (message, event, agent, convoy,
   molecule, role, merge-request, rig), wisp/convoy ID patterns, test
   prefixes, and CLI artifacts (`--help`, `Usage:` titles).
   `testPollutionPatterns` (`jsonl_git_backup.go:34-44`) provide
   additional regex-based test-data filtering.
3. **Spike detection** (`defaultSpikeThreshold`=50% delta,
   `jsonl_git_backup.go:29`) halts the pipeline if issue count
   changes dramatically between cycles, preventing data-corruption
   from propagating to the backup.
4. **Git commit and push** to the backup repo (default
   `~/.dolt-archive/git/`). Push timeout is 120s
   (`gitPushTimeout`). After `maxConsecutivePushFailures`=3
   consecutive failures, the pipeline stops attempting pushes until
   the next daemon restart.

Observability uses `pourDogMolecule` on `MolDogJSONL`.

## Notable patterns

- **Flock-based singleton over PID matching.** Every "is the daemon
  running?" check uses `daemon.lock` as the source of truth, not
  `ps` output. The PID file exists only to report the PID to callers
  that need it for display or signaling. This was a deliberate fix
  (ZFC: gt-utuk) for fragile process-discovery heuristics.

- **Nonce-based PID ownership.** PID files contain `"PID\nNONCE"`
  (`pidfile.go:11-30`) where the nonce is 8 random bytes generated at
  write time. This guards against PID reuse without relying on
  command-line string matching.

- **Heartbeat-loop single-threaded discipline.** Several mutable
  fields on the `Daemon` struct (`deaconLastStarted`, `syncFailures`,
  `bootLastSpawned`, `lastDoctorMolTime`, `lastMaintenanceRun`,
  `mayorZombieCount`, `jsonlPushFailures`) are explicitly documented
  as "only accessed from the heartbeat loop goroutine — no sync
  needed" (`daemon.go:78-118`). The only mutex is `deathsMu` for the
  mass-death detection ring buffer, because that's fed from a
  goroutine outside the main loop.

- **Shutdown lock stays resident.** The `shutdown.lock` file is never
  removed because `flock` works on inodes, not paths. If you delete
  it, concurrent callers end up locking different inodes.

- **Best-effort telemetry.** OpenTelemetry init is wrapped in
  warning-logs-but-continues (`daemon.go:280-302`). Telemetry
  failure never blocks startup.

- **Env propagation for spawned sessions.** `patrolConfig.Env`
  (`types.go:202-207`) is set via `os.Setenv` at daemon startup so
  every Claude session spawned by the daemon inherits it. This is how
  `GT_DOLT_PORT` reaches agent sessions without touching every
  `Manager.Start` call.

- **`ensureXxxRunning` idempotency.** Every `ensureXxxRunning`
  function is safe to call on every heartbeat. The expected hot path
  is "agent already alive, do nothing" — the restart path is the
  exception.

- **Pressure exemption for infrastructure.** `checkPressure` is only
  called for refineries, dogs, and polecats. Deacon, Witness, and
  Mayor always start regardless of system load — they're the
  recovery layer and can't be the thing being recovered.

## Category breakdown of the 33 files

**Main loop & lifecycle (3):**
`daemon.go`, `lifecycle.go`, `lifecycle_defaults.go`.

**State management (3):**
`types.go`, `pidfile.go`, `restart_tracker.go`.

**Signal/OS portability (6):**
`signals_unix.go`, `signals_windows.go`, `proc_unix.go`,
`proc_windows.go`, `pressure_darwin.go`, `pressure_linux.go`
(+ `pressure_other.go`, `pressure_windows.go` fallbacks).

**Dolt infrastructure (3):**
`dolt.go` (47KB — DoltServerManager), `dolt_remotes.go`,
`dolt_backup.go`.

**Backup / export (1):**
`jsonl_git_backup.go` (29KB — issue export, scrub, git push,
spike detection).

**Maintenance dogs (4):**
`checkpoint_dog.go`, `compactor_dog.go` (31KB — commit flatten),
`doctor_dog.go`, `quota_dog.go`.

**Scheduled operations (2):**
`scheduled_maintenance.go`, `main_branch_test_runner.go`.

**Cleanup (2):**
`wisp_reaper.go`, `krc_pruner.go`.

**Handlers (2):**
`handler.go` (dog dispatch), `convoy_manager.go`
(22KB — event-driven convoy delivery).

**Observability (3):**
`metrics.go`, `notification.go`, `dog_molecule.go`.

**Misc (1):**
`log_rotation.go`, `pressure.go`.

## Related wiki pages

- [`gt daemon`](../commands/daemon.md) — CLI wrapper (8 subcommands).
- [`gt up`](../commands/up.md) — launches the daemon via `ensureDaemon`.
- [`gt start`](../commands/start.md) — does NOT launch the daemon
  (deliberately, per Batch 3).
- [`gt down`](../commands/down.md), [`gt shutdown`](../commands/shutdown.md) —
  write `shutdown.lock`; daemon checks it every heartbeat.
- [`gt dashboard`](../commands/dashboard.md) — reads daemon heartbeat
  counter and Dolt env vars.
- [`gt doctor`](../commands/doctor.md) — `DaemonCheck` inspects daemon state.
- [Deacon (role)](../roles/deacon.md) — driven by daemon tick pokes;
  owns `heartbeat.json` / `paused.json` / `redispatch-state.json` /
  `feed-stranded-state.json` / `health-check-state.json` under
  `<townRoot>/deacon/`.
- [Mayor (role)](../roles/mayor.md) — restarted by `ensureMayorRunning`
  with zombie-debounce (3 consecutive misses required).
- [`internal/doctor`](doctor.md) — doctor's DaemonCheck reads this
  package's `LoadState` / `IsRunning`.
- [`internal/doltserver`](doltserver.md) — Dolt server lifecycle
  primitives; `DoltServerManager` in `dolt.go` wraps them.
- [`internal/config`](config.md) — `DaemonPatrolConfig` lives in this
  package's `types.go` but is managed by `gt config`; operational
  thresholds (pressure, timeouts, intervals) come from
  `config.OperationalConfig`.
- [`internal/session`](session.md) — tmux session management.
- [`internal/telemetry`](telemetry.md) — OpenTelemetry exporters.
- [`internal/beads`](beads.md) — `openBeadsStores`, beads store access.
- `internal/feed` — feed curator goroutine (no wiki page yet).
- [`internal/dog`](dog.md) — dog pack management consumed by
  `handler.go`.
- `internal/estop` — E-stop state checked at the top of every
  heartbeat (no wiki page yet).
- `internal/boot` — the separate boot-triage agent poked by this
  daemon each heartbeat (no wiki page yet).

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Daemon patrol loop partial failures:** The daemon runs many patrol
  handlers (dolt health, compactor, lifecycle, restart tracking,
  notification, pressure monitoring) in a single event loop.
  Individual handler failures are logged but do not stop the loop.
  If a handler leaves state dirty (e.g., compactor starts but does
  not finish), the next patrol cycle encounters stale state.
  **Present** — errors are logged per-handler, and the loop continues.

### Silent suppression (what errors are swallowed?)
- **Dolt server management:** `dolt.go` has 11 silent suppressions,
  including connection closes and process kills during Dolt health
  checks. **Present** — these are probe/cleanup operations where
  failure is non-fatal.
- **Compactor dog cleanup:** `compactor_dog.go` has 11 silent
  suppressions for tmux session teardown and worktree cleanup.
  **Absent** — if compactor session cleanup fails, the zombie
  session may persist without any diagnostic trail.
- **Lifecycle lock release:** `lifecycle.go` has 5 suppressed unlock
  calls. **Present** — POSIX flock semantics provide a safety net.

## Troubleshooting

For diagnostic workflows involving this entity, see
[Investigating: data-plane failures](../workflows/investigations/data-plane.md).

## Notes / open questions

- **`dolt.go` (47KB), `compactor_dog.go` (31KB),
  `convoy_manager.go` (22KB), and `jsonl_git_backup.go` (29KB)**
  are now grounded in the "Deep dives" section above. They remain
  unusually large for single files; extracting to their own packages
  is viable but blocked by shared daemon-local state.
- **The `doctor_dog.go` patrol pours `mol-dog-doctor` molecules with
  a 5-minute cooldown** (`daemon.go:130-136`) — this is the Option B
  throttling mentioned in the Dolt health code. Molecule pouring is
  the observability pathway; the actual control flow lives in
  `doltserver.go`.
- **The `Config.HeartbeatInterval` field in `types.go:25` defaults to
  5 minutes** but the actual heartbeat uses
  `recoveryHeartbeatInterval()` (3 min default) from operational
  config. The old field is effectively dead — worth confirming.
- **`daemon.go` has almost 100KB of code in a single file.** Historical
  accretion — the package started small and grew as each new patrol
  was bolted on. The pattern of "add a ticker, add a handler method"
  means `daemon.go` ends up mediating all the tickers.
- **Lifecycle config is written from three places**: `gt init`,
  `gt up` (via `EnsureLifecycleConfigFile`), and `daemon.New` itself.
  The triple-write is defensive: any path into the daemon creates or
  patches the file so lifecycle patrols are never missed.
- **The daemon cannot self-respawn.** If it crashes, the user must
  run `gt up` or `gt daemon start`, or rely on launchd/systemd
  supervision installed by `gt daemon enable-supervisor`. There is no
  in-package watchdog.
