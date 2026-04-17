---
title: internal/reaper
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/reaper/reaper.go
tags: [package, services, reaper, cleanup, dolt, wisps, beads, ttl]
---

# internal/reaper

Wisp and issue cleanup operations for Dolt databases. The package
is the library behind the `mol-dog-reaper` formula and the
[`gt reaper`](../commands/reaper.md) CLI — it executes SQL
operations (scan, reap, purge, auto-close, close-plugin-receipts)
against Dolt but does NOT make eligibility decisions. Those
decisions belong to the Dog agent (or the daemon orchestrator)
driving the formula. The `reaper` package is the muscle, not the
brain.

**Go package path:** `github.com/steveyegge/gastown/internal/reaper`
**File count:** 1 non-test `.go` file — `reaper.go` (952 lines).
**Role:** [`reaper` role](../roles/reaper.md) — the cleanup persona
(really a role performed by a Dog executing `mol-dog-reaper`).
**CLI command:** [`gt reaper`](../commands/reaper.md)
**Imports (notable):** `database/sql`, `github.com/go-sql-driver/mysql`
(MySQL protocol driver for Dolt's wire protocol), stdlib
(`context`, `encoding/json`, `regexp`, `strings`, `time`).
No gastown package imports — the `reaper` package is deliberately
self-contained so it can run even when other gastown packages are
in a broken state.
**Imported by (notable):** [`gt reaper`](../commands/reaper.md)
(`/home/kimberly/repos/gastown/internal/cmd/reaper.go`), the
`mol-dog-reaper` formula's shell execution, and (through the CLI)
the [`deacon` package](deacon.md) when patrol scheduling uses it.

## What it actually does

One file, one concern: issue SQL against Dolt. The package is
stateless — every call takes a `*sql.DB`, does its work, and
returns a result struct. It never spawns goroutines, never keeps
a connection pool of its own, and never touches the filesystem.

### Database discovery

- `validDBName` regex (`reaper.go:22`) — matches safe database
  names (alphanumeric, underscore, hyphen).
- `DefaultDatabases = []string{"hq"}` (`reaper.go:29`) — static
  fallback list when `SHOW DATABASES` fails. The older `gt` / `bd`
  entries were removed in gh#2385; modern towns use `hq` plus
  rig-specific names.
- `testPollutionPrefixes = ["testdb_", "beads_t", "beads_pt",
  "doctest_"]` (`reaper.go:32`) — prefixes the discovery layer
  filters out. Test pollution accumulates on production Dolt and
  degrades performance; see the top-level
  `/home/kimberly/repos/CLAUDE.md` warning about orphan databases.
- `DiscoverDatabases(host string, port int) []string`
  (`reaper.go:54-98`) — `SHOW DATABASES` over MySQL wire protocol,
  filters `information_schema` + `mysql` + test prefixes. Falls
  back to `DefaultDatabases` on any error. 5-second context
  timeout.
- `ValidateDBName(dbName) error` (`reaper.go:161-167`) — regex
  gate before every operation. Callers should warn-and-skip on
  failure rather than crash.
- `OpenDB(host, port, dbName, readTimeout, writeTimeout)
  (*sql.DB, error)` (`reaper.go:169-196`) — opens a MySQL
  connection with explicit read/write timeouts. Parses time
  values in-place via `parseTime=true`.

### Parent-exclusion join

- `parentExcludeJoin(dbName) (joinClause, whereCondition string)`
  (`reaper.go:198-212`) — produces a LEFT JOIN anti-pattern clause
  that excludes wisps whose parent molecule is still open. The
  LEFT JOIN approach avoids an O(n*m) correlated subquery cost
  seen with 1800+ closed wisps (gh-jd1z, gh-wvd2).

### Schema probe

- `HasReaperSchema(db) (bool, error)` (`reaper.go:214-226`) — checks
  whether the `wisps` table exists. Databases without the schema
  are silently skipped by the CLI subcommands. This lets
  `gt reaper run` run against a mixed-schema Dolt server without
  noise.

### Scan

- `ScanResult` struct (`reaper.go:101-109`) — `Database`,
  `ReapCandidates`, `PurgeCandidates`, `MailCandidates`,
  `StaleCandidates`, `OpenWisps`, `Anomalies`.
- `Scan(db, dbName, maxAge, purgeAge, mailDeleteAge, staleIssueAge)
  (*ScanResult, error)` (`reaper.go:228-311`) — read-only count
  pass. Five queries: reap candidates (open wisps past max_age
  whose parent molecule is closed/missing/orphaned via
  `parentExcludeJoin`), purge candidates (closed wisps past
  purge_age — NO parent check, gh-wvd2), mail candidates (closed
  `gt:message`-labelled issues past mail_age), stale candidates
  (open issues past stale_age with priority > 1, not epics, no
  active dependencies), total open wisps. Runs a dangling-
  parent-ref anomaly check at the end (`reaper.go:296-308`) and
  records any findings in `result.Anomalies`. Queries on the
  `issues`/`labels`/`dependencies` tables tolerate
  `isTableNotFound` errors since those tables may live on a
  separate Dolt instance.

### Reap

- `ReapResult` struct (`reaper.go:111-118`).
- `Reap(db, dbName, maxAge, dryRun) (*ReapResult, error)`
  (`reaper.go:315-426`) — the write path for closing stale wisps.
  Uses a 2-minute query timeout instead of the default so batched
  UPDATEs don't hit the server-side limit. Builds the where
  clause from `parentExcludeJoin` plus the age cutoff plus the
  `issue_type != 'agent'` exclusion (agent beads have persistent
  identity and are never wisp-reaped). In dry-run mode counts
  without modifying; otherwise UPDATEs in batches.

### Purge

- `PurgeResult` struct (`reaper.go:120+`).
- `Purge(db, dbName, purgeAge, mailDeleteAge, dryRun)
  (*PurgeResult, error)` (`reaper.go:428-447`) — orchestrator that
  calls `purgeClosedWisps` + `purgeOldMail`.
- `purgeClosedWisps(db, dbName, purgeAge, dryRun)
  (int, []Anomaly, error)` (`reaper.go:449-527`) — DELETEs closed
  wisps past cutoff. Detects anomalies along the way.
- `purgeOldMail(db, dbName, mailDeleteAge, dryRun) (int, error)`
  (`reaper.go:529-585`) — DELETEs closed `gt:message`-labelled
  issues past cutoff.

### AutoClose

- `AutoCloseResult` struct (declared in the types section).
- `AutoClose(db, dbName, staleAge, dryRun) (*AutoCloseResult, error)`
  (`reaper.go:587-707`) — closes open issues with no updates past
  cutoff. Per the CLI Long help, excludes P0/P1 priority, epics,
  and issues with active dependencies. Returns per-bead metadata
  (`ID`, title, days stale) for the caller to display.

### Batch delete helper

- `batchDeleteRows(ctx, db, idQuery, cutoffArg, primaryTable,
  auxTables)` (`reaper.go:709-776`) — chunked DELETE with
  explicit `LIMIT` batches, handling primary + auxiliary table
  deletions atomically per batch.

### Plugin receipt / dispatch cleanup

- `ClosePluginReceipts(db, dbName, maxAge, dryRun)
  (*ClosePluginReceiptResult, error)` (`reaper.go:778-860`) —
  closes plugin receipt beads past maxAge.
- `ClosePluginDispatches(db, dbName, maxAge, dryRun)
  (*ClosePluginReceiptResult, error)` (`reaper.go:862-944`) —
  same for plugin dispatch beads.

### JSON formatting

- `FormatJSON(v interface{}) string` (`reaper.go:946-952`) —
  pretty-printed JSON for CLI `--json` output.

### Error helpers

- `isNothingToCommit(err) bool` (`reaper.go:35-37`) — recognises
  Dolt's "nothing to commit" error so commits are no-op-safe.
- `isTableNotFound(err) bool` (`reaper.go:43-49`) — lets queries
  gracefully skip when `issues` / `labels` / `dependencies` tables
  aren't present on the connected Dolt instance. This tolerance is
  critical because beads may be on a separate Dolt server from the
  `wisps` table the reaper is cleaning.

### Notable design choices

- **Decisionless.** The package never decides WHAT to reap — it
  takes thresholds and a flag and executes. The Dog agent or the
  daemon makes eligibility decisions. This is why
  [`gt reaper`](../commands/reaper.md) Long help calls these
  "callable helper functions" for `mol-dog-reaper`.
- **Dry-run is a first-class flag on every operation.** Every
  mutating function takes `dryRun bool` and returns the
  candidate count without modifying state. The CLI's
  `--dry-run` passes straight through.
- **Parent checks only on OPEN-wisp reap, not on closed-wisp
  purge.** The O(n*m) correlated subquery cost observed at
  1800+ closed wisps (gh-wvd2) forced the purge path to drop the
  parent check. Once a wisp is closed, it's unconditionally
  purgeable after the age cutoff — the lifecycle guarantee is
  enforced upstream by the reap path.
- **Agent-type issues are never reaped.** The
  `issue_type != 'agent'` exclusion in `Reap` protects persistent
  agent identity beads from being closed for age. Agent beads
  carry identity state the system depends on.
- **Test-pollution prefix filter lives here.** The reaper is the
  package most directly responsible for keeping Dolt healthy, so
  the "known test database prefix" list is baked in at
  `reaper.go:32`. This list mirrors the manual
  `gt dolt cleanup` command.
- **5-second context timeout on discovery, 10-second on scan/reap/
  auto-close, 2-minute on batched reap UPDATEs, 30-second on
  purge.** Chosen empirically — writes need more time than reads,
  and batched writes need more time than single writes.
- **Self-contained.** No gastown package imports means the reaper
  can run against a Dolt server even if other gastown packages
  are in a broken state (e.g. during a dependency rebuild). This
  is a deliberate constraint for the cleanup tool of last resort.

## Related wiki pages

- [`reaper` role](../roles/reaper.md) — the cleanup persona.
- [`gt reaper`](../commands/reaper.md) — CLI surface
  (`databases`, `scan`, `reap`, `purge`, `auto-close`, `run`).
- [`wisp` concept](../concepts/wisp.md) — the reaper's primary
  subject; explains WHY there's a reaper at all.
- [`dog` package](dog.md) — the Dog agent executes the
  `mol-dog-reaper` formula that drives these helpers.
- [`dog` role](../roles/dog.md) — persona.
- [`deacon` package](deacon.md) — patrol scheduling path.
- [`deacon` role](../roles/deacon.md) — the Dog's dispatcher.
- [`doltserver` package](doltserver.md) — the Dolt server the
  reaper talks to.
- [`internal/beads`](beads.md) — bead semantics (wisps, agent
  beads, priorities).

## Notes / open questions

- **No gastown package imports.** This is deliberate isolation
  but has the side effect that the reaper cannot easily reuse
  `internal/beads` helpers for query construction. All SQL is
  hand-rolled strings in this file.
- **Issues / labels / dependencies queries silently skip on
  `isTableNotFound`.** When a database lacks those tables, scan
  / reap counts for mail / stale candidates return zero without
  any indication that the count was skipped. A future anomaly
  type for "schema-incomplete database" might help operators
  distinguish "no stale issues" from "didn't even look".
- **Alert threshold `500` for open wisps is NOT in this package.**
  It lives inline in the CLI file
  (`/home/kimberly/repos/gastown/internal/cmd/reaper.go:231-233`).
  Moving it here would make it testable and configurable.
- **`DefaultQueryTimeout` is referenced** but not defined in this
  file — it's a package constant declared elsewhere in this
  single file or in the imports. A future edit should confirm
  the exact value and document it explicitly.
- **No migration / schema-evolution tooling.** If the `wisps`
  table shape changes, the reaper will break silently. There's
  no schema version guard. Coordinated upgrades are the
  operator's responsibility.
