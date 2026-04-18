---
title: gt changelog
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/changelog.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, reporting, bd-wrapper]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt changelog

Aggregate closed beads across all rigs in the current town into a
human- or JSON-readable changelog, filtering out ephemeral/internal
beads.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`changelog.go:30`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/changelog.go:28-306`.

### Invocation

```
gt changelog [--today|--week|--since <YYYY-MM-DD>] [--rig <name>] [--json]
```

### Behavior

`runChangelog` (`changelog.go:76-96`) is a four-step pipeline:

1. **Find town root** via `workspace.FindFromCwdOrError()`
   (`changelog.go:77-80`).
2. **Determine cutoff time** via `changelogSinceTime()`
   (`changelog.go:99-121`):
   - `--since YYYY-MM-DD` parses the explicit date via
     `time.ParseInLocation("2006-01-02", ...)`.
   - `--today` uses 00:00 local today.
   - Default (no flag): current week starting at Monday 00:00 local
     (handles Sunday specifically — `weekday == 0` becomes 7, so
     Monday-start is computed correctly).
3. **Collect entries** via `collectChangelogEntries(townRoot, since)`
   (`changelog.go:124-165`):
   - Always includes the town root as a `hq` location.
   - If no `--rig` filter: loads `mayor/rigs.json` via
     `config.LoadRigsConfig` and for each rig that has a `.beads`
     directory, adds it as a location (`changelog.go:132-142`).
   - If `--rig` was given: validates that `<rig>/.beads` exists,
     otherwise errors out (`changelog.go:143-149`).
   - For each location, calls `fetchClosedBeads(dir, rig, since)`,
     collecting entries and tolerating per-location errors
     silently (`changelog.go:151-158`).
   - Sorts results descending by `ClosedAt` (`changelog.go:161-163`).
4. **Render** via `printChangelog` or `printChangelogJSON`
   (`changelog.go:92-95`).

### Fetch + filter (`fetchClosedBeads`)

`changelog.go:168-203`:

- Shells out to `bd list --status=closed --all --limit=0 --json` with
  `cmd.Dir = dir`.
- Decodes into `[]closedBead` (`changelog.go:66-74`: `ID`, `Title`,
  `IssueType`, `Ephemeral`, `ClosedAt`, `CloseReason`, `Labels`).
- For each bead:
  - Skips if `isInternalBead(b)` returns true (see below).
  - Parses `ClosedAt` as RFC3339; skips on parse failure.
  - Skips if `ClosedAt` is before the cutoff.
  - Emits a `ChangelogEntry` (`changelog.go:56-63`: `ID`, `Title`,
    `Type`, `Rig`, `ClosedAt`, `CloseReason`).

### `isInternalBead` filter (`changelog.go:206-223`)

Excludes from the changelog:

- `b.Ephemeral == true` (wisps).
- `b.IssueType == "event"`.
- Any bead whose lowercased title starts with `mol-`, `wisp-`, `plugin
  run:`, or `cost report`.

This is the "filter out noise" layer — the wisp/patrol/molecule-step
ephemera should show up in [compact](compact.md) territory, not here.

### Rendering (`printChangelog`, `changelog.go:225-273`)

- Groups entries by rig (preserving the order they first appeared in
  the sorted list).
- Prints a bold label + count per rig.
- Per entry: type icon (`🐛` bug, `✨` feature, `✓` task, `🏔` epic,
  `·` default), dim bead ID, dim `Jan 02` date, title
  (`changelog.go:246-265`).
- Footer: `Total: N issues closed across M rig(s)`.

The period header is computed by `formatPeriod` (`changelog.go:285-306`)
which distinguishes "Week of …", "Today", and "Since …" labels.

### JSON output

`printChangelogJSON` (`changelog.go:275-282`) marshals
`[]ChangelogEntry` with `json.MarshalIndent` and writes to stdout.

### Subcommands

None.

### Flags

Defined in `init()` at `changelog.go:46-53`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--today` | bool | `false` | Show today's completions |
| `--week` | bool | `false` | Show this week's completions |
| `--since` | string | `""` | Show completions since date (YYYY-MM-DD) |
| `--rig` | string | `""` | Filter by rig name |
| `--json` | bool | `false` | Output as JSON |

