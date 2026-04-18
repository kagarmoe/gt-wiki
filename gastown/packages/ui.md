---
title: internal/ui
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/ui/styles.go
  - /home/kimberly/repos/gastown/internal/ui/terminal.go
  - /home/kimberly/repos/gastown/internal/ui/pager.go
  - /home/kimberly/repos/gastown/internal/ui/markdown.go
tags: [package, platform-service, ui, colour, ayu, terminal, pager, markdown, no-color]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/ui

The terminal rendering substrate for all `gt` output: Ayu-theme colour
palette, semantic styles and icons for beads (status/priority/type),
terminal capability detection (`NO_COLOR`, `CLICOLOR`, `GT_THEME`,
`GT_AGENT_MODE`), a markdown-to-terminal renderer powered by glamour, and
a pager helper that shells out to `less` (or `$GT_PAGER` / `$PAGER`).

**Also known as:** "the TUI layer", "the Ayu palette".

**Go package path:** `github.com/steveyegge/gastown/internal/ui`
**File count:** 4 go files, 2 test files
**Imports (notable):** `github.com/charmbracelet/lipgloss`,
`github.com/charmbracelet/glamour`, `github.com/muesli/termenv`,
`golang.org/x/term`, stdlib.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:19`),
[`internal/style`](style.md) (consumes the colour palette and icons), and
virtually every command that renders output with semantic colouring —
bead listings (`gt bead`, `gt list`, `gt ready`, `gt done`, …),
dashboards, help text.

## What it actually does

Source files: `styles.go` (palette), `terminal.go` (capability detection),
`pager.go` (output paging), `markdown.go` (glamour rendering).

### Public API — colour palette and styles (`styles.go`)

Package `init()` (`styles.go:15-23`) configures the global lipgloss colour
profile once at startup: `termenv.Ascii` when colours are disabled (via
`ShouldUseColor()`), otherwise `termenv.TrueColor`. This is load-bearing
— every lipgloss render elsewhere in `gt` inherits this profile.

**Ayu theme adaptive colours** (`styles.go:39-112`) — each exposed as
`lipgloss.AdaptiveColor{Light, Dark}`:

- Core: `ColorPass` (green), `ColorWarn` (yellow), `ColorFail` (red),
  `ColorMuted` (gray), `ColorAccent` (blue).
- Workflow status: `ColorStatusOpen` (neutral), `ColorStatusInProgress`
  (yellow), `ColorStatusClosed` (dim), `ColorStatusBlocked` (red),
  `ColorStatusPinned` (purple), `ColorStatusHooked` (cyan).
- Priority: `ColorPriorityP0` (bright red + bold on style), `P1`
  (orange), `P2` (muted gold), `P3`/`P4` (neutral).
- Issue type: `ColorTypeBug` (red), `ColorTypeEpic` (purple),
  `ColorTypeFeature`/`Task`/`Chore` (neutral).
- `ColorID` — neutral (standard text) for issue IDs; styled via `IDStyle`
  at `styles.go:155`.

**Pre-built styles** (`styles.go:146-195`): `PassStyle`, `WarnStyle`,
`FailStyle`, `MutedStyle`, `AccentStyle`, `IDStyle`, plus per-status,
per-priority, and per-type style variables. `PriorityP0Style` is the only
one bolded (`styles.go:169`).

**Icons and separators** (`styles.go:199-233`):

- `IconPass="✓"`, `IconWarn="⚠"`, `IconFail="✖"`, `IconSkip="-"`,
  `IconInfo="ℹ"`, `IconFix="🔧"`.
- Issue status icons: `StatusIconOpen="○"`,
  `StatusIconInProgress="◐"`, `StatusIconBlocked="●"`,
  `StatusIconClosed="✓"`, `StatusIconDeferred="❄"`,
  `StatusIconPinned="📌"`.
- `PriorityIcon="●"`.
- Tree characters (`TreeChild`, `TreeLast`, `TreeIndent`) at
  `styles.go:222-227`.
- Separators: `SeparatorLight` (42 × `─`), `SeparatorHeavy` (42 × `═`).

**Render helpers** (`styles.go:237-420`):

- `RenderPass/Warn/Fail/Muted/Accent/Bold/Command/Category/Separator(s
  string) string` — call the corresponding style.
- `RenderPassIcon/WarnIcon/FailIcon/SkipIcon/InfoIcon/FixIcon()` — styled
  icons.
- `RenderID(id string) string`, `RenderStatus(status string) string`,
  `RenderStatusIcon(status string) string`, `RenderPriority(priority int)
  string`, `RenderPriorityCompact(priority int)`, `RenderType(issueType
  string) string`. These are the canonical source for rendering bead
  metadata — any command page that prints beads should go through these.

### Public API — terminal capability detection (`terminal.go`)

- `ui.ThemeMode` string type with constants `ThemeModeAuto`, `ThemeModeDark`,
  `ThemeModeLight` (`terminal.go:12-21`).
- `ui.InitTheme(configTheme string)` — called once at startup, resolves
  mode and caches the "dark vs light" determination
  (`terminal.go:31-34`).
- `ui.GetThemeMode() ThemeMode`, `ui.HasDarkBackground() bool` — lookups
  against the cached values.
- `ui.ApplyThemeMode()` — propagates the dark/light determination into
  lipgloss's `SetHasDarkBackground` (see `style.go:27-33`).
- `ui.IsTerminal() bool` — stdout-is-TTY check via `term.IsTerminal`.
- `ui.ShouldUseColor() bool` (`terminal.go:102-120`) — honours the
  `NO_COLOR` convention (`no-color.org`), `CLICOLOR=0`, `CLICOLOR_FORCE`,
  and defaults to TTY presence.
- `ui.ShouldUseEmoji() bool` (`terminal.go:123-132`) — adds `GT_NO_EMOJI`
  to the set of opt-outs, otherwise TTY-gated.
- `ui.IsAgentMode() bool` (`terminal.go:140-149`) — returns true when
  `GT_AGENT_MODE=1` **or** when `CLAUDE_CODE` is set in the environment
  (auto-detect for Claude Code subprocesses). Agent mode implies
  ultra-compact, machine-readable output optimised for LLM context
  windows.

**Theme resolution precedence** (`terminal.go:52-80`):

1. `GT_THEME` env var (dark / light / auto).
2. Config file value passed to `InitTheme`.
3. Default: auto (termenv background detection).

### Public API — pager (`pager.go`)

- `ui.PagerOptions` struct with single field `NoPager bool`
  (`pager.go:12-15`) — the `--no-pager` flag from commands maps directly
  onto this.
- `ui.ToPager(content string, opts PagerOptions) error`
  (`pager.go:69-106`) — pipes content to the pager **only if** it would
  actually overflow the terminal height; otherwise prints direct via
  `fmt.Print`. Pager is disabled when:
  - `opts.NoPager` is true,
  - `GT_NO_PAGER` env var is set (any value),
  - stdout is not a TTY.
  Pager command resolution (`getPagerCommand`, `pager.go:35-43`):
  `$GT_PAGER` → `$PAGER` → `"less"`.
  If `$LESS` is unset, the helper injects `LESS=-RFX` so `less`
  preserves ANSI colours (`-R`), auto-quits when content fits (`-F`),
  and doesn't clear the screen on exit (`-X`) — `pager.go:97-103`.

### Public API — markdown (`markdown.go`)

- `ui.RenderMarkdown(markdown string) string` — glamour-rendered
  terminal markdown (`markdown.go:12-39`). Returns the **raw** markdown
  unchanged when:
  1. `IsAgentMode()` is true (machines parse plain text better), or
  2. `ShouldUseColor()` is false, or
  3. glamour renderer construction fails (graceful degradation).
- `getTerminalWidth()` (`markdown.go:44-65`) — caps wrap width at 100
  characters regardless of terminal width, defaulting to 80 when not a
  TTY.

### Internals / Notable implementation

- The `init()` function in `styles.go` is the only place the lipgloss
  colour profile is set globally. Everything else reads from it.
- `IsAgentMode()` piggy-backs on Claude Code's `CLAUDE_CODE` env var,
  which means running `gt` inside a Claude Code subagent automatically
  produces agent-mode output — no opt-in needed on the command side.
- Some status/priority colours deliberately leave `Light` and `Dark`
  empty strings (e.g. `ColorStatusOpen`, `ColorPriorityP3`); this is
  lipgloss-idiomatic for "use standard text colour" and keeps neutral
  entries visually quiet next to coloured ones.

### Usage pattern

- **Startup (root command):** `ui.InitTheme(townSettings.CLITheme)` →
  `ui.ApplyThemeMode()` are called once during `persistentPreRun` so the
  whole process inherits the resolved profile.
- **Per-command rendering:** call `ui.RenderStatus`, `ui.RenderPriority`,
  `ui.RenderType`, etc., inside the command's `RunE` while building
  output. For help pages and long structured output, pipe the final
  string through `ui.ToPager(out, ui.PagerOptions{NoPager: noPagerFlag})`.
- **Markdown docs:** `gt` commands that read markdown from `docs/` or
  from bead descriptions render it via `ui.RenderMarkdown` for TTY
  consumers, falling through to raw text in agent mode.

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; imports `ui` from root.
- [internal/style](style.md) — thin wrapper that bundles semantic
  `Success/Warning/Error` styles on top of `ui`'s palette and icon set.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `getTerminalWidth()`'s 100-column cap is not configurable
  (`markdown.go:46-64`). Users on very wide terminals get hard-wrapped
  markdown, which is deliberate but may surprise some consumers.
- `ShouldUseColor` and `ShouldUseEmoji` don't share their caching story
  with `InitTheme`: they re-read env vars on every call. For hot paths
  this is fine (stdlib reads env from a cached map) but it means
  changing `NO_COLOR` mid-process would take effect on the next render.
- `IsAgentMode()` returns true whenever `CLAUDE_CODE` is present, even if
  `CLAUDE_CODE` is set to an unusual value by a user shim. There's no
  explicit `CLAUDE_CODE != ""` check beyond that.
