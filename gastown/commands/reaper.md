---
title: gt reaper
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/reaper.go
tags: [command, services, dolt, beads, wisps, cleanup, maintenance]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt reaper

Wisp and issue cleanup operations against Dolt databases — the
callable helper functions for the `mol-dog-reaper` formula. The
subcommands execute SQL operations (scan, reap, purge, auto-close,
full cycle) but leave eligibility *decisions* to the Dog agent or
daemon orchestrator.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/reaper.go:27-574`.

The top-level `reaper` command is a parent with `RunE:
requireSubcommand` (`reaper.go:42`), so `gt reaper` by itself prints
the help screen and exits nonzero.

### Invocation

```
gt reaper databases                 # List databases available for reaping
gt reaper scan         [flags]      # Count reap/purge/auto-close/mail candidates
gt reaper reap         [flags]      # Close stale wisps past --max-age
gt reaper purge        [flags]      # Delete old closed wisps + mail
gt reaper auto-close   [flags]      # Close stale open issues past --stale-age
gt reaper run          [flags]      # Full cycle: scan → reap → purge → auto-close → report
```

Default connection: `127.0.0.1:3307` with
`GT_DOLT_HOST` / `GT_DOLT_PORT` or (fallback)
`BEADS_DOLT_SERVER_HOST` / `BEADS_DOLT_SERVER_PORT` environment
overrides — resolved lazily in `init()` (`reaper.go:522-540`).

### Subcommands

**`databases`** (`reaper.go:45-59`)
Thin wrapper over `reaper.DiscoverDatabases(host, port)`. Prints one
database name per line, or a JSON array if `--json` is set. Used to
drive the other subcommands' auto-discovery mode.

**`scan`** (`reaper.go:61-156`)
Reports candidate counts without modifying anything. For each
database:

1. `reaper.ValidateDBName` — skip invalid names with a stderr warning.
2. `reaper.OpenDB(host, port, dbName, 10s, 10s)` — 10-second
   read/write timeouts.
3. `reaper.HasReaperSchema` — skip databases that don't have the
   reaper tables.
4. `reaper.Scan(db, dbName, maxAge, purgeAge, mailAge, staleAge)` —
   returns a `ScanResult` with counts plus an `Anomalies` slice
   (rendered with `style.Warning`'s `ANOMALY:` prefix).

In single-database mode (`--db`), prints per-database counts.
In multi-database mode, also prints a "Scan summary (N databases)"
total block. With `--json`, emits the entire `[]*ScanResult` slice
via `reaper.FormatJSON`.

**`reap`** (`reaper.go:158-238`)
Closes wisps whose age exceeds `--max-age` *and* whose parent
molecule is already closed/missing/orphaned (per the Long help —
`reaper.Reap` enforces it inside the `reaper` package). Dry-run mode
prints `[DRY RUN] would reaped N wisps, M open remain` for each
database; real runs drop the `would`.

**Alert threshold**: in multi-database mode, if total
`totalOpen > 500`, the reap subcommand prints a stderr warning:
`WARNING: N open wisps exceed alert threshold (500)`
(`reaper.go:231-233`). The constant `500` is hard-coded inline.

**`purge`** (`reaper.go:240-324`)
Irreversible. Deletes closed wisps past `--purge-age` and closed
mail past `--mail-age`. Uses a 30-second timeout on the DB
connection (`reaper.go:272`) — longer than `scan`/`reap`/`auto-close`
because deletes can be slower. Per-database output is
`<db>: purged N wisps, M mail`; dry-run prefix `[DRY RUN] would `.
Anomaly lines are also printed under each database block.

**`auto-close`** (`reaper.go:326-406`)
Closes open issues with no updates past `--stale-age`. Per the Long
help, the underlying `reaper.AutoClose` excludes "P0/P1 priority,
epics, and issues with active dependencies". Per-bead output lines
look like:

```
  <ID> <title> (<N>d stale, db:<database>)
```

Followed by a per-database `auto-closed N stale issues` line and a
multi-database summary.

**`run`** (`reaper.go:408-520`)
The inline fallback executed "when Dog dispatch is unavailable.
Normally the daemon dispatches a Dog to execute the mol-dog-reaper
formula." Runs scan → reap → purge → auto-close in one pass per
database, collecting totals, and prints a report block:

```
[DRY RUN] Reaper cycle complete:
  Databases: N
  Reaped:    N
  Purged:    N wisps, N mail
  Closed:    N stale issues
  Open:      N wisps remain
