---
title: internal/style
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/style/style.go
  - /home/kimberly/repos/gastown/internal/style/table.go
tags: [package, platform-service, ui, styling, lipgloss, table]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# internal/style

Thin lipgloss-based styling layer on top of [`internal/ui`](ui.md). Exposes
named `Success`/`Warning`/`Error`/`Info`/`Dim`/`Bold` styles, the
corresponding icon prefixes, a `PrintWarning` helper that always goes to
stderr, and a small `Table` renderer that understands ANSI-aware width
padding.

**Also known as:** "the CLI styling layer", "lipgloss wrapper".

**Go package path:** `github.com/steveyegge/gastown/internal/style`
**File count:** 2 go files, 1 test file
**Imports (notable):** `github.com/charmbracelet/lipgloss`, stdlib (`fmt`,
`os`, `regexp`, `strings`), and [`internal/ui`](ui.md) for the Ayu color
palette (`ColorPass`, `ColorWarn`, `ColorFail`, `ColorAccent`,
`ColorMuted`) and icon constants.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:17`) and roughly every
command that prints a status line, renders a table, or emits warnings —
especially the health-reporting commands like `gt doctor`, `gt stale`,
`gt mayor status`, and the listing commands (`gt rigs`, `gt polecats`,
etc.).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/style/style.go` and
`table.go`.

### Public API — named styles and prefixes

All defined as package-level `var` in `style.go:13-52`, so callers grab
them directly without a constructor:

| Name            | Source line(s)             | Colour / behaviour                                       |
|-----------------|----------------------------|----------------------------------------------------------|
| `Success`       | `style.go:15-17`           | Green (`ui.ColorPass`), bold                             |
| `Warning`       | `style.go:20-22`           | Yellow (`ui.ColorWarn`), bold                            |
| `Error`         | `style.go:25-27`           | Red (`ui.ColorFail`), bold                               |
| `Info`          | `style.go:30-31`           | Blue (`ui.ColorAccent`)                                  |
| `Dim`           | `style.go:34-35`           | Gray (`ui.ColorMuted`)                                   |
| `Bold`          | `style.go:38-39`           | Bold (no colour)                                         |
| `SuccessPrefix` | `style.go:42`              | `Success.Render(ui.IconPass)` — green `✓`                |
| `WarningPrefix` | `style.go:45`              | `Warning.Render(ui.IconWarn)` — yellow `⚠`               |
| `ErrorPrefix`   | `style.go:48`              | `Error.Render(ui.IconFail)` — red `✖`                    |
| `ArrowPrefix`   | `style.go:51`              | `Info.Render("→")` — blue action arrow                    |

### Public API — functions

- `style.PrintWarning(format string, args ...interface{})` —
  `Fprintf`-style helper that writes to **stderr**
  (`style.go:57-60`). The comment explicitly calls out the rationale:
  "Writes to stderr so warnings never contaminate structured (JSON)
  output on stdout." Commands that pipe their stdout into `jq` or another
  structured consumer can still surface warnings without breaking the
  pipe.

### Public API — Table rendering (`table.go`)

- `style.Column` struct — `Name string`, `Width int`, `Align Alignment`,
  `Style lipgloss.Style` (`table.go:12-18`).
- `style.Alignment` enum — `AlignLeft`, `AlignRight`, `AlignCenter`
  (`table.go:21-27`).
- `style.Table` struct — holds columns, rows, header-separator flag,
  indent prefix, and header style (`table.go:29-35`).
- `style.NewTable(columns ...Column) *Table` (`table.go:37-45`) —
  constructor. Defaults: 2-space indent, header separator enabled,
  `Bold` header style.
- `(*Table).SetIndent(s)`, `SetHeaderSeparator(bool)`, `AddRow(values
  ...string) *Table` — chainable setters (`table.go:48-67`). `AddRow`
  pads short rows with empty strings so callers don't have to match
  column count exactly.
- `(*Table).Render() string` (`table.go:69-128`) — produces the final
  string. Truncates long values with an ellipsis (`...`), applies
  per-column style when set, and uses `pad` to align within each column.
  Column separator is a single space; header separator is a row of
  `─` characters in `Dim` style.
- `pad(styledText, plainText string, width int, align Alignment) string`
  (`table.go:131-150`) — ANSI-aware padding: operates on `plainText`
  length (after `stripAnsi`) so lipgloss escape sequences don't inflate
  column widths.
- `stripAnsi(s string) string` (`table.go:155-158`) — strips CSI escape
  sequences using the package-level regex `ansiRegex`
  (`table.go:153`).

### Internals / Notable implementation

- `style.go` depends on `ui` specifically for colours and icons; it does
  **not** gate itself on `ui.ShouldUseColor()`. That gating happens
  inside lipgloss (via the profile set in `ui.init()` at `ui/styles.go:15-23`).
  If the terminal isn't colour-capable, the renders collapse to plain
  text automatically.
- The table renderer's `pad` function walks ANSI codes by doing
  truncation on the plain string (`plainLen := len(plainText)`,
  `table.go:133`) but emits the pre-styled `styledText` in the output —
  i.e., lipgloss styling survives the padding math.
- `Column.Style.Value() != ""` (`table.go:116`) is how the code
  detects an unset column style without a nil-pointer trap — lipgloss
  styles are value types, and an empty style's `.Value()` is the empty
  string.

### Usage pattern

```go
// Quick status line:
fmt.Println(style.SuccessPrefix, "rig up")

// stderr warning without corrupting JSON stdout:
style.PrintWarning("dolt ping latency %v", d)

// Simple table:
tbl := style.NewTable(
    style.Column{Name: "NAME", Width: 20},
    style.Column{Name: "STATE", Width: 12, Style: style.Success},
)
tbl.AddRow("woodhouse", "running")
fmt.Print(tbl.Render())
```

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; imports `style` from its root
  command.
- [internal/ui](ui.md) — colour palette and icon source. `style` is a
  thin semantic layer over `ui`'s Ayu colours.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `style.Table` is a separate, simpler renderer from the one
  `lipgloss/table` offers (not used here). Keeping this in-tree avoids a
  second lipgloss sub-dependency but means two table implementations
  exist across the codebase (the other being whichever command pages
  reach directly for lipgloss or charmbracelet bubbles).
- There is no `PrintSuccess`/`PrintError` analogue of `PrintWarning`.
  Commands that need those emit `fmt.Println(style.SuccessPrefix, ...)`
  inline instead.
