---
title: internal/tui
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/tui/convoy/
  - /home/kimberly/repos/gastown/internal/tui/feed/
tags: [package, tui, bubbletea, lipgloss, convoy, feed]
---

# internal/tui

The terminal UI family built on Charm's Bubble Tea / Bubbles / Lipgloss
stack. **`internal/tui/` has no Go files of its own** — the directory
exists only to group two sibling subpackages that each implement one
full-screen TUI:

- `internal/tui/convoy/` — the convoy-browsing TUI that renders a
  tree of convoys and the issues they track (behind
  [`gt convoy`](../commands/dashboard.md)-family surfaces).
- `internal/tui/feed/` — the activity-feed / problems-view TUI behind
  [`gt feed`](../commands/feed.md) and the panel-based dashboard.

This page covers both subpackages as one family, following the
wiki's "consolidate thin related packages" rule. For a sibling package
that renders non-interactive UI styling, see
[`internal/ui`](ui.md).

**Go package paths:**
`github.com/steveyegge/gastown/internal/tui/convoy`,
`github.com/steveyegge/gastown/internal/tui/feed`
**File counts:**
- `convoy/`: `keys.go` (73), `model.go` (391), `view.go` (161), plus
  one race test file.
- `feed/`: 14 go files, ~4,000 lines total including tests. Key files:
  `model.go` (856), `view.go` (558), `events.go` (658),
  `convoy.go` (485), `stuck.go` (345), `print_events.go` (182),
  `styles.go` (197), `keys.go` (150), `multi_source.go` (80),
  `mq_source.go` (48), `convoy_issues.go` (106).

**Imports (notable):** `github.com/charmbracelet/bubbletea`,
`github.com/charmbracelet/bubbles/key`,
`github.com/charmbracelet/bubbles/help`,
`github.com/charmbracelet/bubbles/viewport`,
`github.com/charmbracelet/lipgloss`. Internal:
[`internal/beads`](beads.md), [`internal/constants`](constants.md),
[`internal/tmux`](tmux.md), [`internal/util`](util.md).
**Imported by (notable):** the [`gt feed`](../commands/feed.md) and
convoy/dashboard command implementations in `cmd/`. These
subpackages are Bubble Tea models that are instantiated and run by the
command, not libraries that other packages consume.

## Subdirectory structure

```
internal/tui/              <-- 0 Go files
  convoy/                  <-- convoy TUI
    keys.go
    model.go
    view.go
    model_race_test.go
  feed/                    <-- feed / dashboard TUI
    convoy.go
    convoy_issues.go
    events.go
    keys.go
    model.go
    model_race_test.go
    mq_source.go
    multi_source.go
    print_events.go
    print_events_test.go
    stuck.go
    stuck_test.go
    styles.go
    view.go
```

Both subpackages follow the conventional Bubble Tea layout: `keys.go`
holds the `KeyMap` and default bindings, `model.go` holds the `Model`
struct plus its `Init`/`Update`/`View` methods, `view.go` holds the
Lipgloss render path, and (for `feed`) `styles.go` holds the style
vocabulary.

## internal/tui/convoy

Full-screen convoy browser. Shows a tree of **convoys** (grouped work
containers, identified by IDs like `hq-<slug>`), each with its
tracked issues inline, and supports expanding/collapsing.

### Types (`model.go:26-64`)

- `IssueItem` — `ID`, `Title`, `Status`.
- `ConvoyItem` — `ID`, `Title`, `Status`, `Issues []IssueItem`,
  `Progress` string (like `"2/5"`), `Expanded` bool.
- `Model` — the bubbletea model, holds `convoys []ConvoyItem`, a
  cursor index into the flattened view, the town beads directory
  path, a `KeyMap`, a `help.Model`, an error state, and other UI
  state.

### Convoy ID validation

`convoyIDPattern = regexp.MustCompile(\`^hq-[a-zA-Z0-9-]+$\`)`
(`model.go:23`) — convoy IDs must start with the `hq-` prefix. This
is the canonical shape; any page that refers to "convoy IDs" in the
wiki should use this regex as the authority.

### Key bindings (`keys.go`)

Vim-style with arrow-key aliases. `KeyMap` (`keys.go:6-16`) covers
Up/Down (k/j), PageUp/PageDown (ctrl-u/ctrl-d), Top/Bottom (g/G),
Toggle (enter/space for expand-collapse), Help (?), and Quit
(q/esc/ctrl+c). `DefaultKeyMap()` at `keys.go:19-58` is the canonical
binding set. `ShortHelp()` and `FullHelp()` at `keys.go:61-72` feed
the `bubbles/help` help pane.

### External commands

The convoy model shells out to other tools via `os/exec` (imported at
`model.go:8`) to fetch convoy and issue data rather than reading the
bead store directly. This keeps the TUI loosely coupled to the shape
of the underlying storage — it consumes whatever the tools print.

## internal/tui/feed

The activity-feed / dashboard / problems-view TUI. Much larger and more
layered than `convoy`: it's a panel-based layout with multiple view
modes, concurrent event sources, and a "stuck agents" detection pass.

### Panel and view model (`model.go:18-42`)

- `Panel` enum (`model.go:18-27`): `PanelTree`, `PanelConvoy`,
  `PanelFeed`, `PanelProblems`. Controls focus — which panel
  receives key events.
