---
title: internal/deacon
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
sources:
  - /home/kimberly/repos/gastown/internal/deacon/manager.go
  - /home/kimberly/repos/gastown/internal/deacon/heartbeat.go
  - /home/kimberly/repos/gastown/internal/deacon/pause.go
  - /home/kimberly/repos/gastown/internal/deacon/feed_stranded.go
  - /home/kimberly/repos/gastown/internal/deacon/redispatch.go
  - /home/kimberly/repos/gastown/internal/deacon/stale_hooks.go
  - /home/kimberly/repos/gastown/internal/deacon/stuck.go
tags: [package, agent-runtime, deacon, watchdog, heartbeat, patrol, redispatch, feed-stranded, stale-hooks, town-level]
---

# internal/deacon

Town-level watchdog infrastructure: lifecycle of the Deacon's
`hq-deacon` tmux session, the heartbeat file the daemon reads to
decide whether to poke, the pause file that makes a Deacon stand
down across restarts, and the four mechanical patrol primitives
the Deacon agent invokes during its cycle (stale-hook detection,
stuck-session health checking, recovered-bead redispatch with
escalation, and stranded-convoy feeding via dog dispatch).

**Go package path:** `github.com/steveyegge/gastown/internal/deacon`
**File count:** 7 non-test go files, 7 test files.
**Role:** [`deacon` role](../roles/deacon.md) — domain persona
**CLI command:** [`gt deacon`](../commands/deacon.md)
**Imports (notable):** [`internal/beads`](beads.md),
[`internal/config`](config.md), `internal/constants`,
[`internal/git`](go-packages.md), `internal/runtime`,
[`internal/session`](session.md), `internal/tmux`,
[`internal/util`](util.md), stdlib.
**Imported by (notable):** [`gt deacon`](../commands/deacon.md)
(`/home/kimberly/repos/gastown/internal/cmd/deacon.go`),
[`gt boot`](../commands/boot.md) (Boot reads heartbeat to triage),
the daemon goroutine in [`gt daemon`](../commands/daemon.md), and
the dolt server lifecycle path.

## What it actually does

Seven files, each owning one concern. The Deacon agent process
itself is a Claude (or compatible) instance running in the tmux
session this package creates; the Go code here is what brings the
session up, what the daemon polls to know it's alive, and what the
Deacon agent re-enters Go via `gt deacon ...` to perform mechanical
tasks safely.

### manager.go — session lifecycle

Source: `/home/kimberly/repos/gastown/internal/deacon/manager.go`.

- `Manager` struct (`manager.go:43-46`) bound to a `townRoot` and a
  `tmuxOps` interface (testable abstraction over the `tmux.Tmux`
  methods the manager uses).
- `tmuxOps` interface (`manager.go:24-40`) — defines exactly the
  tmux surface the manager touches, so tests can supply a fake.
- `NewManager(townRoot string) *Manager` (`manager.go:49-54`).
- `SessionName() string` package-level convenience
  (`manager.go:58-60`) and the receiver method
  (`manager.go:63-65`), both returning `session.DeaconSessionName()`
  (`"hq-deacon"`).
- `Manager.deaconDir() string` private — returns
  `<townRoot>/deacon/`.
