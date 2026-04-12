---
title: gt thanks
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/thanks.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, contributors, ritual, credits]
---

# gt thanks

Prints a hard-coded, sorted credits page of human contributors to the
gastown project. A ritual command — no workspace interaction, no
side effects.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/thanks.go:76-86`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/thanks.go:76-235`.

### Invocation

```
gt thanks
```

No flags, no subcommands. The cobra.Command uses `Run` (not `RunE`)
so it cannot fail (`thanks.go:83-85`).

### Behavior

`printThanksPage` (`thanks.go:108-150`):

1. Gets the full list of contributors sorted by commit count
   descending, name ascending as tiebreaker
   (`getContributorsSorted`, `thanks.go:89-105`).
2. Splits into `featured` (top 20) and `additional` (the rest)
   (`thanks.go:111-118`).
3. Computes a content width based on 4 columns, minimum 60 chars
   (`calculateColumnsWidth`, `thanks.go:153-170`).
4. Renders a double-bordered box via lipgloss containing a
   `THANK YOU!` title and "To all the humans who contributed to Gas
   Town" subtitle (`thanksBoxStyle`, `thanks.go:33-41, 128-135`).
5. Prints `Featured Contributors` section header, then the top 20 in
   4-column left-to-right layout (`printThanksColumns`,
   `thanks.go:173-204`). Names longer than 20 chars are truncated
   with `...` (`thanks.go:195-197`).
6. If there are additional contributors, prints a second
   `Additional Contributors` section as a comma-separated wrapped
   list (`printThanksWrappedList`, `thanks.go:207-235`).

### The contributor list

Hard-coded as a map on `thanks.go:46-74`:

```go
var gastownContributors = map[string]int{
    "Steve Yegge":               2056,
    "Mike Lady":                 19,
    "Olivier Debeuf De Rijcker": 13,
    "Danno Mayer":               11,
    "Dan Shapiro":               7,
    // ... 22 more ...
}
```

The comment at `thanks.go:43-45` states:

> Agent names (`gastown/*`, `beads/*`, lowercase single-word names)
> are excluded. Generated from: `git shortlog -sn --all` (then
> filtered for humans only)

So the map is a point-in-time snapshot, manually regenerated from
git history and filtered to remove agent-authored commits. It does
not query git at runtime.

### Styling

Lipgloss styles at `thanks.go:14-31`, all drawing from the `ui`
color package:

- `thanksTitleStyle` — bold, `ui.ColorWarn`
- `thanksSubtitleStyle` — `ui.ColorMuted`
- `thanksSectionStyle` — bold, `ui.ColorAccent`
- `thanksNameStyle` — `ui.ColorPass`
- `thanksDimStyle` — `ui.ColorMuted`
- `thanksBoxStyle(width)` — double border, muted foreground,
  padding `1,4`, centered

Note: `thanksDimStyle` is used inside `printThanksColumns` to pad
names *before* styling to avoid ANSI-width bugs in column alignment
(`thanks.go:198-200`).

### Subcommands

None (terminal command).

### Flags

None — `init` at `thanks.go:237-239` only registers the command on
`rootCmd`.

## Related

- [version](version.md) — another no-workspace, information-only
  command in the Diag group; thanks is purely decorative where
  version is operational.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  the Diag group registration.
- [README.md](README.md) — command tree index.

## Notes / open questions

- The `gastownContributors` map must be manually regenerated from
  `git shortlog -sn --all` when new human contributors land. There
  is no CI check for staleness and no tooling to regenerate it.
- Positioning `thanks` in `GroupDiag` ("Diagnostics") is
  surprising — it is decorative, not diagnostic. Grouping is set
  at `thanks.go:79`. Consider a `GroupRitual` or `GroupMeta` in
  future refactors.
- The command pipes directly to `stdout` and always exits zero,
  making it safe for shell aliases and motd-style integrations.
- Unlike `bd thanks` (referenced in the comment at `thanks.go:191`
  saying "matches bd thanks"), `gt thanks` does not pull contributor
  data from beads or git at runtime — the list is pinned in source.
