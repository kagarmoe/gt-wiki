---
title: internal/tmux
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/tmux/tmux.go
  - /home/kimberly/repos/gastown/internal/tmux/theme.go
  - /home/kimberly/repos/gastown/internal/tmux/theme_resolver.go
  - /home/kimberly/repos/gastown/internal/tmux/flock_unix.go
  - /home/kimberly/repos/gastown/internal/tmux/flock_windows.go
  - /home/kimberly/repos/gastown/internal/tmux/process_group_unix.go
  - /home/kimberly/repos/gastown/internal/tmux/process_group_windows.go
  - /home/kimberly/repos/gastown/internal/tmux/descendants_stub.go
  - /home/kimberly/repos/gastown/internal/tmux/descendants_windows.go
  - /home/kimberly/repos/gastown/internal/tmux/sysproc_unix.go
  - /home/kimberly/repos/gastown/internal/tmux/sysproc_windows.go
tags: [package, tmux, session, nudge, keybindings, theme, platform-wrapper]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression, cross-platform]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/tmux

The tmux wrapper library: a single `Tmux` struct plus a pile of package-level
helpers that shell out to the `tmux` binary, parse its output, and encode every
piece of arcana Gas Town has learned about driving tmux reliably under
automation. This is the layer every other session-adjacent package in the repo
builds on — [`internal/session`](session.md) is the heaviest consumer,
[`internal/daemon`](daemon.md), [`internal/mayor`](mayor.md),
[`internal/polecat`](polecat.md), [`internal/deacon`](deacon.md),
[`internal/witness`](witness.md), [`internal/refinery`](refinery.md),
[`internal/crew`](crew.md), and [`internal/dog`](dog.md) all import it, and
so do the top-level commands that touch sessions directly
([`gt peek`](../commands/peek.md), [`gt cycle`](../commands/cycle.md),
[`gt town`](../commands/town.md), [`gt session`](../commands/session.md),
[`gt status`](../commands/status.md), [`gt nudge`](../commands/nudge.md),
[`gt feed`](../commands/feed.md), and more).

**Go package path:** `github.com/steveyegge/gastown/internal/tmux`
**File count:** 11 non-test go files (1 main + 2 themes + 8 platform-
specific pairs), 10 test files.
**Imports:** stdlib (`context`, `os/exec`, `regexp`, `sync`, `syscall`),
[`internal/config`](config.md), `internal/constants`,
[`internal/telemetry`](telemetry.md). Notably **not** `internal/session` —
this package is below session in the dependency graph.
**Imported by (notable):** [`internal/session`](session.md) on every call
path that creates/kills/nudges a session, [`internal/daemon`](daemon.md) for
the liveness sweep, [`gt`](../binaries/gt.md) root init
(`/home/kimberly/repos/gastown/internal/cmd/root.go` sets the default socket
at startup), plus every role manager and tmux-adjacent command.

## What it actually does

Eleven files. One giant `tmux.go` (3,857 lines) is the core; `theme.go` and
`theme_resolver.go` handle color schemes; the remaining eight files are
paired Unix/Windows platform shims (`flock_*.go`, `process_group_*.go`,
`descendants_*.go`, `sysproc_*.go`).

### Package-level API (no receiver)

Source: `/home/kimberly/repos/gastown/internal/tmux/tmux.go`.

**Socket management (global mutable state).** Gas Town runs tmux on a
per-town socket (`tmux -L <socket>`) so multiple towns don't collide, and
so that test runs can isolate themselves from the user's default server.

- `SetDefaultSocket(name string)` / `GetDefaultSocket() string`
  (`tmux.go:125-136`) — package-level RWMutex-guarded default socket name.
  Set once during `gt` root init from the town's resolved socket; every
  `NewTmux()` call thereafter picks it up.
- `SocketDir() string` (`tmux.go:141-143`) — returns `/tmp/tmux-<uid>`.
  On macOS, tmux uses `/tmp` directly rather than `$TMPDIR`
  (`/var/folders/...`), so `os.TempDir()` is wrong here.
- `IsInSameSocket() bool` (`tmux.go:148-162`) — parses the `TMUX` env var
  to decide whether the current process is already attached to the town
  socket. Used by [`gt peek`](../commands/peek.md) and
  [`gt cycle`](../commands/cycle.md) to pick between `switch-client`
  (same socket) and `attach-session` (different socket or outside tmux).
