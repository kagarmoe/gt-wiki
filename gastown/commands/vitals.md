---
title: gt vitals
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/vitals.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, dolt, databases, backups, health-dashboard]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt vitals

Unified health dashboard for the Dolt data plane: prints Dolt server
status, per-database issue stats, and backup freshness in a single
text-only report. Calls `doltserver` directly rather than going through
the reusable [health package](../packages/health.md) primitives that
back [gt health](health.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/vitals.go:19-24`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/vitals.go:28-263`.

### Invocation

```
gt vitals
```

No flags, no subcommands.

### Behavior

`runVitals` (`vitals.go:28-39`):

1. Resolves the town root via `workspace.FindFromCwdOrError`
   (`vitals.go:29-32`).
2. Calls three section printers in order (`vitals.go:33-37`):
   - `printVitalsDoltServers(townRoot)`
   - `printVitalsDatabases(townRoot)`
   - `printVitalsBackups(townRoot)`

No JSON mode, no flags, no subcommand routing. Pure text output.

### Section 1: Dolt Servers

`printVitalsDoltServers` (`vitals.go:41-68`):

1. Loads the Dolt config via `doltserver.DefaultConfig(townRoot)`.
2. Checks if the production server is running
   (`doltserver.IsRunning`). When running, calls
   `doltserver.GetHealthMetrics` for disk usage, connection count,
   max connections, and query latency (`vitals.go:47-51`):
   ```
   ● :3307  production  PID 12345  18.2GB  3/100 conn  2ms
   ```
3. Dumps any warnings from `metrics.Warnings` indented (`vitals.go:52-54`).
4. **Zombie discovery** (`vitals.go:61-67, 78-113`): calls
   `findVitalsZombies` which uses `doltserver.FindAllDoltListeners`
   (lsof-based port discovery — a gt-fj87 fix for earlier
   pgrep/ps-based string matching). For each Dolt listener **not** on
   the production port, it inspects the process's `--data-dir` via
   `doltserver.GetDoltDataDirFromProcess` and:
   - If the data dir points at a sibling Gas Town workspace (its
     parent has a `.dolt-data` directory under a valid workspace) →
     marked `foreign` and printed as "foreign workspace PID <pid>"
     with a dim `○` (`vitals.go:93-105`).
   - Otherwise → marked as "test zombie PID <pid>" with a warning
     `○` (`vitals.go:107-112`).

### Section 2: Databases

`printVitalsDatabases` (`vitals.go:115-153`):

1. Loads `doltserver.ListDatabases(townRoot)` and
   `doltserver.FindOrphanedDatabases(townRoot)` (`vitals.go:116-117`).
2. Header line reports `(N registered)` or
   `(N registered, M orphan)` (`vitals.go:119-125`).
3. Prints a table header: `Rig | Total | Open | Closed | %`
   (`vitals.go:132-134`).
4. For each non-orphaned database, calls `queryVitalsStats` which
   runs a single SQL query against the Dolt server via the `dolt`
   CLI (`vitals.go:157-186`):
   ```sql
   SELECT COUNT(*),
          SUM(CASE WHEN status='open'        THEN 1 ELSE 0 END),
          SUM(CASE WHEN status='in_progress' THEN 1 ELSE 0 END),
          SUM(CASE WHEN status='closed'      THEN 1 ELSE 0 END)
     FROM <db>.issues
   ```
   The query has a 5 s context timeout (`vitals.go:163-164`) and
   uses the `dolt` CLI's CSV output mode (`-r csv`) rather than a
   native SQL client. Row format is total, open+in_progress, closed,
   and `closed/total` percentage (`vitals.go:146-152`). On query
   failure the row renders as all dashes.

### Section 3: Backups

`printVitalsBackups` (`vitals.go:188-249`):

1. **Local Dolt backup**: reads `<townRoot>/.dolt-backup/` and counts
   direct subdirectories; the most recent `ModTime` is shown as the
   last-sync timestamp (`vitals.go:192-213`). Reports empty or
   "not found" if the directory is missing/empty.
2. **JSONL git archive**: shells out to
   `git -C <townRoot>/.dolt-archive/git log -1 --format=%ci` for
   the last commit time (`vitals.go:216-222`). Then walks per-rig
   subdirectories reading each `issues.jsonl` and counts newline-
   separated records via `strings.Split` (`vitals.go:228-240`). Output:
   `JSONL: last push YYYY-MM-DD HH:MM (N,NNN records)`.

Helper `vitalsShortHome` (`vitals.go:258-263`) truncates a path that
begins with `$HOME` to `~/...` for display.

### Subcommands

None (terminal command).

### Flags

None — `init` at `vitals.go:26` only registers the command.

## Related

- [doctor](doctor.md) — runs `NewDoltServerReachableCheck` and the
  large suite of Dolt health checks at `doctor.go:178-215` and
  `doctor.go:268-271`; vitals is the live snapshot form of that
  health data.
- [metrics](metrics.md) — metrics is cost/usage-facing, vitals is
  infrastructure-facing; the two together are the diagnostic pair for
  "how much is it costing" vs. "is it healthy."
- [status](status.md) — `status` reports Dolt running/port info as
  one line of a larger summary; vitals is the Dolt-specific deep dive.
- [repair](repair.md) — `repair`'s `StaleDoltPortCheck` touches the
  same `metadata.json` port that vitals displays from `LoadState`
  in `printVitalsDoltServers`.
- [costs](costs.md) — costs is disk/price-facing, vitals is
  operational-health-facing; see vitals' `DiskUsageHuman` line for
  the same quantity costs amortizes over time.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Database listing errors discarded:** `printVitalsDatabases` at
  `vitals.go:117-118` uses `databases, _ := doltserver.ListDatabases(townRoot)`
  and `orphans, _ := doltserver.FindOrphanedDatabases(townRoot)`. If
  the Dolt server is unreachable, both return nil and the user sees
  "Databases (0 registered)" with no error. **Absent.**
- **Per-database stat query failure silently shows dashes:**
  `queryVitalsStats` at `vitals.go:157-186` returns `nil` on any
  error (exec failure, parse failure). The caller at `vitals.go:142-144`
  prints `-` for all columns. **Absent** — no indication whether
  "no data" means empty database or query failure.
- **Dolt process running check error discarded:**
  `doltserver.IsRunning` at `vitals.go:44` has a third return value
  (error) that is discarded with `_, _`. **Absent.**

## Notes / open questions

- `doltserver.FindAllDoltListeners`, `GetDoltDataDirFromProcess`,
  `GetHealthMetrics`, `ListDatabases`, and `FindOrphanedDatabases`
  are all in `internal/doltserver/`. That package deserves a
  dedicated page; vitals is one of its biggest consumers.
- `queryVitalsStats` shells out to the `dolt` CLI for each database
  rather than using an in-process SQL client. Each call incurs
  process startup overhead — measurable on rigs with many databases.
- The "foreign workspace" heuristic relies on `.dolt-data` being the
  parent directory name; a non-standard layout would be misclassified
  as a zombie.
- No `--json` mode means vitals output is not machine-parseable.
  `status --json` is the closest equivalent for programmatic use.
- Dolt CLI credentials are passed via `DOLT_CLI_PASSWORD` env var
  (`vitals.go:168`) rather than a flag — note for operators debugging
  auth failures.
