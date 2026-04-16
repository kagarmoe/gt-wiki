---
title: internal/mayor
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/mayor/manager.go
  - /home/kimberly/repos/gastown/internal/mayor/cleanup.go
  - /home/kimberly/repos/gastown/internal/mayor/process_unix.go
  - /home/kimberly/repos/gastown/internal/mayor/process_windows.go
tags: [package, agent-runtime, mayor, lifecycle, acp, tmux, town-level]
---

# internal/mayor

Lifecycle and dual-mode session management for the Mayor — Gas Town's
town-level orchestrator agent. The package owns Mayor's tmux session,
Mayor's headless ACP (Agent Client Protocol) session, the
mutual-exclusion logic between the two modes, the ACP PID/agent
sidecar files that signal "an ACP Mayor is alive", and the cleanup
veto check that prevents polecat workspaces from being garbage-
collected while a Mayor is reviewing diffs in ACP mode.

**Go package path:** `github.com/steveyegge/gastown/internal/mayor`
**File count:** 4 non-test go files (one Unix process probe, one
Windows process probe), 2 test files.
**Role:** [`mayor` role](../roles/mayor.md) — domain persona
**CLI command:** [`gt mayor`](../commands/mayor.md)
**Imports (notable):** [`internal/acp`](go-packages.md),
[`internal/config`](config.md), `internal/constants`,
[`internal/session`](session.md), `internal/templates`,
`internal/tmux`, [`internal/workspace`](workspace.md), stdlib +
`golang.org/x/sys/windows` on Windows.
**Imported by (notable):** [`gt mayor`](../commands/mayor.md)
(`/home/kimberly/repos/gastown/internal/cmd/mayor.go`),
[`gt boot`](../commands/boot.md) (Boot triages Mayor liveness),
the ACP proxy entry point, and any cleanup path that must respect
the cleanup veto.

## What it actually does

Four files. `manager.go` is the tmux/ACP lifecycle bulk;
`cleanup.go` handles the ACP sidecar files and the cleanup veto;
`process_unix.go` and `process_windows.go` are the two
implementations of `acpProcessAlive` for liveness probing.

### Public API — types

- `Mode` string type (`manager.go:29-36`) with constants `ModeTMUX`,
  `ModeACP`, `ModeBoth`, `ModeNone`. `ModeBoth` is observable when
  someone races a tmux start during ACP shutdown — `CombinedStatus`
  reports it rather than crashing.
- `MayorStatus` struct (`manager.go:39-45`) — `Active`, `Mode`,
  `Tmux *tmux.SessionInfo`, `ACPPid int`, deprecated `Running`
  field.
- `Manager` struct (`manager.go:48-50`) — bound to a `townRoot`. All
  methods are receivers on this type. `NewManager(townRoot)` is the
  only constructor (`manager.go:91-95`).
- `CleanupVetoChecker` struct (`cleanup.go:125-127`) and
  constructors `NewCleanupVetoChecker(townRoot)` /
  `NewCleanupVetoCheckerFromWorkDir(workDir)`
  (`cleanup.go:129-142`).

### Public API — sentinel errors

- `ErrNotRunning` — Mayor not running (`manager.go:23`).
- `ErrAlreadyRunning` — Mayor already running in tmux
  (`manager.go:24`).
- `ErrACPActive` — Mayor running in ACP mode (`manager.go:25`).
- `ErrCleanupVetoed` — cleanup blocked because ACP Mayor is alive
  (`cleanup.go:20`).

### Public API — lifecycle methods

- `Manager.Start(agentOverride string) error` (`manager.go:116-127`)
  — checks combined status first; if any mode is active, returns
  `ErrACPActive` or `ErrAlreadyRunning`. Otherwise delegates to
  `StartTMUX`.
- `Manager.StartTMUX(agentOverride string) error`
  (`manager.go:131-188`) — kills any zombie session via
  `session.KillExistingSession`, ensures `<townRoot>/mayor/`
  exists, resolves `CLAUDE_CONFIG_DIR` from
  `constants.MayorAccountsPath` via
  `config.ResolveAccountConfigDir` (the same pattern crew start
  uses), then calls the unified
  [`session.StartSession`](session.md) with a `BeaconConfig` of
  `Recipient: "mayor", Sender: "human", Topic: "cold-start"`. Sleeps
  `session.ShutdownDelay()` after creation so the session stabilises
  before returning.
