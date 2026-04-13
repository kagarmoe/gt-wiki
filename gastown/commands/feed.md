---
title: gt feed
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/feed.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, tui, events, activity-feed, bubbletea, beads-exempt]
---

# gt feed

Launches an interactive Bubble Tea TUI (default) or plain-text stream
that shows Gas Town events: agent lifecycle, beads activity, convoy
status, and merge queue events. Can also open as a dedicated tmux
window.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/feed.go:45-108`)
**Beads-exempt:** **yes** — `feed` is in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:62`; feed must
work even without `bd` because the `bd activity` source is marked
optional (`feed.go:243-246`).
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/feed.go:45-383`.

### Invocation

```
gt feed [-f] [--no-follow] [-n N] [--since DUR] [--mol ID] [--type T]
         [--rig NAME] [-w] [--plain] [-p]
```

Single terminal command (no subcommands). Runs `runFeed`
(`feed.go:110-160`).

### Behavior

`runFeed` (`feed.go:110-160`) dispatches to three modes:

1. **Tmux window mode** (`--window`/`-w`, `feed.go:121-127`): calls
   `runFeedInWindow` (`feed.go:291-352`), which verifies we're inside
   tmux (`tmux.IsInsideTmux`), gets the current session name, builds a
   `gt feed --plain --follow` command, and creates (or switches to) a
   window named `feed` in that session.
2. **TUI mode** (default when stdout is a TTY and `--plain` is not set,
   `feed.go:130-155`): calls `runFeedTUI` (`feed.go:233-288`).
3. **Plain mode** (non-TTY or `--plain`, `feed.go:159`): calls
   `runFeedDirect` (`feed.go:209-230`), which reads [`.events.jsonl`](../packages/events.md)
   directly via `feed.PrintGtEvents`.

### TUI mode

`runFeedTUI` (`feed.go:233-288`):

1. Resolves town root via `workspace.FindFromCwdOrError`
   (`feed.go:235-238`).
2. Aggregates event sources (each optional — failures don't abort):
   - `feed.NewBdActivitySource(workDir)` — `bd activity` stream
     (`feed.go:243-246`)
   - `feed.NewMQEventSourceFromWorkDir(workDir)` — merge queue events
     (`feed.go:249-252`)
   - `feed.NewGtEventsSource(townRoot)` — `.events.jsonl` stream
     (`feed.go:255-258`)
3. Errors out only if no sources were created (`feed.go:260-262`).
4. Wraps the sources in `feed.NewMultiSource` and wires the combined
   channel into a `feed.Model` (`feed.go:265-280`). Uses
   `feed.NewModelWithProblemsView(bd)` if `--problems` is set, otherwise
   `feed.NewModel(bd)`.
5. Runs the Bubble Tea program with `tea.WithAltScreen()`
   (`feed.go:282-286`).

The `--rig NAME` flag in TUI mode (`feed.go:138-154`) rewrites
`workDir` to `<townRoot>/<rig>/mayor/rig` or `<townRoot>/<rig>` —
whichever has a `.beads` directory. Errors out if the rig is not
found.

### Plain mode

`runFeedDirect` (`feed.go:209-230`): calls `feed.PrintGtEvents(townRoot, opts)`
with a `feed.PrintOptions` populated from the flags. Follow defaults
to true in a TTY and false in a pipe (`feed.go:215-218`).

### Window mode

`runFeedInWindow` (`feed.go:291-352`): requires `tmux.IsInsideTmux`,
resolves the current session name via `getCurrentTmuxSession` (defined
in `handoff.go` per the cross-reference comment at `feed.go:354-355`),
then either creates a new `feed` window or selects an existing one.
The command it runs inside the window is
`cd <workDir> && <gtPath> feed --plain --follow [filtered-args]`
(`feed.go:312-329`). `getCurrentTmuxSession` is the same helper
`handoff` uses — cross-reference only, no call chain duplication here.

### Long description event symbols

Documented at `feed.go:75-97`:

- `+` created/bonded, `→` in_progress, `✓` completed, `✗` failed,
  `⊘` deleted
- Emoji: `🦉` patrol_started, `⚡` polecat_nudged, `🎯` sling,
  `🤝` handoff
- Problems view: `🔥` GUPP violation, `⚠` stalled, `●` working,
  `○` idle, `💀` zombie
- MQ events: `⚙` merge_started, `✓` merged, `✗` merge_failed,
  `⊘` merge_skipped

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`feed.go:30-43`):

| Flag                  | Type   | Default | Purpose                                                        |
|-----------------------|--------|---------|----------------------------------------------------------------|
| `--follow` / `-f`     | bool   | `false` | Stream events in real-time (auto-on in TTY)                    |
| `--no-follow`         | bool   | `false` | Show events once and exit                                      |
| `--limit` / `-n`      | int    | `100`   | Maximum number of events                                       |
| `--since`             | string | `""`    | Events since duration (e.g. `5m`, `1h`, `30s`)                 |
| `--mol`               | string | `""`    | Filter by molecule/issue ID prefix                             |
| `--type`              | string | `""`    | Filter by event type (create, update, delete, comment)         |
| `--rig`               | string | `""`    | Filter events by rig name                                      |
| `--window` / `-w`     | bool   | `false` | Open in dedicated tmux window named `feed`                     |
| `--plain`             | bool   | `false` | Plain text output (bypasses the TUI)                           |
| `--problems` / `-p`   | bool   | `false` | Start in problems view (stuck agents)                          |

## Related

- [activity](activity.md) — writes to `.events.jsonl`, which is one of
  the streams feed reads via `feed.NewGtEventsSource`.
- [audit](audit.md) — historical reader across the same
  `.events.jsonl` file, plus git / beads / townlog.
- [log](log.md) — town log reader; feed's `BdActivitySource` and GT
  events source are distinct streams from `logs/town.log`.
- [dashboard](dashboard.md) — the web-UI equivalent of the feed TUI.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  the beads-exempt list.
- [README.md](README.md) — command tree index.

## Notes / open questions

- `internal/tui/feed` package (imported at `feed.go:12`) is where the
  Bubble Tea model, event-source abstraction, and multi-source
  multiplexing live. Worth a package page.
- `getCurrentTmuxSession` is defined in `handoff.go` and reused here;
  `windowExists`, `createWindow`, `selectWindow` (`feed.go:356-382`)
  shell out via `tmux.BuildCommand` rather than the `tmux.Tmux`
  method set — comments note "direct exec for simplicity."
- When `--rig` is passed in plain mode (`feed.go:158-159`), it's
  forwarded as a filter via `PrintOptions.Rig` rather than rewriting
  the workspace root like TUI mode.
- `bd activity` is an optional source — the code treats its absence as
  non-fatal, which is why `feed` can be beads-exempt.