- `ViewMode` enum (`model.go:29-35`): `ViewActivity` (default
  activity-stream layout) and `ViewProblems` (problem-first layout).
  Toggling between them reshuffles the panel set without tearing
  down event sources.
- Layout constants (`model.go:36-41`): `treePanelPercent = 30`,
  `convoyPanelPercent = 25`, `maxEventHistory = 1000`. These are the
  only "magic numbers" in the layout, kept at the top of the file so
  they can be tuned without spelunking.

### Events and event sources

The feed TUI is built around a pluggable event-source abstraction:

- `Event` struct (`model.go:44-50`): `Time`, `Type` (create, update,
  complete, fail, delete), `Actor` (who did it,
  e.g. `"gastown/crew/joe"`), `Target`, `Message`, `Rig`.
- `EventSource` interface (not shown in the snippets above, defined
  in `events.go`): `Events() <-chan Event`, `Close() error`.
- `MultiSource` (`multi_source.go:8-80`) fans in N event sources into
  one combined channel. Each source runs in its own goroutine;
  `sync.WaitGroup` tracks them and closes the combined channel when
  all sources are drained. `Close` broadcasts via a `done` channel
  and collects the last error across sources.

Concrete event sources:

- `MQEventSource` (`mq_source.go`) is a **stub** — the comment at
  `mq_source.go:7-9` states "The mrqueue package has been removed, so
  this is now a no-op stub. MR events can be observed via beads
  activity instead." The source stays in the code so callers don't
  have to be rewired; it just produces no events. See Drift.
- `BeadsEventSource` and related sources (in `events.go`) are the
  real sources that read activity from the bead store.

### Stuck-agent detection (`stuck.go`)

`stuck.go` and its test file implement the "problems view" backend —
heuristics for detecting polecats that have fallen off a cliff (no
activity past the red threshold). This is the same conceptual space
as [`internal/activity`](activity.md) — that package provides the
color thresholds for the activity widget, and `feed/stuck.go` uses
comparable logic to decide what lands in the problems panel. The two
are not wired together in code today; it's possible they reuse
concepts rather than imports.

### Print events (non-TUI output, `print_events.go`)

`print_events.go` / `print_events_test.go` — the feed subpackage also
exposes a **non-interactive** print path for piping event streams to
stdout. This is what `gt feed` uses in non-TTY mode (and in tests).
It's not a separate package, but it's worth flagging because "tui"
conventionally implies interactive-only.

### Styling (`styles.go`)

All Lipgloss styles live in one file. The convoy subpackage has
styles inlined in `view.go` (`view.go:13-28` uses `titleStyle`,
`selectedStyle`, `convoyStyle`, `issueOpenStyle`, `issueClosedStyle`,
etc.) rather than factoring them out. Both approaches are legitimate
bubbletea patterns; the feed subpackage is simply larger.

## Docs claim

There are no package-level doc comments in either subpackage — every
`package convoy` / `package feed` declaration is comment-less. The
structure and names (`Model`, `KeyMap`, `View`) are self-documenting
to anyone familiar with Bubble Tea, but a reader coming from Go-only
backgrounds will need to recognize the Elm-architecture idioms to
make sense of the file layout. The one explicit doc comment in
`mq_source.go:7-9` flags the MQ source as a deprecated stub.

## Drift

- **`MQEventSource` is a zombie.** The source is kept as a no-op to
  avoid breaking call sites, but the `mrqueue` package it wrapped has
  been removed (`mq_source.go:7-9`). Any doc page still mentioning
  "MQ events in the feed" is describing a feature that no longer
  produces output. The suggested replacement is "beads activity
  instead." This is a real drift marker worth logging in the wiki's
  drift index.
- **`internal/tui/` empty parent.** The directory has zero Go files
  of its own, but the name suggests a package. A reader grepping for
  `package tui` will find nothing. This is consistent with
  `internal/scheduler/` (same pattern) — both use the parent
  directory as a namespace for future sibling packages.
- **`feed` subpackage size.** At 4,000+ lines across 14 files, `feed`
  is on the edge of being a page-worthy topic in its own right. If
  this page starts feeling like "convoy in three paragraphs and feed
  in twelve," split it into `tui/convoy` and `tui/feed` sibling
  pages rather than growing this one.

## Notes / open questions

- The convoy TUI uses `os/exec` to invoke external tools rather than
  linking against the bead store directly (`model.go:8`). This is a
  deliberate loose-coupling choice but has two side effects: (1) the
  TUI can't surface errors from the underlying query beyond what the
  tool prints, and (2) every refresh spawns a subprocess. Acceptable
  for an interactive TUI, but a pain to test.
- `ConvoyItem.Progress` is a pre-formatted string (`"2/5"`)
  (`model.go:38`), not a pair of ints. Callers that want to sort or
  color by completion percentage have to parse it back out.
- `Model.err` is a single error field, not a stack. If two errors
  happen in quick succession, the later one clobbers the earlier
  display.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [internal/ui](ui.md) — sibling terminal-UI package for
  non-interactive styling (prompts, spinners, colorized output).
- [gt feed](../commands/feed.md) — user-facing feed command; the
  primary caller of the `feed` subpackage.
- [gt dashboard](../commands/dashboard.md) — the larger dashboard
  experience that sits alongside these TUIs.
- [internal/activity](activity.md) — activity color thresholds
  conceptually related to `feed/stuck.go`.
- [internal/beads](beads.md) — underlying bead store consumed by
  feed's real event sources.
- [go-packages inventory](../inventory/go-packages.md).