- `Manager.StartACP(ctx context.Context, agentOverride, rigName
  string) error` (`manager.go:192-326`) — the headless mode entry.
  Refuses to start if another ACP Mayor is alive (because the PID
  file is shared and a second start would knock out the first).
  Resolves agent config, verifies the resolved agent supports ACP
  via `config.RuntimeConfigSupportsACP` (Claude does not — must use
  e.g. `opencode`), wires environment variables into the parent
  process (so the spawned proxy inherits them), constructs an
  `acp.Proxy` and `acp.Propeller`, gracefully stops any tmux Mayor
  via `m.Stop()` to free the slot, writes the ACP PID + agent
  sidecar files, then runs the proxy in the foreground via
  `proxy.Forward()`. Sidecar removal is deferred so the files vanish
  when the function returns. Three ACP invocation modes are handled
  (`manager.go:271-306`): native (binary IS the ACP adapter),
  subcommand (`<agent> acp [args...]`), and flag mode
  (`<agent> --experimental-acp ...`).
- `Manager.Stop() error` (`manager.go:329-352`) — best-effort
  `C-c`, 100 ms wait, then `KillSessionWithProcesses`. Returns
  `ErrNotRunning` if the tmux session does not exist. The ACP
  sidecar files are NOT touched here — only `StartACP`'s deferred
  cleanup or `CleanupStaleACP` removes them.
- `Manager.IsRunning() (bool, error)` (`manager.go:355-358`) —
  pure tmux session existence check.
- `Manager.Status() (*tmux.SessionInfo, error)`
  (`manager.go:361-374`) — tmux-only, returns `ErrNotRunning` if
  absent.
- `Manager.CombinedStatus() (*MayorStatus, error)`
  (`manager.go:53-82`) — the actual "what is Mayor doing right
  now" check. Tests both tmux and ACP and computes the `Mode`
  enum. CLI commands always go through this rather than `Status`
  alone.
- `Manager.IsActive() (bool, Mode)` (`manager.go:85-88`) — thin
  wrapper around `CombinedStatus`.
- `Manager.SessionName() string` (`manager.go:104-106`) and the
  package-level `SessionName()` (`manager.go:99-101`) — both
  return `session.MayorSessionName()` (`"hq-mayor"`).

### Public API — ACP sidecar files

- `ACPPidFilePath(townRoot)` →
  `<townRoot>/mayor/mayor-acp.pid` (`cleanup.go:23-25`).
- `WriteACPPid(townRoot)` (`cleanup.go:27-39`) — writes the current
  process's PID.
- `RemoveACPPid(townRoot)` (`cleanup.go:41-47`) — idempotent
  removal.
- `GetACPPid(townRoot) (int, error)` (`cleanup.go:49-63`).
- `ACPAgentFilePath(townRoot)` →
  `<townRoot>/mayor/mayor-acp.agent` (`cleanup.go:67-69`).
- `WriteACPAgent`, `RemoveACPAgent`, `GetACPAgent`
  (`cleanup.go:71-100`) — same shape, holds the agent name (e.g.
  `opencode`) so `gt mayor status` can show what's actually running.
- `IsACPActive(townRoot) bool` (`cleanup.go:102-115`) — reads PID
  file, calls `acpProcessAlive`. The doc comment is explicit: this
  function MUST NOT remove stale PID files as a side effect, because
  removal triggers the ACP proxy's PID-file watcher to shut itself
  down. Stale PID cleanup is deliberately separated into
  `CleanupStaleACP`.
- `IsACPActiveInWorkDir(workDir) bool` (`cleanup.go:117-123`) —
  calls `workspace.Find` to resolve the town root first.

### Public API — cleanup veto

- `CleanupVetoChecker.ShouldVetoCleanup() (bool, string)`
  (`cleanup.go:144-149`) — returns `(true, "ACP session is active
  - Mayor may be reviewing worker diffs")` if ACP Mayor is alive.
- `CleanupVetoChecker.VetoIfActive() error` (`cleanup.go:151-156`)
  — wraps the boolean as `ErrCleanupVetoed` for callers that prefer
  error-returning APIs.
- `CleanupVetoChecker.GetACPExpiry() (time.Time, bool)`
  (`cleanup.go:158-165`) — returns the PID file's mtime, used as a
  proxy for "when did this ACP Mayor start".
- `CleanupVetoChecker.CleanupStaleACP() error`
  (`cleanup.go:167-178`) — the ONLY function permitted to delete a
  stale PID file. Verifies that the recorded PID is actually dead
  before removing.

