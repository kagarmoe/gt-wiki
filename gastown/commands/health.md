---
title: gt health
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/health.go
tags: [command, ungrouped, diagnostics, dolt, beads-exempt, health]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt health

Comprehensive health report for the Gas Town data plane — Dolt server,
production databases, pollution scan, backup freshness, zombie
processes, and orphan databases — in human or JSON form. The reusable
probe primitives (TCP, latency, database count, zombie scan, backup
freshness) live in the [health package](../packages/health.md); this
command is the package's only in-tree importer.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77` — `health`
runs without a functioning `bd`, because part of its job is to diagnose
a broken Dolt)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/health.go:85-416`.

### Invocation

```
gt health [--json]
```

`Use: "health"` (`health.go:86`). `Args`: default (any).

### Flags

| flag | type | default | description |
|------|------|---------|-------------|
| `--json` | bool | `false` | Output as JSON (bound to `healthJSON` on `health.go:103`) |

### Behavior

`runHealth` on `health.go:107-147`:

1. **Require a town root** via `workspace.FindFromCwdOrError()`. Returns
   `not in a Gas Town workspace: <err>` if not found.
2. **Build a `HealthReport`** (`health.go:27-35`) with six sections,
   populated in order by six `checkXxx` helpers. Each is defensive —
   partial failures don't abort the whole report.
3. **Output**: if `--json`, emit a pretty-indented JSON encoding of the
   report; otherwise call `printHealthReport(report)` for a
   human-readable rendering with `●` section headers and `✓`/`!` status
   icons.

### The six sections

**1. Dolt Server** — `checkServerHealth(townRoot)` on `health.go:149-178`.
Calls `doltserver.IsRunning(townRoot)` and, if running, collects PID,
port, query latency, connection counts, disk usage, and last-commit
age/db via `doltserver.GetHealthMetrics(townRoot)` and
`doltserver.LoadState(townRoot)`. Fills `ServerHealth` struct
(`health.go:37-48`).

**2. Databases** — `checkDatabaseHealth(port)` on `health.go:180-214`.
For each of `hq`, `gt`, `mo` (hard-coded production DBs on
`health.go:181`), opens a MySQL connection via the go-sql-driver and
counts:

- `SELECT COUNT(*) FROM issues`
- `SELECT COUNT(*) FROM issues WHERE status IN ('open','in_progress')`
- `SELECT COUNT(*) FROM wisps`
- `SELECT COUNT(*) FROM wisps WHERE status IN ('open','hooked','in_progress')`
- `SELECT COUNT(*) FROM dolt_log` — commits

All queries share a 10s context (`health.go:195`). Errors are swallowed
with `_ =`, so a broken table just reports zero.

**3. Pollution scan** — `checkPollution(port)` on `health.go:216-270`.
For each production DB, runs six `WHERE`-clause probes against the
`issues` table to detect garbage:

| pattern | where clause |
|---------|--------------|
| `--help artifacts` | `title LIKE '--%'` |
| `CLI usage output` | `title LIKE 'Usage: %'` |
| `offlinebrew test prefix` | `id LIKE 'offlinebrew-%'` |
| `non-ephemeral wisp ID in issues table` | `id LIKE '%-wisp-%' AND (ephemeral IS NULL OR ephemeral = false)` |
| `test issue title` | `title LIKE 'Test Issue%'` |
| `test ID prefix` | `id LIKE 'test%'` |

Each query appends `AND status != 'closed' LIMIT 10`, so the scan only
surfaces non-closed pollution and caps at 10 per pattern per DB.

**4. Backups** — `checkBackupHealth(townRoot)` on `health.go:272-309`.

- **Dolt filesystem** (`health.go:276-285`): reads the newest file
  in `<townRoot>/.dolt-backup`. Stale threshold = 30 minutes.
- **JSONL git** (`health.go:288-306`): runs `git -C
  ~/.dolt-archive/git log -1 --format=%ci` and compares against `now()`.
  Stale threshold = 30 minutes. Parsed via
  `time.Parse("2006-01-02 15:04:05 -0700", ...)`.

**5. Processes** — `checkProcessHealth(expectedPort)` on
`health.go:313-319`. Delegates to `health.FindZombieServers([]int{port})`
and returns a `ProcessHealth` with zombie count and PIDs. The inline
comment notes this uses lsof-based port discovery (not `pgrep`/`ps`
string matching) — "ZFC fix: gt-fj87."

**6. Orphan DBs** — `checkOrphanDBs(townRoot)` on `health.go:321-335`.
Delegates to `doltserver.FindOrphanedDatabases(townRoot)` and formats
sizes via `formatBytes` (the same helper re-exported from
[krc](krc.md) at `krc.go:459-471`).

### Human-readable output

`printHealthReport` on `health.go:337-416` prints each section with a
bold `●` header. Icons: `✓` for healthy, `!` for problems, `○` for
missing/not-configured. The server section prints "not running" in dim
style when `doltserver.IsRunning` returns false — no subsequent sections
are populated in that case (`health.go:121-128` gates DB and pollution
on `report.Server.Running`).

## Related commands

- [doctor](doctor.md) — broader Gas Town doctor; probably calls into
  `health` or shares plumbing.
- [vitals](vitals.md) — sibling in the Diagnostics group.
- [repair](repair.md) — remediation counterpart.
- [dolt](dolt.md) — Dolt-server lifecycle operations that health reports
  on.
- [cleanup](cleanup.md), [orphans](orphans.md) — remediate the orphan DBs
  reported here.
- [metrics](metrics.md) — OTLP metrics export; disjoint from this local
  health snapshot.
- [status](status.md) — user-facing workspace status (different scope).
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Ungrouped but diagnostic.** `health` is beads-exempt and obviously
  a diagnostic, but has no `GroupID` — so it does not appear under
  "Diagnostics" in `gt --help`. [doctor](doctor.md), [vitals](vitals.md)
  and [repair](repair.md) are all grouped. Why is `health` the outlier?
  Possibly because it can't be run from outside a town root anyway.
- **Hard-coded production DB list.** `productionDBs := []string{"hq",
  "gt", "mo"}` appears twice (`health.go:181`, `health.go:217`). If
  a fourth production DB is ever introduced, both lists must be kept
  in sync.
- **Pollution scan hard-codes six patterns.** Any new pollution family
  needs a code edit. This could live in a config file alongside the
  KRC TTLs — worth a follow-up note.
- **No `--verbose` mode.** The report is either human pretty-print or
  `--json`. There is no per-section toggle; callers that want a single
  section must parse JSON.
- **Stale threshold = 30 min** is hard-coded for both backup checks
  (`health.go:283`, `health.go:302`). Worth making configurable if
  long-running offline workspaces are a use case.
