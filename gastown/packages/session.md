---
title: internal/session
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/session/identity.go
  - /home/kimberly/repos/gastown/internal/session/lifecycle.go
  - /home/kimberly/repos/gastown/internal/session/names.go
  - /home/kimberly/repos/gastown/internal/session/pidtrack.go
  - /home/kimberly/repos/gastown/internal/session/registry.go
  - /home/kimberly/repos/gastown/internal/session/stale.go
  - /home/kimberly/repos/gastown/internal/session/startup.go
  - /home/kimberly/repos/gastown/internal/session/town.go
  - /home/kimberly/repos/gastown/internal/session/window_tint.go
  - /home/kimberly/repos/gastown/internal/session/agent_logging_unix.go
  - /home/kimberly/repos/gastown/internal/session/agent_logging_windows.go
tags: [package, platform-service, session, tmux, lifecycle, identity, polecat, agent-logging, pid-tracking]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression, partial-completion]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 2}
---

# internal/session

Agent session lifecycle: naming (every tmux session's name encodes
role/rig/agent), identity parsing (decomposing a session name back into
its constituent parts), starting and stopping sessions via tmux,
tracking session PIDs for safe cleanup, managing a rig-prefix registry
(since rig names map to short bead prefixes like `gt`, `bd`, `hop`),
formatting startup beacon messages, setting per-role tmux window tints,
and spawning the detached `gt agent-log` subprocess that streams Claude
Code JSONL conversation logs to VictoriaLogs.

Despite the "polecat" focus in the package doc comment, this package
is the session substrate for **all** agent roles:
[mayor](../roles/mayor.md), [deacon](../roles/deacon.md),
[witness](../roles/witness.md), [refinery](../roles/refinery.md),
[crew](../roles/crew.md), [polecat](../roles/polecat.md),
[dog](../roles/dog.md), boot, overseer. The comment on
`SessionConfig` at `lifecycle.go:22-35` is explicit: it unifies the
common startup pattern that was previously duplicated across every
role's session manager.