- `BuildCommand(args...) *exec.Cmd` /
  `BuildCommandContext(ctx, args...) *exec.Cmd` (`tmux.go:166-180`) —
  factory for code that needs an `exec.Cmd` of its own (not all callers
  want `Tmux.run`). Always prepends `-u` (UTF-8, tmux PATCH-004 /
  issue #1219) and the `-L <socket>` flag when a default is set. Also
  calls `hideConsoleWindow(cmd)` so Windows children don't flash a
  console window.
- `SocketFromEnv() string` (`tmux.go:3687-...`) — resolves the town
  socket from the `GT_TOWN_SOCKET` env var embedded in tmux keybindings,
  used by subcommands spawned from bindings even when `InitRegistry` was
  never called in that process.
- `CurrentSessionName() string` (`tmux.go:3703-...`) — parses the
  current `TMUX` env var to extract the attached session name.
- `IsInsideTmux() bool` (`tmux.go:3176-3178`) — bool wrapper over
  `os.Getenv("TMUX") != ""`.
- `EnsureBindingsOnSocket(socket, townSocket string) error`
  (`tmux.go:3599-...`) — installs Gas Town's global keybindings
  (cycle, rig menu, agents menu, feed, mail click) onto a socket.
  Uses `-L <socket>` directly rather than the package-level default so
  cross-socket binding management still works.

**Validation helpers.**

- `validateSessionName(name) error` (`tmux.go:54-59`) — rejects names
  that don't match `^[a-zA-Z0-9_-]+$`, preventing shell injection and
  tmux's cryptic error-on-dots-and-colons behavior.
- `validateCommandBinary(command) error` (`tmux.go:68-113`) — parses
  a session command to extract the binary path (skipping `exec`, `env`,
  `KEY=VAL` assignments, PowerShell `$env:` prefixes, and `&` call
  operator) and `os.Stat`s it to catch "binary not found" at session
  creation time instead of letting tmux silently exit.

**Sentinels / constants.**

- `ErrNoServer`, `ErrSessionExists`, `ErrSessionNotFound`,
  `ErrSessionRunning`, `ErrInvalidSessionName`, `ErrIdleTimeout`
  (`tmux.go:42-49`).
- `noTownSocket = "gt-no-town-socket"` (`tmux.go:190`) — sentinel
  socket name used when no town is configured; any tmux call against it
  fails loudly rather than hitting the user's default server.
- `EnvAgentReady = "GT_AGENT_READY"` (`tmux.go:196`) — tmux session
  env var set by the agent's `gt prime --hook` SessionStart hook. Used
  by `WaitForCommand` as a ZFC-compliant fallback when `pane_current_command`
  still reads as a shell (wrapped agents, see internal issue gt-sk5u).
- `DefaultReadyPromptPrefix = "❯ "` (`tmux.go:2825`) — default prompt
  suffix the package looks for when deciding a pane is at a shell prompt.

### The `Tmux` struct

`Tmux` is a thin struct (`tmux.go:183-185`) holding just `socketName`.
Every method is `(t *Tmux)`. The struct holds no cache and no lock; all
concurrency control is either inside tmux itself or in the package-level
`sessionNudgeLocks sync.Map` (`tmux.go:30`).

- `NewTmux() *Tmux` (`tmux.go:201-210`) — uses `GetDefaultSocket()` then
  falls back to `GT_TOWN_SOCKET` env var.
- `NewTmuxWithSocket(socket string) *Tmux` (`tmux.go:216-218`) — explicit
  socket, primarily for tests (to isolate from the user's real server and
  prevent leaked keystrokes from hitting the user's prefix table).
- `run(args ...string) (string, error)` (`tmux.go:223-...`) — private.
  Builds `tmux -u [-L <sock>] <args>`, runs it, returns stdout. Unwraps
  common errors into the package's sentinel values via `wrapError`
  (`tmux.go:246-...`).

Method inventory (grouped):

#### Session lifecycle

- `NewSession(name, workDir)` (`tmux.go:271-287`) — plain detached
  session. Sets `window-size=latest` to defeat tmux 3.3+'s manual-80x24
  default for detached sessions.
- `NewSessionWithCommand(name, workDir, command)` (`tmux.go:298-361`)
  — **two-step creation**: create the session with a default shell
  first, set `remain-on-exit on`, then `respawn-pane -k` the command in
  place. Eliminates the race where a fast-failing command exits before
  the health check runs. On Windows (psmux) uses `send-keys` instead
  because psmux's `respawn-pane` doesn't accept a command argument.
  Also unsets `CLAUDECODE` from the global tmux environment so new
  sessions don't inherit it (nested-session detection failures).
- `NewSessionWithCommandAndEnv(name, workDir, command, env)`
  (`tmux.go:373-445`) — same but passes `-e KEY=VAL` flags at
  `new-session` time so the initial shell inherits env from the session
  itself, preventing stale vars (e.g., a parent mayor's `GT_ROLE`) from
  leaking into crew/polecat shells. Kills any stale same-named session
  on another socket first via `killSplitBrainSession`
  (`tmux.go:771-787`). Requires tmux ≥ 3.2.
- `checkSessionAfterCreate` (`tmux.go:447-494`) — private health check
  run right after `respawn-pane`.
- `EnsureSessionFresh` / `EnsureSessionFreshWithCommand`
  (`tmux.go:496-563`) — idempotent kill + recreate.
- `KillSession` (`tmux.go:566`), `KillSessionWithProcesses`
  (`tmux.go:597`), `KillSessionWithProcessesExcluding`
  (`tmux.go:671`) — kill the session and hunt reparented grandchildren
  via the process-group machinery below. The `Excluding` variant
  protects specific PIDs during recycle flows.
- `KillPaneProcesses` / `KillPaneProcessesExcluding`
  (`tmux.go:841-953`), `RenameSession` (`tmux.go:2468`).
- `RespawnPane` / `RespawnPaneWithWorkDir` (`tmux.go:3208-3245`) —
  hot-reload an agent in place; used by
  [`gt handoff`](../commands/handoff.md) and recycle flows.
- `SetRemainOnExit(pane, on)` (`tmux.go:3264`) — must be `on` before
  killing processes so respawn-pane works afterward.
- `ClearHistory(pane)` (`tmux.go:3255`) — resets scrollback.

#### Query / introspection

- `ServerPID` / `KillServer` / `IsAvailable`
  (`tmux.go:956-992`), `HasSession` (`tmux.go:1001`),
  `ListSessions` (`tmux.go:1025`),
  `ListSessionIDs` (`tmux.go:1134`).
- `SessionSet` struct + `NewSessionSet` / `GetSessionSet` /
  `Has` / `Names` (`tmux.go:1056-1131`) — **cache** of "which sessions
  currently exist" that callers pre-fetch once to avoid O(N²)
  `has-session` calls during bulk operations.
- Pane queries: `GetPaneCommand` (`tmux.go:1962`), `FindAgentPane`
  (`tmux.go:1989`), `GetPaneID` (`tmux.go:2045`), `GetPaneWorkDir`
  (`tmux.go:2060`), `GetPanePID` (`tmux.go:2076`).
- Session timestamps / state: `GetSessionActivity` (`tmux.go:2094`),
  `GetSessionCreatedUnix` (`tmux.go:3671`), `ResolveCurrentSession`
  (`tmux.go:2340`) — walks parent PID chain when `TMUX` env var is
  unreliable, `IsSessionAttached` (`tmux.go:1277`).
- `CapturePane` / `CapturePaneAll` / `CapturePaneLines`
  (`tmux.go:2300-2321`).
- Environment: `SetEnvironment`, `GetEnvironment`,
  `SetGlobalEnvironment`, `UnsetGlobalEnvironment`,
  `GetGlobalEnvironment`, `GetAllEnvironment` (`tmux.go:2385-2466`).
- `FindSessionByWorkDir(targetDir, processNames)`
  (`tmux.go:2267`), `GetSessionInfo(name)` (`tmux.go:2963`) →
  `SessionInfo` struct (`tmux.go:2477`).

#### Nudge protocol (the send-keys machinery)

The most delicate part of the package. `NudgeSession`
(`tmux.go:1601-1729`, core at `NudgeSessionWithOpts:1624`) delivers a
message to an agent with an eight-step protocol where every step
exists because it turned out to matter:

0. **Pre-delivery**: if the pane is in Claude Code's double-Escape
   Rewind menu (`isInRewindMode:1378`, `dismissRewindMode:1418`),
   dismiss it first (issue gt-8el).
1. **Exit copy/scroll mode** if `pane_in_mode == 1` (copy mode
   silently intercepts input).
2. **Sanitize** control chars via `sanitizeNudgeMessage`
   (`tmux.go:1350`).
3. **Send text** via `send-keys -l` in 512-byte chunks with 10 ms
   inter-chunk delays (`sendKeysChunkSize = 512` at `tmux.go:1509`,
   `sendMessageToTarget:1511`).
4. **Adaptive delay** scaling with message length
   (`adaptiveTextDelay:1491`, issue gt-0b5).
5. **Send Escape** to exit vim INSERT mode. Skipped for Copilot CLI
   (`GT_AGENT=copilot`) because Escape cancels in-flight generation
   there (issue hq-isz); callers can also opt out via
   `NudgeOpts.SkipEscape` for Gemini CLI.
6. **Wait 600 ms** — must exceed bash readline's `keyseq-timeout`
   (500 ms default) so ESC is processed alone, not as meta prefix
   for the subsequent Enter (ESC+Enter within 500 ms becomes M-Enter
   which does **not** submit the line).
6.5. **Post-Escape Rewind check**: our Escape may itself trigger
   Rewind if a previous Escape was still in the input buffer. If so,
   dismiss and re-send the message.
7. **Send Enter with verification** (`sendEnterVerified:1434`) —
   polls pane content to confirm Enter was processed, retrying with
   exponential backoff under load.
8. **Wake the pane** via `WakePaneIfDetached` (`tmux.go:1327`) to
   trigger SIGWINCH — Claude Code ignores typed input in detached
   panes until SIGWINCH arrives.

**Locking.** Two layers: in-process channel semaphore
(`acquireNudgeLock`/`releaseNudgeLock` at `tmux.go:1248-1268`,
`sessionNudgeLocks sync.Map` at `tmux.go:30`, timeout
`nudgeLockTimeout = 30s`), and cross-process flock
(`acquireFlockLock` in `flock_unix.go:16-42`) on
`<townRoot>/.runtime/nudge_queue/<session>/.lock` when
`NudgeOpts.TownRoot` is set. Every `gt nudge` invocation is a
separate OS process so the channel semaphore alone cannot prevent
interleaving (issue gt-ukl8).

Related nudge-adjacent methods: `NudgePane` (`tmux.go:1735-1790`) —
same protocol targeting a pane ID, `SendKeys` / `SendKeysRaw` /
`SendKeysReplace` / `SendKeysDebounced` / `SendKeysDelayed`
(`tmux.go:1171-1240`) for cases that don't need the full protocol,
`DisplayMessage` / `SendNotificationBanner` (`tmux.go:2489-2526`)
for status-line banners.

#### Startup dialog handling

Automated agent spawns hit a series of first-run dialogs that would
otherwise block indefinitely. `AcceptStartupDialogs` (`tmux.go:1799-...`)
dismisses them in order:

1. Workspace trust dialog — Claude's "Quick safety check" and Codex's
   "Do you trust the contents of this directory?"
   (`AcceptWorkspaceTrustDialog` at `tmux.go:1817-...`; the safe
   "continue" option is pre-selected so Enter accepts).
2. Bypass permissions warning — "Bypass Permissions mode" dialog that
   requires Down+Enter (`AcceptBypassPermissionsWarning` at
   `tmux.go:1892-...`).

`DismissStartupDialogsBlind` (`tmux.go:1941-...`) is a fire-and-forget
variant for callers that don't want to wait for pane content to
stabilize. Helper: `containsWorkspaceTrustDialog`
(`tmux.go:1853-...`).

#### Readiness, liveness, idleness

- `WaitForCommand(session, excludeCommands, timeout)` (`tmux.go:2670`)
  — wait until the pane's current command is *not* one of
  `excludeCommands` (typically `["bash","zsh","fish","pwsh"]`), with
  the `EnvAgentReady` fallback for wrapped agents.
- `WaitForShellReady` (`tmux.go:2708`), `WaitForRuntimeReady(session,
  rc, timeout)` (`tmux.go:2786`) — config-driven readiness that
  supports both `ReadyPromptPrefix` (prompt detection) and
  `ReadyDelayMs` (wall-clock floor) from
  [`internal/config`](config.md)'s `RuntimeTmuxConfig`.
- `WaitForIdle` (`tmux.go:2833`), `IsAtPrompt` (`tmux.go:2907`),
  `IsIdle` (`tmux.go:2934`), `IsAgentRunning` (`tmux.go:2527`),
  `IsRuntimeRunning` (`tmux.go:2556`), `IsAgentAlive` (`tmux.go:2644`).
- `ZombieStatus` enum + `CheckSessionHealth(session, maxInactivity)`
  (`tmux.go:2108-2183`) — classifies a session as healthy or some
  flavor of zombie. Consumed by [`internal/daemon`](daemon.md)'s
  liveness sweep and [`gt doctor`](../commands/doctor.md).

Helpers: `matchesPromptPrefix` (`tmux.go:2752`), `hasBusyIndicator`
(`tmux.go:2765`), `containsPromptIndicator` (`tmux.go:1865`),
`promptSuffixes = []string{">", "$", "%", "#", "❯"}`
(`tmux.go:1861`).

#### Theming, status, and keybindings

Theme application: `ApplyTheme` / `ClearTheme` (`tmux.go:3007-3021`),
`ApplyWindowStyle` (`tmux.go:3023`), `SetStatusFormat` (`tmux.go:3050`),
`SetDynamicStatus` (`tmux.go:3076`), `EnableMouseMode` (`tmux.go:3158`).
`ConfigureGasTownSession` (`tmux.go:3111`) is the one-shot that applies
theme + status format + dynamic status + cycle bindings + mail click +
feed binding in a single call. `roleIcons` (`tmux.go:3035`) maps role
name → status-line icon.

Keybindings: `SetCycleBindings` (`tmux.go:3462`) and the crew/town
aliases (`tmux.go:3287-3296`), `SetFeedBinding` (`tmux.go:3514`),
`SetAgentsBinding` (`tmux.go:3542`), `SetRigMenuBinding`
(`tmux.go:3565`), `SetMailClickBinding` (`tmux.go:3187`),
`SetPaneDiedHook` / `SetAutoRespawnHook` (`tmux.go:3755-3840`) —
tmux `pane-died` hook that respawns the agent automatically.
`EnsureBindingsOnSocket` (`tmux.go:3599`) installs global (non-session)
bindings onto a socket.

Smart binding detection (`isGTBinding`, `isGTBindingWithClient`,
`isGTBindingCurrent`, `getKeyBinding` at `tmux.go:3307-3411`) reads
existing bindings via `list-keys` so repeated
`ConfigureGasTownSession` calls are idempotent *and* preserve the
user's original fallback in the else-branch of an `if-shell` guard.
`safePrefixRe` (`tmux.go:3417`) and `sessionPrefixPattern()`
(`tmux.go:3426`) build a grep `-Eq` pattern from
`config.AllRigPrefixes()` so bindings only fire inside known Gas Town
sessions. Rig prefixes failing `^[a-zA-Z][a-zA-Z0-9-]{0,19}$` are
dropped as defense-in-depth against hand-edited `rigs.json` injection.

#### Client / window ops and cleanup

- `AttachSession` (`tmux.go:2323`), `SelectWindow` (`tmux.go:2329`),
  `SwitchClient` (`tmux.go:3275`), `WakePane` / `WakePaneIfDetached`
  (`tmux.go:1291-1335`).
- `CleanupOrphanedSessions(isGTSession func(string) bool)`
  (`tmux.go:3725`) — called by [`gt cleanup`](../commands/cleanup.md).

### Process-group machinery and platform shims

Sources: `process_group_unix.go`, `process_group_windows.go`,
`descendants_*.go`, `flock_*.go`, `sysproc_*.go`.

tmux creates agents as children of its own pane process. When the pane
exits but the agent has spawned grandchildren (Claude Code → node.js →
workers), those grandchildren reparent to init and become orphans.
`KillSessionWithProcesses` hunts them down via:

- `killProcessGroup(pgid)` — SIGTERM then SIGKILL to `-pgid` on Unix
  (`process_group_unix.go:12-16`); `proc.Kill` on the PID itself on
  Windows where POSIX process groups don't exist
  (`process_group_windows.go:13-19`).
- `getParentPID` / `getProcessGroupID` / `getProcessGroupMembers` —
  wrappers over `ps -o ppid=/pgid=` on Unix
  (`process_group_unix.go:20-57`); Windows models a "process group" as
  just the PID, using `tasklist` to check existence
  (`process_group_windows.go:21-85`).
- `getAllDescendants` (`tmux.go:809-839`) — transitive descendant walk
  from a root PID.
- `collectReparentedGroupMembers` (`tmux.go:789-807`) — finds orphans
  that share the pgid but aren't in the known descendant set (i.e.,
  they've reparented to init).
- `hasDescendantWithNames` / `hasDescendantWithNamesPosix`
  (`tmux.go:2210-2265`) — bounded-depth descendant search by command
  name. `hasDescendantWithNamesWindows` in `descendants_windows.go`
  uses the Toolhelp32 snapshot API; `descendants_stub.go` is the
  `!windows` build-tag stub that returns `false`.

Platform shims:

- `flock_unix.go` / `flock_windows.go` — `acquireFlockLock(lockPath,
  timeout) (func(), error)`. Unix uses `syscall.Flock` with
  `LOCK_EX|LOCK_NB` polled in a 100 ms loop. Used only by the nudge
  cross-process serialization.
- `sysproc_unix.go` / `sysproc_windows.go` — `hideConsoleWindow(cmd)`.
  No-op on Unix. On Windows sets `CREATE_NEW_PROCESS_GROUP |
  CREATE_NO_WINDOW (0x08000000)` so tmux subprocesses spawned from the
  Windows daemon don't flash a visible console window.

### Theming

Source: `theme.go`, `theme_resolver.go`.

- `Theme` struct (`theme.go:21-30`) with optional nested `WindowStyle`
  (`theme.go:10-13`). Theme has `Name`, `BG`, `FG`; window adds
  separate `BG`/`FG` for the tmux `window-style`.
- `DefaultPalette` (`theme.go:34-45`) — 10 curated themes: `ocean`,
  `forest`, `rust`, `plum`, `slate`, `ember`, `midnight`, `wine`,
  `teal`, `copper`.
- Special role themes: `MayorTheme()` returns `bg=default,fg=default`
  so the mayor session inherits the user's terminal (the mayor is the
  primary interactive session and should blend in — `theme.go:50-52`).
  `DeaconTheme()` is purple/silver (`theme.go:56-58`). `DogTheme()`
  is brown/tan (`theme.go:62-64`).
- `AssignTheme(rigName)` (`theme.go:79-81`) — consistent-hash (FNV-1a)
  of the rig name modulo the palette so the same rig always gets the
  same color. `AssignThemeFromPalette` lets callers supply a custom
  palette.
- `GetThemeByName(name) *Theme` (`theme.go:68-75`),
  `ListThemeNames() []string` (`theme.go:99-106`).

Resolver (separate file — drives theme selection from layered config):

- `ResolveSessionTheme(townRoot, rigName, role)` (`theme_resolver.go:13-47`)
  — priority order: per-rig settings
  (`<townRoot>/<rigName>/.gastown/rig.json` via
  `config.RigSettingsPath` → `config.LoadRigSettings`), then
  town-level mayor config (`<townRoot>/mayor/config.json` via
  `config.LoadMayorConfig`), then the builtin role theme map
  (`config.BuiltinRoleThemes`), finally the special-case fallbacks
  (Mayor/Deacon/Dog themes or `AssignTheme(rigName)`).
- `unresolvedTheme` sentinel (`theme_resolver.go:49`) distinguishes
  "not configured at this layer" from "explicitly disabled" (nil).
- `normalizeThemeRole` (`theme_resolver.go:148-157`) — maps legacy
  aliases like `coordinator` → mayor and `health-check` → deacon.

### Internals / notable implementation

- **Nudge mutex is a channel semaphore, not a sync.Mutex.** Channels
  support timed acquisition; a hung tmux must not permanently lock
  out nudges.
- **Adaptive delays everywhere.** Every timing constant has a github
  issue behind it (gt-0b5, gt-8el, gt-ukl8, gt-sk5u, hq-isz, #307,
  #280, #1219, #1548). The package is unusually empirical.
- **Windows is not a no-op.** Most platform abstractions have real
  Windows implementations (psmux respawn-pane differences, Toolhelp32
  descendant walking, `CREATE_NO_WINDOW` subprocess flags). The one
  visible no-op is `descendants_stub.go` — build tag `!windows`, so
  the "stub" is for non-Windows, not Windows.
- **Idempotent binding management.** `isGTBinding*` makes repeated
  `ConfigureGasTownSession` calls skip re-binding *and* captures the
  user's original binding as the else-branch of an `if-shell` guard,
  so GT bindings are conditional on being in a GT session.
- **Two-step session creation is a race fix.** Creating with
  `new-session -d <shell>` then `respawn-pane <command>` eliminates
  the race where `new-session -d '<command>'` + immediate health
  check fails if the command exits before the check runs.
- **`CLAUDECODE` gets unset on every session create.** If the tmux
  server was started from inside a Claude Code session, it inherits
  `CLAUDECODE=1` and every new session inherits it too, breaking
  nested-session detection for every new agent.

### Usage pattern

```go
t := tmux.NewTmux() // picks up the default socket
_ = t.NewSessionWithCommandAndEnv(name, workDir, startCmd, envMap)
_ = t.WaitForCommand(name, []string{"bash","zsh","pwsh"}, 10*time.Second)
_ = t.AcceptStartupDialogs(name)
theme := tmux.AssignTheme("gastown")
_ = t.ConfigureGasTownSession(name, &theme, "gastown", "toast", "polecat")
_ = t.NudgeSessionWithOpts(name, prompt, tmux.NudgeOpts{TownRoot: townRoot})
```

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; sets the package default
  socket at root init.
- [internal/session](session.md) — the primary consumer. Session's
  `lifecycle.go` calls `NewSessionWithCommandAndEnv`, `StopSession`,
  `TrackSessionPID` (via `GetPanePID`), and most of the nudge
  machinery.
- [internal/daemon](daemon.md) — uses `CheckSessionHealth`,
  `ListSessions`, `GetSessionSet`, `CleanupOrphanedSessions` for the
  liveness sweep and zombie reaping.
- [internal/config](config.md) — source of `RuntimeConfig`,
  `RuntimeTmuxConfig`, `ThemeConfig`, `RigSettingsPath`,
  `BuiltinRoleThemes`, and `AllRigPrefixes` consumed by the resolver
  and binding pattern builder.
- [internal/telemetry](telemetry.md) — tmux reports session creation
  and nudge events through telemetry hooks.
- [internal/runtime](runtime.md) — sibling package that builds on
  `Tmux` via the `startupPromptSession` interface to deliver startup
  prompts when the runtime can't take them as a CLI argument.
- Roles that live inside tmux sessions:
  [mayor](../roles/mayor.md), [deacon](../roles/deacon.md),
  [polecat](../roles/polecat.md), [crew](../roles/crew.md),
  [dog](../roles/dog.md), [refinery](../roles/refinery.md),
  [witness](../roles/witness.md).
- Commands that touch tmux directly: [gt peek](../commands/peek.md),
  [gt cycle](../commands/cycle.md), [gt town](../commands/town.md),
  [gt session](../commands/session.md), [gt status](../commands/status.md),
  [gt nudge](../commands/nudge.md), [gt feed](../commands/feed.md),
  [gt cleanup](../commands/cleanup.md), [gt doctor](../commands/doctor.md),
  [gt handoff](../commands/handoff.md).
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Silent suppression (what errors are swallowed?)
- **tmux command failures in session management:** `tmux.go` has 53
  instances of `_ =` across session operations. Many are
  `SetOption`, `SetEnvironment`, and `SendKeys` calls where the tmux
  server may have already died. **Present** — most are non-fatal
  cleanup or best-effort configuration. The critical session-creation
  calls properly propagate errors.

### Cross-platform concerns
- **6 platform shim pairs:** flock_unix/windows,
  process_group_unix/windows, sysproc_unix/windows,
  descendants_stub/windows. The Windows implementations provide
  different semantics (e.g., process group uses
  `CREATE_NEW_PROCESS_GROUP`, descendants uses Win32 job objects).
  **Untested** — no indication the Windows tmux integration has been
  tested end-to-end (tmux itself is Unix-only; Windows uses WSL or
  similar).

## Troubleshooting

For diagnostic workflows involving this entity, see
[Investigating: message delivery](../workflows/investigations/message-delivery.md).

## Detail tables

### Nudge protocol timing constants

| Constant | Value | Why | Source |
|---|---|---|---|
| `nudgeLockTimeout` | 30s | In-process channel semaphore timeout; prevents hung tmux from permanently locking out nudges | `tmux.go:1248-1268` |
| `sendKeysChunkSize` | 512 bytes | Max bytes per `send-keys -l` call; prevents tmux buffer overflow | `tmux.go:1509` |
| Inter-chunk delay | 10ms | Between 512-byte chunks during `sendMessageToTarget` | `tmux.go:1511-1535` |
| Post-Escape wait | 600ms | Must exceed bash readline `keyseq-timeout` (500ms default) so ESC is processed alone, not as meta prefix for Enter | `tmux.go` NudgeSession step 6 |
| Flock poll interval | 100ms | Cross-process flock retry loop interval | `flock_unix.go:16-42` |

### Nudge protocol steps (NudgeSession)

| Step | Action | Failure it prevents | Source |
|---|---|---|---|
| 0 | Dismiss Rewind mode (`isInRewindMode`) | Rewind UI consuming the message (gt-8el) | `tmux.go:1378` |
| 1 | Exit copy/scroll mode | Copy mode silently intercepting input | `tmux.go:1601+` |
| 2 | Sanitize control chars | Shell metacharacter injection | `tmux.go:1350` |
| 3 | Send text in 512-byte chunks | tmux buffer limits on large messages | `tmux.go:1509-1535` |
| 4 | Adaptive delay | Message truncation under load (gt-0b5) | `tmux.go:1491` |
| 5 | Send Escape | Stuck in vim INSERT mode (skipped for Copilot CLI, hq-isz) | `tmux.go` step 5 |
| 6 | Wait 600ms | ESC+Enter within 500ms becomes M-Enter (no submit) | `tmux.go` step 6 |
| 6.5 | Post-Escape Rewind check | Escape itself triggers Rewind if buffered | `tmux.go` step 6.5 |
| 7 | `sendEnterVerified` | Enter not processed under load | `tmux.go:1434` |
| 8 | `WakePaneIfDetached` (SIGWINCH) | Claude Code ignores input in detached panes | `tmux.go:1327` |

### Liveness check constants

| Constant | Value | Source |
|---|---|---|
| `ZombieStatus` enum values | `SessionHealthy`, `SessionDead`, `AgentDead`, `AgentHung` | `tmux.go:2108-2120` |
| `CheckSessionHealth` | Checks session existence, agent process liveness, activity staleness | `tmux.go:2155` |
| `IsAgentAlive` | Uses `GT_PROCESS_NAMES` env var to identify expected processes in pane | `tmux.go:2644` |
| `DefaultReadyPromptPrefix` | `"❯ "` | `tmux.go:2825` |

### Environment variables consumed

| Var | Set by | Consumed by | Source |
|---|---|---|---|
| `GT_AGENT_READY` | `gt prime --hook` SessionStart | `WaitForCommand` fallback for wrapped agents | `tmux.go:196` |
| `GT_PROCESS_NAMES` | Agent session creation | `IsAgentAlive` process-tree verification | `tmux.go:2644` |
| `GT_TOWN_SOCKET` | Tmux keybindings | `SocketFromEnv` for subcommands spawned from bindings | `tmux.go:3687` |
| `GT_AGENT` | Session startup | `NudgeSession` step 5 — skip Escape for `copilot` | `tmux.go` step 5 |
| `TMUX` | Tmux server | `IsInsideTmux`, `CurrentSessionName`, `IsInSameSocket` | `tmux.go:3176`, `3703`, `148` |

## Notes / open questions

- `sessionNudgeLocks sync.Map` grows monotonically — no eviction when
  a session dies. Slow leak for long-running daemons.
- Empirical-timing constants (600 ms post-Escape, 100 ms flock poll,
  512-byte send-keys chunk, 30 s nudge lock timeout) are scattered
  across the file rather than collected into one config struct.
- `validateSessionName` forbids dots and colons. Third-party callers
  constructing session names directly can hit `ErrInvalidSessionName`
  unexpectedly; `session/names.go` only produces safe `<prefix>-<suffix>`
  patterns.
- `flock_windows.go` not read here; future pass to confirm it matches
  Unix semantics.
- Theme resolution quietly falls back to `AssignTheme(rigName)` for
  unknown roles — a typo silently gets a deterministic-random theme
  instead of an error.
