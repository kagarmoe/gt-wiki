---
title: gt session
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/session.go
tags: [command, agents, session, tmux, polecat-session, lifecycle]
---

# gt session

Manage tmux sessions for polecats. Thin wrapper over
`polecat.SessionManager` that starts, stops, attaches, lists,
captures, and injects into polecat-owned tmux sessions.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`session.go:40`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no
**Alias:** `gt sess`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/session.go`
(707 lines). Registration at `session.go:171-207` wires flags
and attaches nine subcommands to `sessionCmd`, then registers
the parent on `rootCmd`.

The parent `sessionCmd` (`session.go:37-50`) uses
`RunE: requireSubcommand`. Note the TIP in the Long help at
`session.go:48-49`: `"To send messages to a running session, use
'gt nudge' (not 'session inject'). The nudge command uses
reliable delivery that works correctly with Claude Code."`

### Invocation

```
gt session start    <rig>/<polecat> [--issue <id>]
gt session stop     <rig>/<polecat> [-f|--force]
gt session at       <rig>/<polecat>            (alias: attach)
gt session list                     [--rig <r>] [--json]
gt session capture  <rig>/<polecat> [count]    [-n <lines>]
gt session inject   <rig>/<polecat> [-m <msg>] [-f <file>]
gt session restart  <rig>/<polecat> [-f|--force]
gt session status   <rig>/<polecat> [--json]
gt session check    [rig]
```

The `<rig>/<polecat>` argument is a single positional with a
slash separator. If no slash is present, `parseAddress`
(`session.go:211-229`) tries to infer the rig from cwd via
`inferRigFromCwd` — so `gt session at Toast` works when you are
already inside `gastown/polecats/Toast/`.

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `start`   | `sessionStartCmd`   | `session.go:52-65`   | `runSessionStart`   (`session.go:244-290`) |
| `stop`    | `sessionStopCmd`    | `session.go:67-76`   | `runSessionStop`    (`session.go:292-326`) |
| `at` (alias `attach`) | `sessionAtCmd` | `session.go:78-87` | `runSessionAttach` (`session.go:328-341`) |
| `list`    | `sessionListCmd`    | `session.go:89-96`   | `runSessionList`    (`session.go:351-428`) |
| `capture` | `sessionCaptureCmd` | `session.go:98-111`  | `runSessionCapture` (`session.go:430-461`) |
| `inject`  | `sessionInjectCmd`  | `session.go:113-130` | `runSessionInject`  (`session.go:463-495`) |
| `restart` | `sessionRestartCmd` | `session.go:132-141` | `runSessionRestart` (`session.go:497-548`) |
| `status`  | `sessionStatusCmd`  | `session.go:143-151` | `runSessionStatus`  (`session.go:550-597`) |
| `check`   | `sessionCheckCmd`   | `session.go:153-169` | `runSessionCheck`   (`session.go:617-707`) |

### `start`

`runSessionStart` (`session.go:244-290`). Parses the address,
gets a session manager, then **verifies the polecat name exists
in the rig's polecat list** (`session.go:256-267`). On miss, uses
`suggest.FindSimilar(polecatName, r.Polecats, 3)` and
`suggest.FormatSuggestion` to produce a typo-correction error
with a `gt polecat identity add` hint.

Options are minimal: just `Issue` from `--issue`. Calls
`polecatMgr.Start(polecatName, opts)`. On success, prints the
attach hint and logs a `townlog.EventWake` with the agent
address and issue (`session.go:283-287`).

### `stop`

`runSessionStop` (`session.go:292-326`). `--force` / `-f` skips
graceful shutdown. Calls `polecatMgr.Stop(polecatName, force)`.
Logs a `townlog.EventKill` with reason `"gt session stop"` or
`"gt session stop --force"`. No error handling branches for
`ErrNotRunning` — the session manager owns that.

### `at` (attach)

`runSessionAttach` (`session.go:328-341`). Calls
`polecatMgr.Attach(polecatName)`, which "replaces the process"
(comment at `session.go:339`) via exec. No auto-start — if the
session is not running, this errors out.

### `list`

`runSessionList` (`session.go:351-428`). Loads rigs config from
`<town>/mayor/rigs.json`, tolerating a missing file with an empty
map. Uses `rig.NewManager(...).DiscoverRigs()` to enumerate
rigs. Applies `--rig` filter client-side. For each rig, creates a
`polecat.NewSessionManager(t, r)` and calls `List()`, collecting
`SessionListItem`s:

```go
type SessionListItem struct {
    Rig       string `json:"rig"`
    Polecat   string `json:"polecat"`
    SessionID string `json:"session_id"`
    Running   bool   `json:"running"`
}
```

Human output groups by `●`/`○` running indicator, `rig/polecat`
name, and a dimmed session ID.

### `capture`

`runSessionCapture` (`session.go:430-461`). Takes an optional
positional `[count]` OR a `-n / --lines` flag. Positional wins if
present (`session.go:442-452`), with explicit positive-integer
validation. Calls `polecatMgr.Capture(polecatName, lines)` and
prints the raw output. Default is 100 lines.

### `inject`

`runSessionInject` (`session.go:463-495`). Reads the message
from `-m` or `-f` (file). Errors if both empty. Calls
`polecatMgr.Inject(polecatName, message)` — low-level
`send-keys` path.

The Long help at `session.go:116-127` is blunt:

> NOTE: For sending messages to Claude sessions, use 'gt nudge'
> instead. It uses reliable delivery (literal mode + timing)
> that works correctly with Claude Code's input handling.
>
> This command is a low-level primitive for file-based injection
> or cases where you need raw tmux send-keys behavior.

### `restart`