- `Manager.Start(agentOverride string) error`
  (`manager.go:75-185`) — the elaborate startup ritual. Checks for
  an existing session; if present and the agent is alive, returns
  `ErrAlreadyRunning`; if present but dead, kills with
  `KillSessionWithProcesses`. Ensures `<townRoot>/deacon/` exists
  and runtime settings are installed, then builds the startup
  prompt as a `BuildStartupPrompt(BeaconConfig{Recipient: "deacon",
  Sender: "daemon", Topic: "patrol"}, "I am Deacon. Start patrol:
  ...")` (`manager.go:108-112`). Creates the tmux session via
  `NewSessionWithCommand` directly (avoiding the send-keys race
  documented at gh#280), immediately sets `remain-on-exit` for the
  PATCH-010 resilience trick, sets identity env vars from
  `config.AgentEnv`, records the pane PID into `GT_PANE_ID` for
  ZFC liveness checks (gt-qmsx), themes the session, waits for
  Claude to start (fatal if it doesn't), tracks the PID via
  `session.TrackSessionPID`, installs the auto-respawn hook, and
  finally accepts startup dialogs.
- `Manager.Stop() error` (`manager.go:188-213`) — best-effort
  `C-c`, 100 ms wait, then `KillSessionWithProcesses`. Returns
  `ErrNotRunning` if absent.
- `Manager.IsRunning() (bool, error)` (`manager.go:216-218`).
- `Manager.Status() (*tmux.SessionInfo, error)`
  (`manager.go:221-234`) — tmux session info, returns
  `ErrNotRunning` if absent.
- Sentinel errors `ErrNotRunning`, `ErrAlreadyRunning`
  (`manager.go:18-21`).

### heartbeat.go — Deacon liveness contract

Source: `/home/kimberly/repos/gastown/internal/deacon/heartbeat.go`.

The heartbeat is a JSON file the Deacon agent writes on every wake
cycle. The daemon reads its mtime to decide whether the Deacon is
alive enough to skip poking.

- Compile-time thresholds: `HeartbeatStaleThreshold = 5 * time.Minute`
  and `HeartbeatVeryStaleThreshold = 20 * time.Minute`
  (`heartbeat.go:16-25`). The very-stale threshold is deliberately
  longer than the patrol backoff-max (15 m) so a Deacon legitimately
  sleeping in await-signal does not get false-flagged.
- `Heartbeat` struct (`heartbeat.go:30-45`) — `Timestamp`, `Cycle`,
  `LastAction`, `HealthyAgents`, `UnhealthyAgents`.
- `HeartbeatFile(townRoot)` (`heartbeat.go:48-50`) →
  `<townRoot>/deacon/heartbeat.json`.
- `WriteHeartbeat(townRoot, hb)` (`heartbeat.go:54-83`) — writes
  the JSON file AND touches a legacy `.deacon-heartbeat` mtime
  marker for backward compatibility with shell scripts that
  predate the JSON format.
- `ReadHeartbeat(townRoot) *Heartbeat` (`heartbeat.go:87-101`) —
  returns nil on any error (file missing or unparseable).
- `Heartbeat.Age()`, `IsFresh()`, `IsStale()`, `IsVeryStale()`
  (`heartbeat.go:105-132`) — the three-state staleness ladder.
- `Touch(townRoot)` (`heartbeat.go:136-148`) — convenience: read
  existing heartbeat, increment cycle, write a minimal heartbeat
  with just timestamp and cycle.
- `TouchWithAction(townRoot, action, healthy, unhealthy)`
  (`heartbeat.go:151-165`) — same with a description and counters.

### pause.go — Deacon stand-down across restarts

Source: `/home/kimberly/repos/gastown/internal/deacon/pause.go`.

A pause is a file at `<townRoot>/.runtime/deacon/paused.json`
containing a `PauseState` JSON. While present, the Deacon agent
is contractually obligated to skip patrol actions, health checks,
and autonomous interventions. The pause file outlives Deacon
restarts.

- `PauseState` struct (`pause.go:13-25`) — `Paused bool`, `Reason`,
  `PausedAt`, `PausedBy`.
- `GetPauseFile(townRoot) string` (`pause.go:28-30`).
- `IsPaused(townRoot) (bool, *PauseState, error)`
  (`pause.go:35-52`) — tristate return: not paused / paused (with
  state) / error.
- `Pause(townRoot, reason, pausedBy) error` (`pause.go:55-76`).
- `Resume(townRoot) error` (`pause.go:79-88`) — idempotent file
  removal.

### feed_stranded.go — convoy feeding via dog dispatch

Source:
`/home/kimberly/repos/gastown/internal/deacon/feed_stranded.go`.

Stranded convoys are work batches that nobody is currently slinging
to a polecat. The Deacon's job is to (a) auto-close empty convoys,
(b) dispatch a dog to feed convoys with ready issues, (c) surface
"tracked but not ready" convoys for agent review without taking
autonomous action — that classification "requires agent judgment"
and Go does not attempt it (`feed_stranded.go:73-74, 246-260`).

Defaults are `DefaultMaxFeedsPerCycle = 3` and
`DefaultFeedCooldown = 10 * time.Minute`
(`feed_stranded.go:18-26`).

- `FeedStrandedState` (`feed_stranded.go:30-36`) — persisted to
  `<townRoot>/deacon/feed-stranded-state.json`.
- `ConvoyFeedState` (`feed_stranded.go:39-48`) — per-convoy
  cooldown tracking with `RecordFeed`, `IsInCooldown`,
  `CooldownRemaining` methods (`feed_stranded.go:157-180`).
- `StrandedConvoy`, `FeedResult`, `FeedConvoyResult`
  (`feed_stranded.go:51-89`) — input and output shapes.
- `LoadFeedStrandedState` / `SaveFeedStrandedState`
  (`feed_stranded.go:98-140`).
- `FindStrandedConvoys(townRoot)` (`feed_stranded.go:183-199`) —
  shells out to `gt convoy stranded --json`.
- `FeedStranded(townRoot, maxPerCycle, cooldown) *FeedResult`
  (`feed_stranded.go:206-334`) — the orchestrator. For each
  stranded convoy: classifies as needs_attention / closed / fed /
  cooldown / limit / error and updates state. Empty convoys
  auto-close via `gt convoy check`; feedable convoys dispatch via
  `gt sling <MolConvoyFeed> deacon/dogs --var convoy=<id>`
  (`feed_stranded.go:347-355`). Per-cycle and per-convoy rate
  limits both applied.
- `closeEmptyConvoy` / `dispatchFeedDog`
  (`feed_stranded.go:337-355`).
- `PruneFeedStrandedState(townRoot)` (`feed_stranded.go:359-381`)
  — drops state entries for closed convoys.
- `getConvoyStatus` (`feed_stranded.go:384-401`) — `bd show ...
  --json` shell-out.

### redispatch.go — RECOVERED_BEAD handling

Source: `/home/kimberly/repos/gastown/internal/deacon/redispatch.go`.

When a polecat dies and its work bead is recovered, the Deacon
re-dispatches the bead to a fresh polecat — but with rate limiting,
because some beads kill polecats deterministically and need to be
escalated to the Mayor instead of looping forever.

Defaults: `DefaultMaxRedispatches = 3`,
`DefaultRedispatchCooldown = 5 * time.Minute`
(`redispatch.go:20-28`).

- `RedispatchState`, `BeadRedispatchState`, `RedispatchResult`
  types (`redispatch.go:32-69`).
- State persistence to
  `<townRoot>/deacon/redispatch-state.json` via
  `LoadRedispatchState` / `SaveRedispatchState`
  (`redispatch.go:78-120`).
- Bead-state methods: `IsInCooldown`, `CooldownRemaining`,
  `ShouldEscalate`, `RecordAttempt`, `RecordEscalation`
  (`redispatch.go:137-172`).
- `Redispatch(townRoot, beadID, sourceRig, maxAttempts, cooldown)
  *RedispatchResult` (`redispatch.go:183-295`) — the main entry.
  Already-escalated → `"already-escalated"`; in cooldown →
  `"cooldown"`; over the limit → `"escalated"` (sends mail to
  Mayor); bead not currently `open` → `"skipped"` (treats unknown
  status as not-open per gt-sy8 to avoid re-dispatching closed
  beads on bd query failure); else `gt sling <bead> <rig>` and
  records the attempt.
- `PruneRedispatchState` (`redispatch.go:299-322`).
- `escalateToMayor` (`redispatch.go:364-390`) — sends a mail with
  subject `REDISPATCH_FAILED: <bead> (<n> attempts)` to `mayor/`.
- `ParseRecoveredBeadSubject` / `ParseRecoveredBeadBody`
  (`redispatch.go:394-420`) — message parsers for the mail format
  the Witness uses to report recovered beads.
- `resolveRigFromBead` (`redispatch.go:325-331`) — uses
  `beads.ExtractPrefix` + `beads.GetRigNameForPrefix` to figure
  out which rig owns a bead from its prefix.

### stale_hooks.go — orphaned hook detection

Source: `/home/kimberly/repos/gastown/internal/deacon/stale_hooks.go`.

A "hooked" bead has been claimed by an agent but the agent has
gone away. The Deacon scans hooked beads, checks whether the
assignee's tmux session is alive, and unhooks the bead so it can
be re-claimed by another agent.

- `StaleHookConfig` (`stale_hooks.go:21-26`), default `MaxAge = 1h`.
- `HookedBead`, `StaleHookResult`, `StaleHookScanResult` types
  (`stale_hooks.go:36-69`). `StaleHookResult` carries
  `PartialWork`, `WorktreeDirty`, `UnpushedCount` so callers can
  see if unhooking would lose work.
- `ScanStaleHooks(townRoot, cfg) (*StaleHookScanResult, error)`
  (`stale_hooks.go:77-155`) — the entry. Two staleness modes (per
  gt-pqf9x): if the assignee's session is confirmed dead, unhook
  immediately regardless of bead age; if the session can't be
  checked (unknown assignee format) AND the bead is older than
  MaxAge, fall back to age-based unhook.
- `listHookedBeads(townRoot)` (`stale_hooks.go:158-183`) — `bd
  list --status=hooked --json --flat --limit=0`.
- `assigneeToSessionName(assignee)` (`stale_hooks.go:187-193`) —
  delegates to `session.ParseAddress` so the codebase has one
  parser.
- `checkWorktreeState` / `assigneeToWorktreePath`
  (`stale_hooks.go:198-251`) — best-effort partial-work probe
  before unhooking. Supports both new (`rig/polecats/name/rigname/`)
  and old (`rig/polecats/name/`) layouts.
- `unhookBead(townRoot, beadID)` (`stale_hooks.go:254-259`) — `bd
  update <id> --status=open`.

### stuck.go — health-check state for stuck-session detection

Source: `/home/kimberly/repos/gastown/internal/deacon/stuck.go`.

Defaults `DefaultPingTimeout = 30s`, `DefaultConsecutiveFailures
= 3`, `DefaultCooldown = 5m` (`stuck.go:18-22`).

- `StuckConfig` and `LoadStuckConfig(townRoot)` which layers in
  `operational.deacon.*` settings overrides
  (`stuck.go:25-51`).
- `AgentHealthState` per-agent struct
  (`stuck.go:54-72`) and `HealthCheckState` map
  (`stuck.go:75-82`), persisted to
  `<townRoot>/deacon/health-check-state.json` via
  `LoadHealthCheckState` / `SaveHealthCheckState`
  (`stuck.go:90-133`).
- Per-agent transitions: `RecordPing`, `RecordResponse`,
  `RecordFailure`, `RecordForceKill`
  (`stuck.go:168-189`). `RecordResponse` and `RecordForceKill`
  both reset `ConsecutiveFailures`.
- `IsInCooldown`, `CooldownRemaining`, `ShouldForceKill(threshold)`
  predicates (`stuck.go:192-214`).
- `HealthCheckResult` struct (`stuck.go:150-158`).
- Sentinel errors `ErrAgentInCooldown`, `ErrAgentNotFound`,
  `ErrAgentResponsive` (`stuck.go:161-165`).

### Notable design choices

- **Patrol primitives are split between Go and the agent.** Go
  performs deterministic mechanical actions (rate limits,
  state files, bd queries, sling shell-outs). Anything requiring
  judgement is either deferred to the Deacon agent (e.g. hung
  dog clearing — see [`dog` package](dog.md)) or escalated to the
  Mayor (e.g. redispatch failures over the limit). The
  `feed_stranded.go` comment at lines 73-74 is the canonical
  statement: "requires agent judgment — Go surfaces the raw data
  but does not classify or act on them."
- **Heartbeat staleness is a 3-state ladder.** Fresh (<5m) →
  silent. Stale (5–20m) → may be a long operation, no action.
  Very stale (≥20m) → poke the Deacon. The 20m threshold is
  deliberately above the 15m patrol backoff-max so legitimate
  await-signal sleeps don't trigger pokes.
- **Pause file outlives restarts.** A paused Deacon stays paused
  even if it dies and is restarted by Boot — the file is checked
  on every action.
- **Session creation avoids the send-keys race.** The Deacon's
  startup uses `NewSessionWithCommand` (passes the startup
  command directly to tmux) rather than `NewSession` followed by
  `SendKeys`, because the latter has a documented race condition
  (gh#280) where keys can be sent before the shell is ready.
- **The auto-respawn hook.** PATCH-010: after creating the
  session, the manager calls `SetRemainOnExit(true)` and then
  `SetAutoRespawnHook`. Together these mean tmux will keep the
  pane around if the Deacon process dies and will automatically
  respawn it, breaking the crash loop where the daemon was
  repeatedly restarting a Deacon that crashed on startup.

## Related wiki pages

- [`deacon` role](../roles/deacon.md) — domain persona.
- [`gt deacon`](../commands/deacon.md) — CLI surface (15
  subcommands).
- [`internal/session`](session.md) — `DeaconSessionName`,
  `BuildStartupPrompt`, `BeaconConfig`, `TrackSessionPID`,
  `MergeRuntimeLivenessEnv`.
- [`internal/config`](config.md) — `AgentEnv`,
  `BuildStartupCommandFromConfig`, `LoadOperationalConfig` (used
  by `LoadStuckConfig`).
- [`internal/beads`](beads.md) — `ExtractPrefix`,
  `GetRigNameForPrefix`, used by `redispatch.resolveRigFromBead`.
- [`internal/util`](util.md) — `SetDetachedProcessGroup` for
  shell-outs, `AtomicWriteJSON` indirectly.
- [`mayor` package](mayor.md) — sibling town-level agent;
  redispatch escalates to it via mail.
- [`polecat` package](polecat.md) — `feed_stranded.go` and
  `redispatch.go` ultimately drive polecat work.
- [`dog` package](dog.md) — `feed_stranded.go` dispatches to dogs;
  health checks for hung dogs are deferred to the Deacon agent
  itself.
- [`gt convoy`](../commands/convoy.md), [`gt sling`](../commands/sling.md),
  [`gt mail`](../commands/mail.md) — shell-out targets from the
  patrol primitives.
- [`gt boot`](../commands/boot.md) — Boot triages the Deacon on
  every daemon tick.

## Notes / open questions

- Test pollution risk: `feed_stranded.go`, `redispatch.go`, and
  `stuck.go` all persist state files under `<townRoot>/deacon/`
  with similar shapes. Tests that don't isolate `townRoot` per
  test could clobber each other.
- The `tmuxOps` interface in `manager.go:24-40` is the only
  place in this batch where the manager mocks tmux behind an
  interface. The mayor and polecat managers go directly through
  `tmux.Tmux`. Worth a future consistency check.
- `feed_stranded.go` and `redispatch.go` both assume `gt` and
  `bd` are on PATH (they use `exec.Command("gt", ...)` and
  `exec.Command("bd", ...)`). If a future install layout breaks
  PATH for the Deacon's tmux session, these patrol primitives
  will silently start failing. The `manager.go:Start` flow does
  not currently verify these binaries.
- `stale_hooks.go` only knows about polecat and crew worktree
  layouts (`assigneeToWorktreePath` at lines 221-251). Dogs are
  not covered, presumably because dogs don't hook beads — but if
  that ever changes, the partial-work check would silently skip
  dog assignees.
- The legacy `.deacon-heartbeat` mtime file
  (`heartbeat.go:79-80`) is written but never read by Go code in
  this package. Comment says shell scripts predating the JSON
  file rely on it. Worth a sweep to see if any of those are still
  alive.