**Go package path:** `github.com/steveyegge/gastown/internal/session`
**File count:** 11 non-test go files (one Unix, one Windows stub), 10
test files.
**Imports (notable):** `github.com/google/uuid`,
[`internal/config`](config.md) (for `RuntimeConfig`, `AgentEnvConfig`,
cost tier, etc.), `internal/constants`, `internal/git`,
`internal/runtime`, [`internal/telemetry`](telemetry.md) (for
`RecordSessionStart`/`Stop`, `RecordAgentInstantiate`),
`internal/tmux`, [`internal/util`](util.md), stdlib.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:16`), and every
command that creates or touches an agent session —
[`gt boot`](../commands/boot.md), [`gt mayor`](../commands/mayor.md),
[`gt deacon`](../commands/deacon.md),
[`gt witness`](../commands/witness.md),
[`gt refinery`](../commands/refinery.md),
[`gt crew`](../commands/crew.md),
[`gt polecats`](../commands/polecats.md),
[`gt dog`](../commands/dog.md), [`gt hire`](../commands/hire.md),
[`gt send`](../commands/send.md), [`gt done`](../commands/done.md),
[`gt kill`](../commands/kill.md), the daemon lifecycle code, and the
`gt doctor`/`gt cleanup` family.

## What it actually does

Eleven files, each focused on one concern within session management.

### identity.go — role/rig parsing

Source: `/home/kimberly/repos/gastown/internal/session/identity.go`.

- `session.Role` string type with constants `RoleMayor`, `RoleDeacon`,
  `RoleOverseer`, `RoleWitness`, `RoleRefinery`, `RoleCrew`,
  `RolePolecat`, `RoleDog` (`identity.go:10-21`).
- `session.AgentIdentity` struct (`identity.go:24-30`): `Role`, `Rig`,
  `Name` (crew/polecat/dog name), `Prefix` (beads prefix for
  rig-level agents like `gt`, `bd`, `hop`).
- `session.ParseAddress(address) (*AgentIdentity, error)`
  (`identity.go:32-98`) — decomposes a user-facing address like
  `"gastown/polecat-toast"` or `"mayor"` into an `AgentIdentity`.
- `session.ParseSessionName(session) (*AgentIdentity, error)`
  (`identity.go:100-104`) — decomposes a tmux session name like
  `gt-gastown-toast` using the default registry.
- `session.ParseSessionNameWithRegistry(session, registry
  *PrefixRegistry) (*AgentIdentity, error)` (`identity.go:106-...`)
  — same with an explicit prefix registry, used in tests and rig-
  switching contexts.

### names.go — canonical tmux session name factories

Source: `/home/kimberly/repos/gastown/internal/session/names.go`.

Every function here is the single source of truth for one session
pattern. Commands construct names via these helpers rather than
string interpolation, so future renames propagate cleanly.

- `session.DefaultPrefix = "gt"` (`names.go:9`) — default beads
  prefix when no rig-specific prefix is known.
- `session.HQPrefix = "hq-"` (`names.go:12`) — prefix for town-level
  services (Mayor, Deacon, Overseer, Boot).
- `session.MayorSessionName() string` → `"hq-mayor"` (`names.go:16-18`).
  "One mayor per machine — multi-town requires containers/VMs for
  isolation."
- `session.DeaconSessionName()` → `"hq-deacon"` (`names.go:22-24`).
- `session.WitnessSessionName(rigPrefix)` → `"<prefix>-witness"`
  (`names.go:28-30`).
- `session.RefinerySessionName(rigPrefix)` → `"<prefix>-refinery"`
  (`names.go:34`).
- `session.CrewSessionName(rigPrefix, name)` →
  `"<prefix>-<name>"` (`names.go:40`).
- `session.PolecatSessionName(rigPrefix, name)` →
  `"<prefix>-<name>"` (`names.go:46`). Note: crew and polecat use
  the same session naming pattern; disambiguation happens via agent
  identity lookups, not the session name itself.
- `session.OverseerSessionName()` → `"hq-overseer"` (`names.go:52`).
- `session.BootSessionName()` → `"hq-boot"` (`names.go:59`).
- `session.DogSessionName(name)` (`names.go:66`).

### registry.go — rig prefix ↔ rig name registry

Source: `/home/kimberly/repos/gastown/internal/session/registry.go`.

- `session.PrefixRegistry` struct (`registry.go:25-30`) — bi-directional
  map between rig prefixes (`gt`) and rig names (`gastown`).
- `session.NewPrefixRegistry() *PrefixRegistry`
  (`registry.go:32-34`).
- `session.DefaultRegistry() *PrefixRegistry` (`registry.go:103-108`)
  — process-global default (for tests and convenience).
- `session.SetDefaultRegistry(r *PrefixRegistry)`
  (`registry.go:110-...`).
- `session.InitRegistry(townRoot) error` (`registry.go:122-178`) —
  loads all rigs from the town config and populates the default
  registry. Called once during startup.
- `session.LegacySocketName(townRoot) string`
  (`registry.go:180-194`).
- `session.PrefixFor(rigName) string` (`registry.go:196-202`) —
  `rigName` → prefix lookup.
- `session.BuildPrefixRegistryFromTown(townRoot)
  (*PrefixRegistry, error)` (`registry.go:204-242`) — file-system
  based construction.
- `session.BuildPrefixRegistryFromFile(path)
  (*PrefixRegistry, error)` (`registry.go:244-274`).
- `session.HasKnownPrefix(s string) bool` (`registry.go:276-300`) —
  does `s` start with any prefix we know about?
- `session.IsKnownSession(sess string) bool`
  (`registry.go:302-...`) — is this a Gas Town-owned tmux session?

### lifecycle.go — start/stop session machinery

Source: `/home/kimberly/repos/gastown/internal/session/lifecycle.go`.

The file that unified the startup pattern across all the role-
specific managers. Before this existed, polecat / mayor / boot /
deacon / witness / refinery / crew / dog session managers each
manually coordinated `config`, `runtime`, `session`, and `tmux`.

- `session.SessionConfig` struct (`lifecycle.go:36-115`) — all the
  inputs the unified startup routine needs: SessionID, WorkDir,
  Role, Rig, Name, AgentOverride, Prompt, TownRoot, plus runtime
  config pointers. Doc comment demonstrates the usage pattern.
- `session.StartResult` struct (`lifecycle.go:116-142`) — return
  value including the tmux session handle, runtime config, env
  vars actually used, and the GASTOWN run UUID.
- `session.StartSession(t *tmux.Tmux, cfg SessionConfig)
  (*StartResult, error)` (`lifecycle.go:144-310`) — the unified
  entry point. Validates required fields, resolves agent config,
  builds env vars and startup command via `config` package helpers,
  creates the tmux session, records telemetry (`RecordSessionStart`,
  `RecordAgentInstantiate` via
  `RecordAgentInstantiateFromDir`), tracks the session's PID via
  `TrackSessionPID`, and returns the result.
- `session.RecordAgentInstantiateFromDir(ctx, runID, resolvedAgent,
  role, agentName, sessionID, rigName, townRoot, issueID, workDir)`
  (`lifecycle.go:312-342`) — synthesises an `AgentInstantiateInfo`
  from the given working directory (reading git branch/commit) and
  emits the root GASTOWN telemetry event.
- `session.StopSession(t, sessionID, graceful bool) error`
  (`lifecycle.go:344-385`) — stops a tmux session. When `graceful`
  is true, sends a shutdown signal to the agent first and waits.
- `session.MergeRuntimeLivenessEnv(envVars, runtimeConfig)
  map[string]string` (`lifecycle.go:387-425`) — merges liveness-
  related env from `RuntimeConfig` into the outgoing env map.
- `session.KillExistingSession(t, sessionID, checkAlive bool)
  (bool, error)` (`lifecycle.go:427-465`) — idempotent kill with
  optional liveness verification. Returns `true` if a session was
  actually killed.
- `session.ShutdownDelay() time.Duration` (`lifecycle.go:467-...`)
  — resolves the grace-period duration for graceful shutdown.

### pidtrack.go — per-session PID tracking

Source: `/home/kimberly/repos/gastown/internal/session/pidtrack.go`.

Tracks the root pane PID of every known session so cleanup code can
safely kill the whole subtree even after tmux has lost track.

- `pidsDir(townRoot)` (`pidtrack.go:25-27`) — private: the PID
  tracking directory is `<townRoot>/.runtime/pids/`. Comment notes
  that tmux session names are globally unique (they include the rig
  name), so a single flat directory works.
- `session.TrackSessionPID(townRoot, sessionID string, t
  *tmux.Tmux) error` (`pidtrack.go:45-58`) — queries tmux for the
  session's pane PID, then calls `TrackPID`.
- `session.TrackPID(townRoot, sessionID string, pid int) error`
  (`pidtrack.go:60-73`) — atomically writes a JSON file containing
  the PID and the process start time to
  `<townRoot>/.runtime/pids/<sessionID>.json`. Start time is
  fingerprinted so recycled PIDs don't match.
- `session.UntrackPID(townRoot, sessionID string)`
  (`pidtrack.go:75-84`) — removes the tracking file.
- `session.KillTrackedPIDs(townRoot) (killed int,
  errSessions []string)` (`pidtrack.go:86-...`) — iterates tracked
  PIDs and signals each after verifying via start-time fingerprint.
- `pidStartTimeFunc` — test-overridable function pointer at
  `pidtrack.go:19`. **Test hazard:** pidtrack tests must NOT use
  `t.Parallel()` because they mutate this package-level variable
  without synchronization. (Comment at `pidtrack.go:17-19`.)

### stale.go — "is this session old enough to be stale?"

Source: `/home/kimberly/repos/gastown/internal/session/stale.go`.

- `session.SessionCreatedAt(sessionName) (time.Time, error)`
  (`stale.go:12-21`) — queries tmux for the session's creation
  timestamp.
- `session.ParseTmuxSessionCreated(created string) (time.Time, error)`
  (`stale.go:23-30`) — parses tmux's timestamp format.
- `session.StaleReasonForTimes(messageTime, sessionCreated
  time.Time) (bool, string)` (`stale.go:32-...`) — comparison
  helper that returns `(isStale, reason)`.

### startup.go — beacon and initial prompt formatting

Source: `/home/kimberly/repos/gastown/internal/session/startup.go`.

- `session.BeaconRecipient(role, name, rig string) string`
  (`startup.go:14-27`) — picks the beacon target based on role.
- `session.BeaconConfig` struct (`startup.go:29-68`) — carries the
  inputs for rendering a startup beacon message.
- `session.FormatStartupBeacon(cfg BeaconConfig) string`
  (`startup.go:69-127`) — renders the beacon text that gets
  included in the first nudge to a freshly spawned agent.
- `session.BuildStartupPrompt(cfg BeaconConfig, instructions string)
  string` (`startup.go:128-...`) — combines the beacon with any
  role-specific instructions to produce the final initial prompt.

### town.go — town-wide session operations

Source: `/home/kimberly/repos/gastown/internal/session/town.go`.

- `session.TownSession` struct (`town.go:14-21`) — represents one
  session for town-wide bulk operations.
- `session.TownSessions() []TownSession` (`town.go:22-32`) — lists
  the canonical town-level sessions (mayor, deacon, overseer, boot).
- `session.StopTownSession(t, ts, force bool) (bool, error)`
  (`town.go:33-46`).
- `session.StopTownSessionWithCache(t, ts, force, cache
  *tmux.SessionSet) (bool, error)` (`town.go:47-83`) —
  same with pre-fetched session set to avoid O(N²) tmux calls in
  bulk shutdown.
- `session.WaitForSessionExit(t, sessionID, timeout time.Duration)
  bool` (`town.go:84-...`) — polls tmux for session disappearance
  up to `timeout`.

### window_tint.go — per-role tmux window styling

Source: `/home/kimberly/repos/gastown/internal/session/window_tint.go`.

- `session.ResolveWindowTint(rig, role string) *tmux.WindowStyle`
  (`window_tint.go:21-101`) — looks up the configured tint for a
  (rig, role) combination, falling back to theme defaults.
- `session.IsWindowTintEnabled(rig string) bool`
  (`window_tint.go:102-...`) — per-rig enable flag.

### agent_logging_unix.go / agent_logging_windows.go — agent-log spawner

Sources:
`/home/kimberly/repos/gastown/internal/session/agent_logging_unix.go`,
`/home/kimberly/repos/gastown/internal/session/agent_logging_windows.go`.

- `session.ActivateAgentLogging(sessionID, workDir, runID string)
  error` — spawns a **detached** `gt agent-log` process to stream
  the session's Claude Code JSONL conversation log to
  VictoriaLogs (`agent_logging_unix.go:31-...`). Key properties:
  - Started with `Setsid` so it survives the parent's exit
    (Unix-only; Windows stub is a no-op).
  - Single-watcher guarantee via PID file at
    `/tmp/gt-agentlog-<session>.pid`. Any previous watcher is
    killed before spawning a new one.
  - `--since` is set to ~60 s before `now()` so only the current GT
    session's JSONL files are ingested, excluding pre-existing
    user sessions or other Gas Town rigs sharing the same work
    directory.
  - `runID` is the GASTOWN waterfall primary key; passed so every
    agent.event the subprocess emits carries the same `run.id` for
    correlation.
  - **Opt-in:** caller must check `GT_LOG_AGENT_OUTPUT=true` before
    calling. See [`internal/telemetry`](telemetry.md#privacy-summary-answer-to-wiki-9u4)
    for the privacy model around agent output logging.
- `session.DeactivateAgentLogging(sessionID string)`
  (`agent_logging_unix.go:88-...`) — Unix implementation; stops the
  detached watcher by reading the PID file and sending SIGTERM.
- Windows stubs (`agent_logging_windows.go`) return nil / no-op
  because the detached-subprocess approach relies on Unix-specific
  `Setsid` and `SIGTERM` semantics.

### Internals / Notable implementation

- **Session-name uniqueness** — every session name contains the rig
  prefix, so the `.runtime/pids/` directory can be flat across the
  whole town without collision (`pidtrack.go:25-27` comment).
- **Default registry is process-global mutable state** — tests that
  exercise `ParseSessionName` and friends have to save and restore
  the default registry (`identity_test.go` does this explicitly).
- **Telemetry is deeply wired in.** `StartSession` emits both
  `RecordSessionStart` and `RecordAgentInstantiate` unconditionally;
  the latter carries the GASTOWN run UUID that every downstream
  event is correlated to. See [`internal/telemetry`](telemetry.md)
  for the full event set.
- **Graceful shutdown uses a grace period.** `StopSession(graceful:
  true)` waits up to `ShutdownDelay()` after signalling, then
  `KillExistingSession` as a last resort.

### Usage pattern

```go
// Start a polecat session (abbreviated):
result, err := session.StartSession(t, session.SessionConfig{
    SessionID: session.PolecatSessionName("gt", "toast"),
    WorkDir:   "/home/kimberly/gt/gastown/rigs/toast",
    Role:      "polecat",
    Rig:       "gastown",
    Name:      "toast",
    TownRoot:  "/home/kimberly/gt",
    Prompt:    "gt prime && bd ready",
})
// result.SessionID is live in tmux, PID is tracked, telemetry is emitted.
```

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; imports session from root
  command init.
- [internal/config](config.md) — session consumes `RuntimeConfig`,
  `AgentEnvConfig`, and the `BuildStartupCommand*` family.
- [internal/telemetry](telemetry.md) — session emits
  `RecordSessionStart`, `RecordSessionStop`,
  `RecordAgentInstantiate` during every lifecycle event.
- [internal/workspace](workspace.md) — upstream: provides the
  `townRoot` that `session.InitRegistry` and `pidtrack` use.
- [internal/util](util.md) — `AtomicWriteJSON` used by pidtrack, and
  process-group helpers for any subprocess calls.
- [gt boot](../commands/boot.md), [gt hire](../commands/hire.md),
  [gt mayor](../commands/mayor.md), [gt polecats](../commands/polecats.md),
  [gt done](../commands/done.md), [gt kill](../commands/kill.md) —
  prominent user-facing consumers.
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Partial completion (what doesn't it clean up?)
- **Session created but post-startup steps fail:** `StartSession` at
  `lifecycle.go:200-201` creates the tmux session, then runs steps
  5-14 (env vars, theme, wait-for-agent, respawn hook, PID tracking,
  agent logging). Most post-creation failures are silently swallowed
  (see below), but if `WaitFatal` is true and `WaitForCommand` fails
  at `lifecycle.go:237-241`, the session is killed and an error is
  returned. However, steps 5-7 (`SetEnvironment`, `SetRemainOnExit`,
  `ConfigureGasTownSession`) have already fired and their side effects
  (tmux server state) are cleaned up by the kill. **Present** — the
  kill at `lifecycle.go:239` handles this case.
- **Agent-log watcher orphan on crash:** `ActivateAgentLogging` at
  `agent_logging_unix.go:31-82` spawns a detached process with
  `Setsid`. If the parent `gt` process crashes without calling
  `DeactivateAgentLogging`, the watcher persists until a future session
  start for the same `sessionID` kills it via the PID file at
  `agent_logging_unix.go:39`. **Present** — the PID file mechanism
  handles restart, but a crash of an entirely different session name
  leaves the watcher orphaned until the next `gt` process with the
  same session ID starts.

### Silent suppression (what errors are swallowed?)
- **tmux SetEnvironment failures:** `lifecycle.go:222-227` calls
  `t.SetEnvironment` for every env var and discards the error with
  `_ = t.SetEnvironment(...)`. If tmux rejects an env var (e.g., the
  session died between creation and env-set), the session runs with
  missing environment. **Absent** — no log or warning. The agent may
  start without `GT_ROLE`, `GT_RIG`, or `GT_RUN`, causing silent
  failures in downstream code that reads those vars.
- **Remain-on-exit failure:** `lifecycle.go:208` — `_ =
  t.SetRemainOnExit(cfg.SessionID, true)`. If this fails, the session
  dies on exit instead of remaining for inspection. **Absent** — no
  warning that crash-debugging capability was lost.
- **Theme application failure:** `lifecycle.go:232` — `_ =
  t.ConfigureGasTownSession(...)`. **Absent** — cosmetic, but no
  indication the theme was not applied.
- **PID tracking failure:** `lifecycle.go:288` — `_ =
  TrackSessionPID(...)`. **Absent** — if PID tracking fails, orphan
  cleanup in `KillTrackedPIDs` will miss this session's processes.
  The doc comment at `pidtrack.go:44` says callers should treat this
  as non-fatal, but there is no log message.
- **GT_PANE_ID capture failure:** `lifecycle.go:282-284` — if
  `GetPaneID` fails, the pane ID env var is not set, and liveness
  checks fall back to the slower scanning path. **Absent** — no log.

### Cross-platform concerns
- **Windows agent logging is a no-op:**
  `agent_logging_windows.go` returns nil for `ActivateAgentLogging`
  and is a no-op for `DeactivateAgentLogging`. Windows agents get no
  conversation log streaming to VictoriaLogs even when
  `GT_LOG_AGENT_OUTPUT=true`. **Untested** — the stub exists but the
  feature is architecturally absent on Windows.

## Notes / open questions

- The package doc comment says "polecat session lifecycle management"
  (`identity.go:1`, `lifecycle.go:1`), but the code covers all role
  types. Stale doc comment vs. current scope.
- Crew and polecat session names are structurally identical
  (`<prefix>-<name>`). Downstream code that needs to distinguish
  them must consult the agent identity lookup, not just parse the
  session name. Possible source of subtle bugs if future code paths
  assume name uniqueness per (rig, role).
- `ActivateAgentLogging`'s 60-second `--since` window is hard-coded
  (`agent_logging_unix.go:23-25` comment). If a spawn is slow and
  more than 60 s elapses between session creation and log
  activation, the watcher could miss the first few events.
- The Windows agent-logging stub is a no-op, so Windows agents get
  no conversation log streaming to VictoriaLogs even when
  `GT_LOG_AGENT_OUTPUT=true` is set. Drift between platforms.
- `pidStartTimeFunc` test hazard: tests can't use `t.Parallel()`.
  This is worth flagging in any future test-suite refactor.