`runSessionRestart` (`session.go:497-548`). Checks `IsRunning`
first; if running, stops (with `--force` honored) and then
waits up to **2 seconds** (`10 × 200ms`) for the session to
fully terminate before starting a new one
(`session.go:528-534`):

> Without this, Start may fail or create a duplicate if the old
> session hasn't been cleaned up by tmux yet.

Then `polecatMgr.Start(polecatName, opts)` with empty options
(no issue override on restart).

### `status`

`runSessionStatus` (`session.go:550-597`). Gets
`polecatMgr.Status(polecatName)`, which returns running state,
session ID, attached state, and timestamps. JSON output is
whatever the session manager returns (no wrapper struct).
Human output shows state, session ID, attached yes/no, created
timestamp + `formatDuration`-computed uptime, and a final
"Attach with: ..." hint.

`formatDuration` (`session.go:600-615`) handles four ranges:
`<1m` → `Ns`, `<1h` → `Nm Ns`, `<24h` → `Nh Nm`, `≥24h` →
`Nd Nh Nm`.

### `check`

`runSessionCheck` (`session.go:617-707`). Rig-scoped health
check. Walks `<rig>/polecats/` on disk directly (not via the
rig manager), filters out dot-prefixed names, computes the
expected session name via
`session.PolecatSessionName(session.PrefixFor(r.Name), polecatName)`,
and asks tmux `HasSession`.

Per-polecat status lines are printed as `✓ session alive`,
`✗ session not running`, or `⚠ error checking session`. A
summary line at the end reports `N checked, M healthy, K not
running`. If any crashed, appends a tip
`"To restart crashed polecats: gt session restart
<rig>/<polecat>"`.

Note: this command does NOT use the session manager — it
reimplements the polecat enumeration via `os.ReadDir` (see
`session.go:662-675`). The comment at
`session.go:662` says "Rig might not have polecats" to tolerate
missing directories.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--issue <id>`   | string | `""` | `start` | `session.go:173` |
| `--force` / `-f` | bool | `false` | `stop`, `restart` | `session.go:176,190` |
| `--rig <name>`   | string | `""` | `list` | `session.go:179` |
| `--json`         | bool | `false` | `list`, `status` | `session.go:180,193` |
| `--lines <n>` / `-n` | int | `100` | `capture` | `session.go:183` |
| `--message <s>` / `-m` | string | `""` | `inject` | `session.go:186` |
| `--file <path>` / `-f` | string | `""` | `inject` | `session.go:187` |

`--force` / `-f` is shared between stop and restart via the
`sessionForce` global (`session.go:28`); `-f` is also used for
`inject --file`, but on different subcommands so the flag set
names don't collide.

### Shared helpers

- `parseAddress(addr)` at `session.go:211-229` — `rig/polecat`
  splitter with cwd-inference fallback. Used by every subcommand
  that takes a polecat address. Also used by
  [polecat.md](polecat.md) subcommands.
- `getSessionManager(rigName)` at `session.go:232-242` — wraps
  `getRig` and `polecat.NewSessionManager`.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [polecat.md](polecat.md) — polecat identity and lifecycle
  management. `gt session` is the low-level tmux layer under
  `gt polecat`.
- [nudge.md](nudge.md) — the preferred message-injection path.
  `session inject` is explicitly deprecated in favor of nudge
  per the Long help at `session.go:48-49,116-121`.
- [seance.md](seance.md) — higher-level "resume a dead session"
  helper; uses session metadata `gt session status` exposes.
- [resume.md](resume.md) — another resume path.
- [prime.md](prime.md) — `gt prime` runs inside a session once
  it has been started; the session is started before prime
  runs.
- [handoff.md](handoff.md) — polecat-side session handoff; ends
  up killing and restarting the session through this subsystem.
- [done.md](done.md) — `gt done` ends polecat work and triggers
  sandbox cleanup; the session may be stopped by the Witness
  after.
- [witness.md](witness.md) — the per-rig Witness watches polecat
  sessions; `gt session check` is a manual version of what the
  Witness patrol does.
- [deacon.md](deacon.md) — `gt deacon health-check` targets
  polecat sessions addressed the same way.
- [hook.md](hook.md) — slung work lives on polecats and is
  inspected inside their sessions.
- [status.md](status.md) — town-level status includes session
  counts.
- [agents.md](agents.md) — `gt agents list` enumerates the
  polecats whose sessions this command operates on.

## Notes / open questions

- **Both `--lines` flag and positional `count`.** The positional
  overrides the flag if present. Surprising if a user writes
  `gt session capture rig/cat -n 50 200` expecting 50 lines.
- **`gt session check` bypasses rig/polecat manager entirely.**
  It does its own `os.ReadDir` on `<rig>/polecats/`. This means
  polecats without an identity bead but with a directory will
  still be checked, and polecats with an identity bead but no
  directory won't. Divergent view from `gt polecat list`.
- **`gt session attach` does NOT auto-start.** Unlike
  `gt witness attach` and `gt deacon attach`, this errors if the
  session is missing. Inconsistent with sibling agent
  commands.
- **`parseAddress` is permissive.** It accepts a bare polecat
  name and infers the rig from cwd, but only if `inferRigFromCwd`
  succeeds. Errors in that helper silently become
  `"invalid address format"` — the real cause is hidden.
- **No `gt session cycle`** — handoff/cycle is in
  `gt handoff` / `gt polecat` space, not here.
- **`gt sess` alias** is the shortest Agents-group alias; other
  commands use three-letter (`may`, `dea`, `ref`) or full
  names.
- **`townlog.EventWake` / `EventKill`** — this command file is
  one of the few producers of those event types. Worth a
  `townlog` package page eventually.
