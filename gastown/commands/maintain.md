---
title: gt maintain
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/maintain.go
tags: [command, services, dolt, maintenance, gc, backup]
---

# gt maintain

Runs the full Dolt maintenance pipeline in one command: backup, reap
closed wisps, flatten over-threshold databases, and `dolt_gc()`. All
operations run via SQL on the live server — no downtime — leaning on
Tim Sehn's (Dolt CEO) 2026-02-28 confirmation that
`DOLT_RESET --soft` + `DOLT_COMMIT` and `dolt_gc()` are safe on a
running server even with concurrent writes.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/maintain.go:38-230`.

### Invocation

```
gt maintain                # Interactive — show plan, ask y/N
gt maintain --force        # Non-interactive (daemon/cron)
gt maintain --dry-run      # Preview without changes
gt maintain --threshold 50 # Custom flatten threshold
```

### Flags

(`maintain.go:63-68`)
- `--force` (bool, default false) — skip the confirmation prompt
- `--dry-run` (bool, default false) — preview without making changes
- `--threshold N` (int, default `defaultMaintainThreshold = 100`) —
  minimum commit count before a database is flattened

### Constants and timeouts

(`maintain.go:21-30`)
- `defaultMaintainThreshold = 100` commits
- `maintainGCTimeout = 5 * time.Minute` per database
- `maintainBackupTimeout = 2 * time.Minute` per database
- `maintainQueryTimeout = 30 * time.Second` for individual SQL queries
  during flatten

### Phase sequence

`runMaintain` at `maintain.go:77-230`.

**Preflight**
- Resolve town root via `workspace.FindFromCwdOrError`.
- Refuse if [`doltserver`](../packages/doltserver.md)`.DefaultConfig(townRoot).IsRemote()` —
  maintenance requires a local server (`maintain.go:84-86`).
- Refuse if `doltserver.IsRunning(townRoot)` is false (suggests
  `gt dolt start`) — needed for the reap and flatten phases.

**Phase 0 — Build maintenance plan** (`maintain.go:94-135`)
- `doltserver.ListDatabases(townRoot)` enumerates databases. Bail
  cleanly if none.
- For each database, build a `maintainDBInfo` (`maintain.go:71-75`)
  containing:
  - `commitCount` from `maintainCountCommits` — runs
    `SELECT COUNT(*) FROM <db>.dolt_log` with the 30s query timeout
    (`maintain.go:233-249`).
  - `hasBackup` from `maintainHasBackup` — shells out to `dolt backup`
    in the database dir and looks for `<dbname>-backup` in the output
    (`maintain.go:252-272`).
- Print the plan: per-database commit count, plus `→ flatten` and
  `[backup]` tags. Footer counts: total databases, will-backup,
  will-flatten (with the active threshold), will-gc.
- If `--dry-run`, stop here.

**Confirmation** (`maintain.go:142-152`)
Unless `--force`, prompt `Proceed? [y/N]` with `bufio.NewReader`. Any
answer other than `y`/`yes` aborts.

**Phase 2 — Backup** (`maintain.go:159-173`)
For each DB with `hasBackup`, runs `maintainBackupSync`
(`maintain.go:275-288`) which shells out to `dolt backup sync
<dbname>-backup` in the database dir under the 2-minute timeout.
Failures are warnings, not aborts.

**Phase 3 — Reap closed wisps** (`maintain.go:175-188`)
Per database: `doltserver.PurgeClosedEphemerals(townRoot, db.name,
false)`. Reports per-DB count and accumulates `totalReaped`.

**Phase 4 — Flatten over-threshold databases** (`maintain.go:190-207`)
Per database where `commitCount >= threshold`, calls
`maintainFlattenDB` (`maintain.go:301-363`). After flatten, recounts
commits via `maintainCountCommits` and prints the `pre → post`
delta.

**Flatten internals** (`maintain.go:301-363`):
1. Open a `database/sql` connection via `maintainOpenDB`
   (`maintain.go:291-295`) — the DSN uses MySQL protocol with
   `parseTime=true&timeout=5s&readTimeout=30s&writeTimeout=30s`.
2. Trivial `SELECT 1` connection check.
3. **Pre-flight row counts** via `flattenGetRowCounts(db, dbName)` —
   captured for integrity verification.
4. Find the root commit:
   `SELECT commit_hash FROM <db>.dolt_log ORDER BY date ASC LIMIT 1`.
5. `USE <db>` to scope subsequent calls.
6. `CALL DOLT_RESET('--soft', '<rootHash>')` — collapses history but
   keeps every row staged.
7. `CALL DOLT_COMMIT('-Am', 'maintain: flatten <db> history')` to
   commit the flattened state.
8. **Post-flight row counts** — must match pre-flight per table; any
   table missing or any row count differing is reported as an
   integrity error (and the flatten is considered failed).

**Phase 5 — GC** (`maintain.go:209-221`, internals at
`maintain.go:367-384`)
Per database: `CALL dolt_gc()` via `maintainGCDatabase`. Bound by the
5-minute `maintainGCTimeout`. If the context deadline is hit, the
error is rewritten to `timeout after 5m0s`. Per-DB timing is reported
in the output.

**Summary** (`maintain.go:223-229`)
Reports total elapsed time (rounded to seconds), total wisps reaped,
databases flattened, databases gc'd.

### Notable invariants

- **Local-only**: refuses on remote Dolt config. Maintenance is
  always run against the workspace's own server.
- **Server stays up**: every phase is SQL-on-running-server.
  Documented attribution: Tim Sehn, 2026-02-28
  (`maintain.go:156-157`, `:298-300`, `:366-367`).
- **Integrity gate** in flatten — pre/post row counts per table must
  match exactly. Any divergence aborts the flatten for that database.
- **Per-DB failure isolation**: failures during backup, reap, flatten,
  or gc are reported per-database and the pipeline continues. Only
  preflight (workspace resolution, remote check, server-running
  check) is fatal.

## Related

- [dolt](./dolt.md) — `gt dolt start/stop/recover` and the underlying
  primitives this command orchestrates
- [doctor](./doctor.md) — diagnostics that may suggest running
  `gt maintain`
- [compact](./compact.md) — separate compaction primitive (compare
  scopes)
- [repair](./repair.md) — alternative recovery path
- [upgrade](./upgrade.md) — separate lifecycle command
- [daemon](./daemon.md) — likely scheduled to invoke `gt maintain
  --force` from cron / supervisor (verify in
  `internal/daemon`)

## Notes / open questions

- **`flattenGetRowCounts` is referenced** at
  `maintain.go:318, :349` but defined elsewhere in this package
  (likely a sibling file). Worth confirming the exact table iteration
  it does — is it all user tables, or does it skip `dolt_*` system
  tables?
- **`PurgeClosedEphemerals` API**: third arg is `false` here
  (`maintain.go:179`) — likely a `dryRun` flag. Confirm the
  signature and surface it as a CLI option if useful.
- **No `--db` flag**: the command always operates on every database
  reported by `ListDatabases`. Compare to
  [`gt dolt sync`](./dolt.md) which does support `--db`. Could be
  useful to add for surgical maintenance runs.
- **Failure-mode telemetry**: per-DB failures print to stdout but no
  feed event is emitted (compare `gt down`'s `events.LogFeed` call).
  Worth a small note if maintenance failures need to be surfaced
  elsewhere.
- **Threshold of 100 commits** is a magic default — likely informed
  by Dolt commit-graph performance characteristics. Origin not
  documented in the file.
