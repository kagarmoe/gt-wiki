---
title: gt krc
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/krc.go
tags: [command, ungrouped, ttl, ephemeral, beads-exempt, krc, prune]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt krc

Manage TTL-based lifecycle for Level 0 ephemeral data — the "Key Record
Chronicle." Inspect event statistics, prune expired events, view/modify
TTL configuration, and read auto-prune scheduling state.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`; KRC is
event-file-based and does not depend on `bd`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/krc.go:16-634`.

### Invocation

```
gt krc stats [--json]
gt krc prune [--dry-run] [--auto]
gt krc config
gt krc config set <pattern> <ttl>
gt krc config reset
gt krc decay [--json]
gt krc auto-prune-status
```

`Use: "krc"` (`krc.go:17`). The parent `RunE` is `cmd.Help()`
(`krc.go:35-37`) — running `gt krc` bare prints the subcommand list
rather than doing anything.

### Concept

From the `Long` help on `krc.go:19-34`:

> Key Record Chronicle (KRC) manages TTL-based lifecycle for Level 0
> ephemeral data.
>
> Per DOLT-STORAGE-DESIGN-V3.md, Level 0 includes patrol heartbeats,
> status checks, and other operational data that decays in forensic
> value over days.

The storage target is the town's `.events.jsonl` and `.feed.jsonl`
files. Each event has a type, and each type has a configurable TTL and
a "decay curve" (`rapid`, `steady`, `slow`, `flat`) that models how
forensic value degrades over time.

### Subcommands

Registered in `init()` on `krc.go:138-152`:

#### 1. `gt krc stats` (`krc.go:40-50`, run on `krc.go:154-234`)

`Use: "stats"`. Shows:

- File sizes and event counts for `.events.jsonl` and `.feed.jsonl`
  (`krc.go:184-187`).
- Age distribution: `0-1d`, `1-7d`, `7-30d`, `30d+` (`krc.go:190-197`).
- Oldest/newest event timestamps and ages (`krc.go:200-205`).
- TTL status by type, sorted expired-count-desc then name-asc
  (`krc.go:211-232`). For each type, prints TTL, count, and either
  `OK` or `<N> expired`.

`--json` flag (`krc.go:150`) outputs `stats` as JSON via
`json.MarshalIndent`.

#### 2. `gt krc prune` (`krc.go:52-62`, run on `krc.go:236-332`)

`Use: "prune"`. Three modes:

- **`--auto`** (`krc.go:149`): daemon-mode gate. Calls
  `runKrcAutoPrune(townRoot, config)` (`krc.go:473-504`) which calls
  `krc.AutoPrune(...)`. Only prunes if `PruneInterval` has elapsed
  since the last run. If skipped, prints `Auto-prune skipped (next in
  <dur>)`. If ran but found nothing, prints `Auto-prune ran: no
  expired events`.
- **`--dry-run`** (`krc.go:148`): computes expired counts via
  `krc.GetStats`, prints a per-type breakdown sorted by count,
  totals, and `Run without --dry-run to prune.`
- **Default (actually prune)** (`krc.go:294-329`): instantiates
  `krc.NewPruner(townRoot, config)`, calls `pruner.Prune()`, and
  prints a `Prune complete:` block with processed/pruned/retained
  counts, bytes saved, duration, and a `Pruned by type:` breakdown.

#### 3. `gt krc config` (`krc.go:64-81`, run on `krc.go:334-367`)

`Use: "config [subcommand]"`. Bare form prints the active
configuration:

- `Config file:` path via `krc.ConfigFile(townRoot)`.
- `Default TTL`, `Prune interval`, `Min retain` from `krc.LoadConfig`.
- `TTLs by pattern:` sorted alphabetically.

##### 3a. `gt krc config set <pattern> <ttl>` (`krc.go:83-94`, run on `krc.go:369-404`)

