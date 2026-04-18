---
title: gt ready
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/ready.go
  - /home/kimberly/repos/gastown/internal/beads/
tags: [command, work, beads, ready-work, town-wide, priority]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt ready

Show all ready work across the town and all rigs — issues with no
blockers that can be worked immediately. Aggregates `bd ready`-style
queries from the town beads database plus each discovered rig's
beads, filters out formula scaffolds, wisps, and identity beads, and
prints sorted by priority.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`ready.go:28`)
**Polecat-safe:** no
**Beads-exempt:** no (runs `bd` via `beads.New(...).Ready()`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/ready.go:77-254`
for `runReady`; helpers through line 473.

### Invocation

```
gt ready [--json] [--rig <name>]
```

### Behavior (`runReady`, `ready.go:77-254`)

1. **Find town root** — `workspace.FindFromCwdOrError` (`:79`).
2. **Discover rigs** — load `config.LoadRigsConfig` from
   `constants.MayorRigsPath(townRoot)`; build a `rig.Manager` with
   the git wrapper and call `DiscoverRigs()` (`:85-97`).
3. **Filter rigs** — if `--rig <name>` was passed, narrow to that
   single rig or error if not found (`:100-112`).
4. **Parallel fetch** — a `sync.WaitGroup`+`sync.Mutex` fan-out
   (`:115-175`):
   - **Town source** (only when `--rig` is unset, `:120-145`):
     `beads.New(beads.GetTownBeadsPath(townRoot)).Ready()`.
   - **Each rig** (`:147-174`): `beads.New(r.BeadsPath()).Ready()`.
5. **Per-source filters** run after the `Ready()` call (`:134-142`,
   `:163-171`) in this order:
   - `filterFormulaScaffolds` (`:356-379`) — removes any issue whose
     ID matches a formula name from the `.beads/formulas/` directory,
     plus step scaffolds with IDs shaped `<formula-name>.<step-id>`.
     References `gt-579`.
   - `filterWisps` (`:461-473`) — queries
     `bd mol wisp list --json` (`getWispIDs`, `:385-408`) and drops
     those IDs. Defense-in-depth: `bd ready` is supposed to filter
     these, but this re-checks at the display layer.
   - `filterIdentityBeads` (`:418-457`) — drops agent beads
     (`beads.IsAgentBead`), issues with labels `gt:agent`/`gt:role`/
     `gt:rig`, IDs ending in `-role`, and IDs containing `-rig-`.
6. **Sort.** Sources: `"town"` first, then rig names alphabetically
   (`:178-188`). Within each source, by `Priority` (lower = higher
   priority, `:190-194`).
7. **Summary** — total, per-source count, P0–P4 counts (`:197-218`).
8. **Output** — JSON (`:235-239`) or the human formatter
   `printReadyHuman` (`:256-327`), which uses colored priority tags
   (P0/P1 = error, P2 = warning, else dim) and truncates titles
   over 60 chars.
9. **Error surfacing** — if *some* sources errored, print a warning
   with the failed source names (`:226-232`, `:246-251`). If *all*
   errored, return a hard error.

### Data types (`ready.go:53-75`)

```go
type ReadySource struct {
    Name   string         // "town" or rig name
    Issues []*beads.Issue
    Error  string
}

type ReadyResult struct {
    Sources  []ReadySource
    Summary  ReadySummary
    TownRoot string
}

type ReadySummary struct {
    Total    int
    BySource map[string]int
    P0Count, P1Count, P2Count, P3Count, P4Count int
}
```

### Flags

| flag | default | notes |
|---|---|---|
| `--json` | `false` | Emit the full `ReadyResult` as indented JSON. |
| `--rig <name>` | `""` | Filter to a single rig (also skips the town source). |

### Subcommands

None.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Rig beads errors silently degraded:** `ready.go:158-160` — when `rigBeads.Ready()` fails for a rig, the error is stored in `ReadySource.Error` and surfaced as a warning at `ready.go:246-251`, but only if some sources succeeded. If ALL sources fail, a proper error is returned. **Present** — partial failure is visible but doesn't block the command.
- **Wisp ID query failures swallowed:** `ready.go:389-390` — `getWispIDs` returns `nil` on any error (Dolt unavailable, wisp table missing), meaning wisps silently leak into ready output. **Absent** — user sees wisps in ready work with no indication the filter failed.
- **Formula names read failure swallowed:** `ready.go:333-335` — `os.ReadDir` failure returns `nil` map, so formula scaffolds silently appear in ready work. **Absent** — same pattern as wisp filter.

## Notes / open questions

- **`gt ready` vs `bd ready`.** `bd ready` is per-database; this
  command is the town-wide aggregator. Polecats and solo crew
  members can still use `bd ready` inside a rig; coordinators use
  `gt ready` to see everything.
- **Wisp double-filter.** `getWispIDs` (`:385`) shells out to `bd
  mol wisp list --json` once per source, so on a 10-rig town that's
  11 `bd mol wisp list` subprocess spawns. Performance concern
  lurking.
- **Formula scaffold filter is path-based.** `getFormulaNames`
  walks `<beadsPath>/formulas` for `*.formula.toml` files (`:331-
  351`). If formulas are discovered elsewhere (e.g. `~/.beads/`
  or `GT_ROOT/.beads/`, as [synthesis](synthesis.md) searches), a
  scaffold could leak through on this command.
- **Identity bead filter uses heuristics.** ID-suffix `-role` and
  ID-substring `-rig-` are brittle proxies for label queries
  (`bd ready --json` doesn't include labels, per the comment at
  `:412-416`). A schema change to the bead ID format could break
  the filter.
- **Related commands.** [bead](bead.md) and [cat](cat.md) view
  individual beads; [close](close.md)/[done](done.md) transition
  them out of ready state. See [scheduler](scheduler.md) for the
  dispatch side of the same readiness concept.