### Public API — prime context renderer

- `Manager.buildACPStartupPrompt() (string, error)`
  (`manager.go:379-395`) — composes the ACP-mode startup prompt:
  startup beacon (`session.FormatStartupBeacon` with
  `Recipient: "mayor", Sender: "human", Topic: "acp"`) followed by
  the rendered Mayor prime context.
- `GetMayorPrime(townRoot) (string, error)`
  (`manager.go:401-429`) — renders the `mayor` role template via
  `templates.New().RenderRole`, threading through `TownRoot`,
  `TownName`, `MayorSession`, `DeaconSession`, and stamping the
  result with `[prime-rendered-at: <RFC3339>]`. Used by the ACP
  startup path so the headless Mayor gets the same context a tmux
  Mayor would assemble via `gt prime`.

### Internals — process liveness

The Unix and Windows files are the two halves of the
`acpProcessAlive(pid int) bool` predicate.

- Unix (`process_unix.go:10-21`) — `os.FindProcess` followed by
  `process.Signal(syscall.Signal(0))`. The classic "ping with no
  signal" trick.
- Windows (`process_windows.go:17-28`) — `windows.OpenProcess` with
  `PROCESS_QUERY_LIMITED_INFORMATION`. `ERROR_ACCESS_DENIED` is
  treated as alive (the process exists, just inaccessible).

### Notable design choices

- **One Mayor per machine.** `MayorSessionName()` is a constant
  `"hq-mayor"` with no rig component (see comment in
  [`internal/session`](session.md)`/names.go`). Multi-town setups
  must use containers / VMs for isolation.
- **ACP and tmux are mutually exclusive.** The dance in
  `Start`/`StartTMUX`/`StartACP` is built around the invariant
  that the PID file's existence and a live tmux session never
  coexist intentionally. `ModeBoth` is a transient observation,
  not a target state.
- **The ACP cleanup veto is critical.** Without it, the polecat
  garbage collector would eat the worker workspaces a Mayor is
  reviewing diffs in. The veto reason string is
  user-visible and worth grepping for.
- **`Stop` does not touch the ACP files.** This is deliberate — a
  tmux-mode Stop must not delete an ACP-mode Mayor's PID file.
  Cleanup only happens via `StartACP`'s `defer` or
  `CleanupStaleACP`.

## Related wiki pages

- [`mayor` role](../roles/mayor.md) — domain persona; what a Mayor
  IS in Gas Town.
- [`gt mayor`](../commands/mayor.md) — CLI surface.
- [`internal/session`](session.md) — `StartSession`,
  `KillExistingSession`, `MayorSessionName`, `BeaconConfig`,
  `FormatStartupBeacon`, `ShutdownDelay`.
- [`internal/config`](config.md) — `ResolveAccountConfigDir`,
  `ResolveAgentConfigWithOverride`, `RuntimeConfigSupportsACP`,
  `AgentEnv`, `GetACPConfigFromRuntime`.
- [`internal/workspace`](workspace.md) — `workspace.Find` for the
  ACP-active-in-work-dir variant.
- [`deacon` package](deacon.md) — sibling town-level agent
  package.
- [`polecat` package](polecat.md) — the worker workspaces the ACP
  cleanup veto protects.
- [`gt boot`](../commands/boot.md) — Boot triages Mayor on each
  daemon tick.

## Notes / open questions

- `ModeBoth` is observable but not a configured state; CLI display
  paths in [`gt mayor status`](../commands/mayor.md) handle it as
  an edge case. Worth a future drift check that confirms the
  `Both` branch is never reachable under normal flows.
- The ACP startup path mutates `os.Setenv` in the parent process
  (`manager.go:216-218, 224-227`) so the spawned agent inherits
  identity vars. This is a side effect of `gt mayor acp` running
  in the foreground (no subprocess for the env to live in). It
  also means a `gt mayor acp` invocation pollutes the calling
  shell's environment if the process exits cleanly — observable
  only if someone backgrounds and reuses the shell.
- The `Stop` graceful path is a 100 ms wait then hard kill. There
  is no per-agent shutdown timeout knob here (unlike
  `session.ShutdownDelay()`'s use elsewhere). Possibly worth
  unifying.
- `GetMayorPrime` returns the rendered text + a timestamp banner.
  Re-rendering on every ACP start is intentional but it does mean
  the prime context can drift between a tmux and ACP Mayor that
  start at different times. Not a bug, but worth noting for
  diagnostics.