`Args: cobra.ExactArgs(2)`. Parses TTL via `krcParseDuration` on
`krc.go:422-433` (supports `d`/`h`/`m`/etc.; `7d` = `168h`). Pattern
`"default"` sets the default TTL; otherwise writes to `config.TTLs[pattern]`.
Saves via `krc.SaveConfig(townRoot, config)`.

##### 3b. `gt krc config reset` (`krc.go:96-104`, run on `krc.go:406-419`)

Overwrites the config file with `krc.DefaultConfig()`.

#### 4. `gt krc decay` (`krc.go:106-119`, run on `krc.go:506-581`)

`Use: "decay"`. Generates a forensic-value decay report via
`krc.GenerateDecayReport(stats, config)`. Prints a header row with
total events, average score, at-risk count, expired count. Then a
table with columns `TYPE | CURVE | TTL | COUNT | AVG AGE | SCORE |
STATUS`. Status is color-coded:

- `>= 50%` → `healthy` (green)
- `25%–50%` → `decaying` (dim)
- `< 25%` → `low value` (yellow)
- `ExpiredCount > 0` → `<N> expired` (yellow, takes precedence)

Footer reminds that curves are: `rapid` (heartbeats), `steady`
(sessions), `slow` (errors), `flat` (audit). `--json` emits the
`report` struct as JSON.

#### 5. `gt krc auto-prune-status` (`krc.go:121-129`, run on `krc.go:583-634`)

`Use: "auto-prune-status"`. Reads `krc.LoadAutoPruneState(townRoot)`
and prints:

- `Prune interval`
- `Last prune` time and age (or `never`)
- Status: `prune pending` (never ran), `prune due`, or `next in <dur>`
- Cumulative totals: `Total prunes`, `Total pruned` events, `Total
  space freed`
- If the last run had prunes, a `Last prune result:` block with
  processed/pruned/retained counts.

### Shared helpers

- **`krcFormatDuration`** (`krc.go:436-457`) — formats durations with
  `d`/`h`/`m` suffixes and mixed forms like `1d12h` / `2h30m`.
- **`krcParseDuration`** (`krc.go:422-433`) — extends
  `time.ParseDuration` with `d` for days (`7d` → 168h).
- **`formatBytes`** (`krc.go:459-471`) — binary human-readable bytes.
  Re-used by [health](health.md).

## Related commands

- [feed](feed.md), [trail](trail.md), [audit](audit.md) — read
  `.events.jsonl` / `.feed.jsonl`; KRC manages the same files' TTLs.
- [patrol](patrol.md), [heartbeat](heartbeat.md) — emitters of the
  Level 0 ephemeral events that KRC prunes.
- [seance](seance.md) — may consume the same event streams for
  forensics; its value window is what KRC's decay curves model.
- [deacon](deacon.md) — the `--auto` mode is "designed for daemon/deacon
  integration" per `krc.go:473-474`, so deacon is presumably the
  scheduler that calls `gt krc prune --auto` on a timer.
- [health](health.md) — `formatBytes` is shared between the two
  commands (same package).
- [../packages/krc.md](../packages/krc.md) — the TTL pruner library
  and forensic decay model that this command drives.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Kubernetes/resource management utility. All operations propagate errors. No multi-step sequences with partial-completion risk at the command level.

## Notes / open questions

- **Reference doc is DOLT-STORAGE-DESIGN-V3.md** (`krc.go:20`). Worth
  reading when building this page further — it will explain the
  "Level 0" classification.
- **Only the deacon tracks auto-prune state.** `LoadAutoPruneState` /
  `SaveAutoPruneState` imply a single authoritative state file per
  town root. If multiple agents call `krc prune --auto` simultaneously,
  races are theoretically possible though the `min retain count` /
  atomic-rename invariant mitigates data loss.
- **Decay curve tuning.** Each type's curve is set by a config field
  not visible from `krc config` output — `rapid`/`steady`/`slow`/`flat`
  are baked into `krc.GenerateDecayReport`. If customization is wanted,
  the config surface needs extension.
- **`gt krc` bare runs `cmd.Help()`** rather than printing stats — a
  minor UX detail.
