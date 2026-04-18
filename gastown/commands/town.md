---
title: gt town
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/town_cycle.go
  - /home/kimberly/repos/gastown/internal/cmd/cycle.go
tags: [command, ungrouped, tmux, session, cycle, navigation, town-level]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt town

Town-level tmux session navigation — `next`/`prev` to cycle between
Mayor and Deacon sessions on the town-level tmux socket.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/town_cycle.go:41-121`.

### Invocation

```
gt town next [--session <name>]
gt town prev [--session <name>]
```

`Use: "town"` (`town_cycle.go:42`). The parent command has no `RunE`
and a generic `Short: "Town-level operations"` / `Long: "Commands for
town-level operations including session cycling."`
(`town_cycle.go:43-45`). Today only two subcommands exist — both are
cycle navigation — but the name "Town-level operations" suggests room
for future growth.

### Subcommands

Registered in `init()` on `town_cycle.go:32-39`:

1. **`gt town next`** (`townNextCmd`, `town_cycle.go:47-58`) —
   direction `+1`. Delegates to `cycleTownSession(1, townCycleSession)`.
2. **`gt town prev`** (`townPrevCmd`, `town_cycle.go:60-71`) —
   direction `-1`. Delegates to `cycleTownSession(-1, townCycleSession)`.

### Flags

| flag | type | default | description |
|------|------|---------|-------------|
| `--session` | string | `""` | Override current session (used by tmux binding) |

Declared per-subcommand on `town_cycle.go:37-38`. Bound to
`townCycleSession` global.

### `cycleTownSession` (`town_cycle.go:76-95`)

Shared with [cycle](cycle.md) — `cycle.go:115-118` calls this same
function when it detects a town-level current session.

1. Resolve current session via `resolveCurrentSession(sessionOverride)`
   (defined in [cycle.go:156-161](cycle.md)). Returns `not in a tmux
   session` error on failure.
2. If the current session is not town-level
   (`!isTownLevelSession(currentSession)`), return nil silently —
   typing `gt town next` from a non-town pane is a no-op, not an error.
3. Find all running town-level sessions via
   `findRunningTownSessions()` (`town_cycle.go:98-120`).
4. Delegate to `cycleInGroup(direction, currentSession, sessions)`
   from [cycle.go:168-198](cycle.md) — the same shared helper used by
   the crew / rig-ops / town cycle paths.

### `isTownLevelSession` (`town_cycle.go:25-30`)

```go
func isTownLevelSession(sessionName string) bool {
    mayorSession := getMayorSessionName()   // "hq-mayor"
    deaconSession := getDeaconSessionName() // "hq-deacon"
    return sessionName == mayorSession || sessionName == deaconSession
}
```

The comment on `town_cycle.go:22-24` notes these sessions use the
`hq-` prefix so they can be identified by name alone — critical for
`tmux run-shell` which may execute from outside the workspace
directory and can't always `workspace.FindFromCwd()`.

### `getTownLevelSessions` (`town_cycle.go:15-19`)

Returns `[mayorSession, deaconSession]` — the two town-level
candidates. Used by both `findRunningTownSessions` here and the
branch in [cycle.go:112](cycle.md).

### `findRunningTownSessions` (`town_cycle.go:98-120`)

Lists all tmux sessions via `listTmuxSessions()` (defined in
[cycle.go:218-224](cycle.md)), then filters to those matching a
town-level candidate. Returns `[]string` of running town sessions.

## Relationship with `gt cycle`

`gt town next` and `gt cycle next` overlap but aren't equivalent:

- `gt cycle next` **auto-detects** the current session's type and
  cycles within the detected group (town / crew / rig-ops). On a
  non-town pane it cycles a different group.
- `gt town next` **forces town-level semantics** regardless of the
  current session. On a non-town pane it silently no-ops (returns nil
  with no output).

Both share `cycleTownSession` and `cycleInGroup`. The duplication
exists so that keybindings can be explicit: `C-b n` in a mayor pane
calls `gt cycle next` (which happens to dispatch to `cycleTownSession`
because the session is town-level), but scripts or custom bindings can
call `gt town next` directly when they already know the caller is
town-level.

## Related commands

- [cycle](cycle.md) — the sibling that this command shares
  `cycleTownSession`, `cycleInGroup`, `listTmuxSessions`,
  `resolveCurrentSession`, and `townCycleSession` semantics with.
  The `[cycle.go]` file is the canonical home for the shared helpers;
  `town_cycle.go` is the town-specific entry point.
- [mayor](mayor.md), [deacon](deacon.md) — the two town-level
  sessions this command cycles between.
- [up](up.md), [down](down.md), [boot](boot.md) — lifecycle commands
  that create the sessions this one navigates between.
- [internal/tmux package](../packages/tmux.md) — `listTmuxSessions` / `switch-client` / `resolveCurrentSession` bottom out in this package; the `hq-` prefix convention interacts with the package-level `defaultSocket` discipline
- [status-line](status-line.md) — renders the tmux status right of the
  same mayor/deacon panes.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Town-level session cycling. Same pattern as `cycle.go` but across town-level sessions (mayor, deacon). All errors propagated.

## Notes / open questions

- **Sparse surface area.** Only two subcommands today, and both are
  cycle navigation. "Town-level operations" is a wider naming
  convention that suggests planned expansion (e.g. `gt town status`,
  `gt town shutdown`). Worth watching for growth.
- **Hard-coded `hq-` prefix.** `isTownLevelSession` depends on the
  Mayor/Deacon sessions being named `hq-mayor` / `hq-deacon`. If a
  future multi-town setup wanted to differentiate, the prefix would
  need plumbing.
- **No `--client` flag** — unlike [cycle](cycle.md), this command
  doesn't accept a `--client` TTY override. Town-level cycle works
  fine because `cycleInGroup` reads `cycleClientTarget` from the
  package global — but anyone invoking `gt town next` from a tmux
  run-shell would need to set `cycleClientTarget` via a different path
  (or not — the call site matters). Worth verifying the town-level
  `C-b n` binding actually works from run-shell.
- **Silent no-op on non-town panes.** The "return nil if not
  town-level" branch (`town_cycle.go:85-87`) means users who mistype
  `gt town next` in a crew pane get no feedback. For a keybinding
  this is correct; for a script it could be confusing.
