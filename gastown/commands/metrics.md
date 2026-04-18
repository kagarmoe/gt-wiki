---
title: gt metrics
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/metrics.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, usage, command-stats, dead-code, beads-exempt]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 1}
---

# gt metrics

Reads the local append-only command usage log at `~/.gt/cmd-usage.jsonl`
and prints usage frequency, a per-actor breakdown, or a list of
registered commands that were never invoked ("dead commands").

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/metrics.go:29-36`)
**Beads-exempt:** **yes** — `metrics` is in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:71`, with the inline
comment: "Metrics reads local JSONL, no beads needed."
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/metrics.go:15-end`
(the file continues past line 228; only the first ~228 lines are covered
here).

### Invocation

```
gt metrics [--by-actor] [--since N] [--dead]
```

Single terminal command (no subcommands). Runs `runMetrics`
(`metrics.go:45-70`).

### Behavior

`runMetrics` (`metrics.go:45-70`):

1. Reads the usage log via `readUsageLog` (`metrics.go:72-92`), which
   opens a file at the package-level constant `logUsagePath` (defined
   in another file in `internal/cmd/` — not visible in this source, but
   referenced at `metrics.go:73`). Returns an error with the exact
   message `"no usage data yet (run some gt commands first)"` when the
   file does not exist (`metrics.go:75-77`).
2. Scans line-by-line, decoding each line as a `usageEntry` JSON
   object. Malformed lines are silently skipped (`metrics.go:83-89`).
3. If `--since N` is set, filters to entries newer than N days ago
   via `time.Now().AddDate(0, 0, -metricsSince)` (`metrics.go:52-61`).
4. Dispatches to one of three output modes:
   - `--dead` → `showDeadCommands` (`metrics.go:63-65`, defined at
     `metrics.go:181-213`)
   - `--by-actor` → `showByActor` (`metrics.go:66-68`, defined at
     `metrics.go:135-178`)
   - default → `showFrequency` (`metrics.go:69`, defined at
     `metrics.go:95-132`)

### Default mode — frequency table

`showFrequency` (`metrics.go:95-132`):

1. Counts invocations per command and tracks the latest timestamp per
   command (`metrics.go:101-108`).
2. Formats last-used as `YYYY-MM-DD` via `time.Parse(time.RFC3339, ...)`
   (`metrics.go:117-120`).
3. Sorts rows descending by count (`metrics.go:123`).
4. Prints a fixed-width table: `COMMAND (35) | COUNT (6) | LAST USED`
   (`metrics.go:125-129`).
5. Footer: `"Total: N invocations across M commands"`
   (`metrics.go:130`).

### `--by-actor` mode

`showByActor` (`metrics.go:135-178`): groups entries into
`actor → cmd → count` map, sorts actors alphabetically, and for each
actor prints a per-command table sorted descending by count, with a
`"ACTOR (N invocations)"` header.

### `--dead` mode

`showDeadCommands` (`metrics.go:181-213`):

1. Collects all invoked command names from the usage log into a set
   (`metrics.go:183-186`).
2. Walks the cobra command tree starting at `rootCmd`
   (`metrics.go:189-190`, `walkCommands` at `metrics.go:216-227`).
   Only leaf commands with a `Run` or `RunE` handler are counted
   (`metrics.go:218-221`).
3. Any command in the registered set but not in the invoked set is
   "dead" (`metrics.go:193-198`).
4. Prints an alphabetized list with counts
   (`metrics.go:200-211`).

This is an in-process drift detection tool — the list of registered
commands comes from the live `rootCmd` tree (same registry the binary
is running), not a static manifest.

### Data shape

`usageEntry` (`metrics.go:38-43`):

```go
type usageEntry struct {
    Ts    string `json:"ts"`
    Cmd   string `json:"cmd"`
    Actor string `json:"actor"`
    Argc  int    `json:"argc"`
}
```

Written by `logCommandUsage` in `persistentPreRun` — see the
`persistentPreRun` sequence documented in
[../binaries/gt.md](../binaries/gt.md) step 4, which notes that
usage logging is fire-and-forget and excludes `tap` and [`signal`](signal.md).

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`metrics.go:21-27`):

| Flag          | Type | Default | Purpose                                                  |
|---------------|------|---------|----------------------------------------------------------|
| `--by-actor`  | bool | `false` | Show breakdown by actor                                  |
| `--since N`   | int  | `0`     | Only include entries from the last N days (`0` = all)    |
| `--dead`      | bool | `false` | Show registered commands never invoked                   |

## Related

- [costs](costs.md) — reads `~/.gt/costs.jsonl`; metrics reads
  `~/.gt/cmd-usage.jsonl`. Both are local append-only JSONL logs under
  `~/.gt/`, both beads-exempt.
- [audit](audit.md) — provenance of *who did what* across workspace
  sources; metrics is *which commands got run* from the local usage
  log. No overlap.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; `metrics`
  log is fed by the `logCommandUsage` step of `persistentPreRun`.
- [README.md](README.md) — command tree index.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Malformed JSONL lines silently skipped:** `readUsageLog` at
  `metrics.go:86-88` skips lines that fail `json.Unmarshal` with a
  bare `continue`. No warning, no count. Corrupted entries are
  invisible. **Absent.**
- **Timestamp parse errors silently ignored:** `showFrequency` at
  `metrics.go:118-119` uses `time.Parse` and on error keeps the
  raw string as the "last used" value. Not harmful but produces
  inconsistent output format. **Absent** — no warning.

## Notes / open questions

- `logUsagePath` is referenced at `metrics.go:73` but defined
  elsewhere in `internal/cmd/` — finding the exact path would require
  a cross-file grep. Likely `~/.gt/cmd-usage.jsonl` per the Long text
  at `metrics.go:33-34`.
- The `logCommandUsage` helper and the `tap`/[`signal`](signal.md) exclusion list
  live in `root.go`; linking to a dedicated page would be useful.
- `buildCommandPath` is referenced at `metrics.go:219` but not defined
  in this file — another cross-reference to resolve.
- `--since N` counts calendar days, not wall-clock hours. `--since 1`
  means "since 24h ago" via `AddDate(0, 0, -1)`.
- Dead-command detection excludes cobra's built-in `help` and
  `completion` commands if they lack `Run`/`RunE` — worth verifying
  the tree walk matches user expectations.
- File continues past line 228 (`buildCommandPath` may be in a
  shared helper file); this page covers the public-facing flags and
  output modes.
