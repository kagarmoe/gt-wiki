---
title: gt cycle
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/cycle.go
  - /home/kimberly/repos/gastown/internal/cmd/town_cycle.go
tags: [command, ungrouped, tmux, session, cycle, navigation]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt cycle

Cycle the current tmux client between related sessions (town-level,
crew, or rig-ops group), driven by `C-b n` / `C-b p` keybindings.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/cycle.go:34-225`.

### Invocation

```
gt cycle next [--session <name>] [--client <tty>]
gt cycle prev [--session <name>] [--client <tty>]
```

`Use: "cycle"` (`cycle.go:35`). The parent command has no `RunE` — it is
a pure subcommand group. Both subcommands share the `--session` and
`--client` string flags declared on `cycle.go:28-31`.

### Subcommands

Registered in `init()` on `cycle.go:23-32`:

1. **`gt cycle next`** (`cycleNextCmd`, `cycle.go:51-66`) — direction `+1`.
2. **`gt cycle prev`** (`cyclePrevCmd`, `cycle.go:68-83`) — direction `-1`.

Both delegate to `cycleToSession(direction, cycleSession, cycleClient)`
(`cycle.go:63-64`, `cycle.go:80-81`).

### Flags

Declared per-subcommand on `cycle.go:28-31`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--session` | string | `""` | Override current session (used by tmux binding via `#{session_name}`) |
| `--client` | string | `""` | Target client TTY (used by tmux binding via `#{client_tty}`) |

The flags exist because `run-shell` from tmux spawns a subprocess without
reliable session/client context, so the binding passes both explicitly.

### `cycleToSession` dispatch (`cycle.go:89-134`)

1. **Override default tmux socket** (`cycle.go:95-97`): call
   `tmux.SocketFromEnv()` and, if non-empty, `tmux.SetDefaultSocket(...)`.
   `persistentPreRunE` sets the socket to the town path hash, but `cycle`
   is invoked from `run-shell` which may be on a different socket
   (e.g., `default`). Without this override, `switch-client` silently
   fails because the sessions aren't on the town-named socket.
2. **Resolve current session** — use `sessionOverride` if non-empty,
   else `getCurrentTmuxSession()` (`cycle.go:99-106`). On failure, return
   nil (not in tmux — nothing to do).
3. **Store client TTY** into package-level `cycleClientTarget`
   (`cycle.go:109`) for later use by `cycleInGroup`.
4. **Branch by session type:**
   - **Town-level** (`cycle.go:112-119`): if the current session matches
     any `getTownLevelSessions()` entry (Mayor or Deacon), delegate to
     `cycleTownSession(direction, session)` — defined in
     `town_cycle.go:76-95` and shared with [town next / prev](town.md).
   - **Crew** (`cycle.go:122-124`): if
     `sessionpkg.ParseSessionName(...).Role == RoleCrew`, delegate to
     `cycleCrewSession(direction, session)` (body lives elsewhere).
   - **Rig ops** (`cycle.go:127-130`): if `parseRigOpsSession(...)`
     returns a non-empty rig name (witness, refinery, or polecat),
     delegate to `cycleRigOpsSession(direction, session, rig)`.
5. **Unknown session type** → return nil silently (`cycle.go:133`).

### `parseRigOpsSession` (`cycle.go:138-148`)

Parses a session name and returns the rig name if the session is any of
`RoleWitness`, `RoleRefinery`, or `RolePolecat`. Empty string otherwise.
This is what makes witness + refinery + all polecats cycle together as a
single per-rig group.

### `cycleInGroup` (`cycle.go:168-198`)

Shared across all three cycle variants (town, crew, rig-ops) — also
called by [town](town.md) via `town_cycle.go:94`:

1. Empty list → return nil.
2. `sort.Strings(sessions)` — stable order.
3. Find `currentSession` index; if not present, return nil.
4. `targetIdx = (currentIdx + direction + len) % len`. If the target
   equals the current index, return nil (only one session in the group).
5. Run `tmux switch-client [-c <client>] -t <target>`, where the `-c`
   flag is only passed if `cycleClientTarget` is non-empty
   (`cycle.go:192-196`). This is the "client TTY" fix for `run-shell`
   invocations that otherwise target the wrong client.

### `cycleRigOpsSession` (`cycle.go:201-215`)

Lists every tmux session via `listTmuxSessions()`, filters by
`parseRigOpsSession(s) == rig`, and hands the result to `cycleInGroup`.

### `listTmuxSessions` (`cycle.go:218-224`)

Runs `tmux list-sessions -F #{session_name}` and splits output by lines.

## Related commands

- [town](town.md) — its `next`/`prev` subcommands share
  `cycleTownSession` and `cycleInGroup` from this file
  (`town_cycle.go:76-95`). The two commands are twins with different
  entry points: `gt cycle` detects the session type automatically; `gt
  town` forces town-level cycling regardless of current session.
- [witness](witness.md), [refinery](refinery.md), [polecat](polecat.md)
  — all three share the same rig-ops cycle group.
- [crew](crew.md) — crew sessions share their own cycle group within a rig.
- [mayor](mayor.md), [deacon](deacon.md) — the two town-level sessions
  cycled by `cycleTownSession`.
- [boot](boot.md), [up](up.md), [down](down.md) — bring-up / tear-down
  lifecycle commands that create the sessions this one navigates between.
- [internal/tmux package](../packages/tmux.md) — `SocketFromEnv` / `SetDefaultSocket` / `NewTmux` + the underlying `switch-client` wrappers all live here; the socket override rationale at `cycle.go:95-97` pairs with the package-level `defaultSocket` design
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Session cycling via tmux operations. Resolves current session, finds cycle group, switches. All errors propagated. No state mutations beyond tmux session switching.

## Notes / open questions

- **Four cycle scopes.** Town / crew / rig-ops / unknown. The rig-ops
  group intentionally conflates `witness`, `refinery`, and `polecat`
  sessions — an agent in one can `C-b n` straight to the other two.
- **Silent failure modes.** Many branches return nil without error
  messages: not in tmux, unknown session type, only one session in
  group, current session not in detected list. This is deliberate for a
  keybinding — errors would fire on every `C-b n` from an unrelated
  pane.
- **Shared state via package globals.** `cycleClientTarget` is set by
  `cycleToSession` and read by `cycleInGroup`. Any future caller of
  `cycleInGroup` must be aware that it reads this global.
- **Socket override precedence.** `cycle.go:95-97` mutates package-level
  tmux state (`tmux.SetDefaultSocket`) for the lifetime of the process.
  This is fine for a one-shot `gt cycle` subprocess but would be a bug
  if `cycleToSession` were ever called in a long-lived daemon context.