Note: `--week` is defined but never consulted in `changelogSinceTime()`
— the week-of-Monday logic is the default, so `--week` is currently
redundant. This is worth flagging.

### Related commands

- [compact](compact.md) — handles the wisp/molecule-step ephemeral
  beads that this command filters out.
- [bead](bead.md), [cat](cat.md), [close](close.md) — the bd-wrapper
  family. `bd close` is what populates the `closed` status this
  command counts.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Docs claim

### Source

- `/home/kimberly/repos/gastown/internal/cmd/changelog.go:32-42` —
  `changelogCmd.Long` including the Examples block.
- `/home/kimberly/repos/gastown/internal/cmd/changelog.go:49` —
  flag description on `--week`.

### Verbatim

> `Show a changelog of closed beads across all rigs in Gas Town.`
>
> `Filters out ephemeral/internal beads (wisps, patrols) to show only real work.`
>
> `Examples:`
> `  gt changelog            # This week's completed work (default)`
> `  gt changelog --today    # Today's completions`
> `  gt changelog --week     # This week's completions`
> `  gt changelog --since 2026-03-10  # Since a specific date`
> `  gt changelog --rig gastown       # One rig only`
> `  gt changelog --json              # JSON output`

> `changelogCmd.Flags().BoolVar(&changelogWeek, "week", false, "Show this week's completions")`

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### `--week` flag is defined but never consulted

- **Claim source:** `changelogCmd.Long` Examples block at
  `/home/kimberly/repos/gastown/internal/cmd/changelog.go:39` and the
  flag description at `changelog.go:49`.
- **Docs claim:** the Long-text Examples line `gt changelog --week
  # This week's completions` and the flag description `"Show this
  week's completions"` both present `--week` as a selectable option
  distinct from the default.
- **Code does:** `changelogWeek` is declared as a package-level bool
  (`changelog.go:22`) and bound to the flag at `changelog.go:49`,
  but `changelogSinceTime` (`changelog.go:99-121`) — the only
  function that computes the cutoff time — never reads
  `changelogWeek`. The resolution cascade is `--since` →
  `--today` → default-this-week-monday; the `--week` path is absent
  entirely. A caller who types `gt changelog --week` gets
  "this-week's completions" only because it coincides with the
  default. If the default ever changes (e.g., to "all time") or the
  caller combines `--week` with `--since`, the flag will silently
  fail to influence behavior. No error is surfaced.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — either wire `changelogWeek` into
  `changelogSinceTime` (the `--today` branch at `:109-112` is the
  template: add a symmetric `if changelogWeek { ...monday... }`
  branch before the default), or remove the flag registration at
  `changelog.go:49` and the Examples-block line at `changelog.go:39`
  together. Removing is consistent with the current "it's the
  default" behavior but loses the explicit UX affordance; wiring is
  consistent with the advertised help text.
- **Release position:** `in-release` — `changelogWeek` is declared
  and registered byte-identical at `v1.0.0:internal/cmd/changelog.go:22`
  and `:49`, and `changelogSinceTime` at `:99-121` never reads it at
  v1.0.0 either.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Per-rig beads query failures silently skipped:** `changelog.go:157-159` — if `fetchClosedBeads` returns an error for a rig location, the `continue` drops it with no warning. A rig's entire closed-bead history is invisible if its beads database is corrupted or unreachable. **Absent** — no indication to the user that results are incomplete.
- **ClosedAt parse errors silently skip beads:** `changelog.go:188-189` — if `time.Parse(time.RFC3339, b.ClosedAt)` fails, the bead is silently skipped with `continue`. **Absent** — beads with non-RFC3339 timestamps are invisible in the changelog.

## Notes / open questions

- **`--week` is a no-op** — see `## Drift` above (promoted from
  Phase 2 notes bullet 1).
- **`isInternalBead` hard-codes title prefixes.** `mol-`, `wisp-`,
  `plugin run:`, `cost report` are baked into the filter
  (`changelog.go:216-221`). A configuration-driven approach would be
  cleaner — but out of scope for a mapping pass.
- **Per-rig errors silently dropped.** `collectChangelogEntries`
  swallows errors with "Non-fatal: rig may have no beads db"
  (`changelog.go:154-156`). This is fine for the happy path but
  obscures real failures.