```

Uses a 30-second per-database DB connection timeout and parses all
four duration flags up front so a bad flag fails fast.

### Database selection

All non-`databases` subcommands share the same selection logic:

1. `reaper.DiscoverDatabases(host, port)` returns every database on
   the Dolt server.
2. If `--db` is non-empty, override with `strings.Split(reaperDB,
   ",")` — so `--db=a,b,c` is a multi-database explicit list rather
   than a single database named "a,b,c".
3. Each database is then validated, opened, schema-checked, and
   either processed or skipped with a stderr note.

Databases without the reaper schema are silently skipped in
subcommands (`scan`/`reap`/`purge`/`auto-close`) but explicitly
reported with `skipped (no reaper schema)` in `run`.

### Flags

Shared flags registered in `init()` (`reaper.go:542-564`):

- `--db <list>` — comma-separated database list; default
  auto-discover
- `--host <host>` (env: `GT_DOLT_HOST` / `BEADS_DOLT_SERVER_HOST`,
  default `127.0.0.1`)
- `--port <n>` (env: `GT_DOLT_PORT` / `BEADS_DOLT_SERVER_PORT`,
  default `3307`)
- `--dry-run` — report without acting

Applied to `scan`, `reap`, `purge`, `auto-close`, `run`,
`databases`.

JSON output flag (`--json`) is only on the single-purpose
subcommands — `scan`, `reap`, `purge`, `auto-close`, `databases` —
**not** on `run` (`reaper.go:550-552`).

Threshold flags (`reaper.go:554-564`):

- `--max-age` (default `24h`) — on `scan`, `reap`, `run`
- `--purge-age` (default `168h` / 7d) — on `scan`, `purge`, `run`
- `--mail-age` (default `168h` / 7d) — on `scan`, `purge`, `run`
- `--stale-age` (default `720h` / 30d) — on `scan`, `auto-close`,
  `run`

Duration format is anything `time.ParseDuration` accepts.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/reaper.go:34-45` — Cobra `Long` text

### Verbatim
> Execute wisp reaper operations against Dolt databases.
>
> These subcommands are the callable helper functions for the mol-dog-reaper
> formula. They execute SQL operations but leave eligibility decisions to the
> Dog agent or daemon orchestrator.
>
> When run by a Dog:
>   gt reaper scan --db=gastown          # Discover candidates
>   gt reaper reap --db=gastown          # Close stale wisps
>   gt reaper purge --db=gastown         # Delete old closed wisps + mail
>   gt reaper auto-close --db=gastown    # Close stale issues

## Drift

### `reaperCmd.Long` "When run by a Dog" block lists 4 subcommands; 6 registered
- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/reaper.go:40-45`
- **Docs claim:** "When run by a Dog" example block enumerates 4 subcommands: `scan`, `reap`, `purge`, `auto-close`.
- **Code does:** `init()` at `reaper.go:566-571` registers 6 subcommands: **`databases`**, `scan`, `reap`, `purge`, `auto-close`, **`run`**. `databases` (`reaper.go:45-59`) lists databases available for reaping. `run` (`reaper.go:408-520`) executes the full scan-reap-purge-auto-close cycle in one pass — the "inline fallback when Dog dispatch is unavailable."
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code`
- **Release position:** `in-release`

See also: [gastown/drift/README.md](../drift/README.md)

## Failure modes

No failure modes discovered. `reaper.go` finds and kills orphaned processes. Process-kill failures are logged individually but don't abort the sweep. Correct continue-on-error pattern for cleanup commands.

## Notes / open questions

- **"Dog-callable helpers"** — the package doc calls these helpers
  for the `mol-dog-reaper` formula. Cross-reference
  [formula](./formula.md) and [dog](./dog.md) when those pages are
  revisited; the dispatch path is daemon → Dog → `gt reaper run`
  subcommands.
- **Alert threshold is hard-coded.** `500 open wisps` in `reap`
  and no threshold at all in `scan` or `run`. Likely drifted from
  configurable state.
- **`HasReaperSchema` means silent skip** — a database without the
  schema is not a failure; the subcommand just continues to the next
  one. Makes `gt reaper run` against a mixed-schema Dolt server
  safe by default.
- **`run` has no JSON flag** — so the canonical "full cycle"
  invocation can't be machine-parsed. Presumably intentional so the
  daemon invokes the individual subcommands with `--json` instead.
- **`reaper.FormatJSON`** lives in `internal/reaper`; whether it
  uses `encoding/json` directly or wraps it with pretty-printing
  isn't visible from this file.
- **The `databases` subcommand does not accept `--dry-run` in a
  meaningful sense** — it's a read-only listing — but the flag is
  still registered on it (`reaper.go:546`). Harmless but slightly
  noisy.

## Related

- [patrol](./patrol.md) — scheduled patrols in daemon.json; the Dog
  dispatch pathway for reaper lives there
- [deacon](./deacon.md) — health orchestrator that consumes reaper
  anomaly output
- [orphans](./orphans.md) — overlapping cleanup concept (orphan
  bead / process detection) at the session layer
- [cleanup](./cleanup.md) — related housekeeping command
- [dolt](./dolt.md) — the backing server the reaper talks to via
  `reaper.OpenDB`
- [maintain](./maintain.md) — `gt maintain` cycle also touches Dolt
  state; reaper is more focused on beads table content
- [formula](./formula.md) — `mol-dog-reaper` is the formula that
  wraps these helpers
- [dog](./dog.md) — the agent role that invokes reaper subcommands
  in normal dispatch
