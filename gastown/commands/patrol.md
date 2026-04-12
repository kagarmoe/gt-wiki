---
title: gt patrol
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/patrol.go
  - /home/kimberly/repos/gastown/internal/cmd/patrol_new.go
  - /home/kimberly/repos/gastown/internal/cmd/patrol_report.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, patrol, wisp, digest, deacon, witness, refinery]
---

# gt patrol

Group command for patrol cycle management. Owns three subcommands
(`new`, `report`, `digest`) that create patrol wisps, close them with
a summary, and aggregate ephemeral per-cycle digests into a permanent
daily summary bead.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/patrol.go:25-37`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/patrol.go:25-62`.

### Invocation

```
gt patrol <subcommand> [flags]
```

The root `patrolCmd` has no `RunE` of its own — it is a grouping
command. Running `gt patrol` with no subcommand prints the cobra
help text. The three subcommands are registered in `init()` at
`patrol.go:58-62`:

```go
patrolCmd.AddCommand(patrolDigestCmd)
patrolCmd.AddCommand(patrolNewCmd)
patrolCmd.AddCommand(patrolReportCmd)
rootCmd.AddCommand(patrolCmd)
```

### Subcommands

#### `gt patrol new`

Source: `/home/kimberly/repos/gastown/internal/cmd/patrol_new.go:13-92`.

Creates a new patrol wisp for the current role, injecting rig config
variables into the wisp so the formula has correct settings.

- Resolves role via `GetRole()` (`patrol_new.go:37-39`). `--role`
  flag overrides the detected role (`patrol_new.go:45-47`).
- Builds a `PatrolConfig` struct based on role:
  - `RoleDeacon` → `PatrolMolName: constants.MolDeaconPatrol`,
    `Assignee: "deacon"` (`patrol_new.go:52-58`)
  - `RoleWitness` → `MolWitnessPatrol`, `Assignee: <rig>/witness`
    (`patrol_new.go:59-65`)
  - `RoleRefinery` → `MolRefineryPatrol`, `Assignee: <rig>/refinery`,
    plus `ExtraVars` from `buildRefineryPatrolVars(roleInfo)` which
    reads MQ config from `config.json` and `settings/config.json`
    (`patrol_new.go:66-73`). The docstring at `patrol_new.go:20-23`
    lists the merge-queue vars that flow through: `run_tests`,
    `test_command`, `target_branch`, etc.
  - Any other role → error "unsupported role for patrol"
    (`patrol_new.go:74-76`).
- Calls `autoSpawnPatrol(cfg)` to create and hook the wisp
  (`patrol_new.go:79`). On partial failure (wisp created but hook
  attachment failed) it warns to stderr, prints the ID, and returns
  nil (`patrol_new.go:81-86`).
- Prints the patrol ID to stdout (`patrol_new.go:90`).

Flag: `--role <name>` (`patrol_new.go:32`).

#### `gt patrol report`

Source: `/home/kimberly/repos/gastown/internal/cmd/patrol_report.go:20-138`.

Closes the current patrol cycle with a summary and starts the next
cycle. Replaces the old squash+new pattern with a single atomic
command (`patrol_report.go:23-33`).

Behavior:

1. Resolves role and builds `PatrolConfig` (same switch as
   `patrol new`, `patrol_report.go:46-82`).
2. Finds the active patrol via `findActivePatrol(cfg)`
   (`patrol_report.go:85-91`). Errors if none.
3. Builds a step audit checklist via `buildStepAudit(cfg.PatrolMolName,
   patrolReportSteps)` (`patrol_report.go:97`).
4. Updates the patrol root's description with
   `"Patrol report: <summary>\n\n<step audit>"`
   (`patrol_report.go:99-105`). Update failure is non-fatal — it
   only warns.
5. Force-closes all descendant wisps recursively via
   `forceCloseDescendants` (`patrol_report.go:113-116`). If any
   descendant can't be closed, aborts so patrol retries on the next
   cycle (gt-7lx3 comment).
6. Force-closes the patrol root with reason
   `"patrol cycle complete: <summary>"` (`patrol_report.go:119-121`).
7. Spawns a new patrol cycle via `autoSpawnPatrol(cfg)`
   (`patrol_report.go:126-134`). Same partial-failure handling as
   `patrol new`.

**Step audit format** (`buildStepAudit`, `patrol_report.go:146-192`):
the formula is loaded via
`formula.GetEmbeddedFormulaContent(formulaName)` and parsed to get
the canonical step list. Each step is rendered as `<id> <STATUS>`,
joined by ` | `, with a final `(<ok>/<total>)` tally. Unreported
steps default to `SKIP`. Output format:

```
Steps: heartbeat OK | inbox-check OK | orphan-cleanup SKIP | ... (14/25)
```

`parseStepResults` (`patrol_report.go:197-210`) parses the
`--steps` flag as a comma-separated list of `step:STATUS` pairs,
upper-casing statuses.

Flags:

- `--summary <text>` — **required**, marked via
  `MarkFlagRequired("summary")` (`patrol_report.go:41-43`).
- `--steps <pairs>` — comma-separated step audit, e.g.
  `heartbeat:OK,inbox-check:OK,orphan-cleanup:SKIP`.

#### `gt patrol digest`

Source: `/home/kimberly/repos/gastown/internal/cmd/patrol.go:39-386`.

Aggregates ephemeral per-cycle patrol digest beads for a target date
into a single permanent "Patrol Report YYYY-MM-DD" event bead, then
deletes the source digests.

`runPatrolDigest` (`patrol.go:90-183`):

1. Resolves target date: `--date YYYY-MM-DD` parses via
   `time.Parse("2006-01-02", ...)`, or `--yesterday` uses
   `time.Now().UTC().AddDate(0, 0, -1)`. Neither set → error
   (`patrol.go:94-106`). UTC is used because Dolt stores
   timestamps in UTC (gt-ty4 comment at `patrol.go:101-102`).
2. Idempotency: `findExistingPatrolDigest(dateStr)` queries recent
   event beads via `bd list --type=event --json --limit=50` and
   looks for `Title == "Patrol Report <dateStr>"`. Match → exit
   silently with the existing ID (`patrol.go:111-121, 328-358`).
3. `queryPatrolDigests(targetDate)` shells out to
   `bd list --status=closed --label=digest --json --limit=0`
   (`patrol.go:186-251`), filters to ephemeral beads whose title
   starts with `"Digest: mol-"` and whose `CreatedAt.UTC()` date
   matches, and extracts a `Role` from the title via
   `extractPatrolRole` (`patrol.go:257-271`) which parses
   `"mol-<role>-patrol"` → `<role>`.
4. Builds a `PatrolDigest` struct with per-role counts (`patrol.go:135-144`).
5. `--dry-run` → print preview and return (`patrol.go:146-159`).
6. `createPatrolDigestBead(digest)` (`patrol.go:274-324`) creates a
   non-ephemeral `--type=event` bead with category
   `patrol.digest`, an embedded JSON payload, a markdown
   description, and then auto-closes it with reason
   `daily patrol digest`.
7. `deletePatrolDigests(targetDate)` batches all source digest IDs
   and runs `bd delete --force <ids...>` (`patrol.go:361-386`).

Flags (set in `init()` at `patrol.go:65-68`):

- `--yesterday` — digest yesterday's cycles
- `--date YYYY-MM-DD` — explicit target date
- `--dry-run` — preview without creating
- `--verbose` / `-v` — verbose output

### Data shapes

`PatrolDigest` and `PatrolCycleEntry` (`patrol.go:72-87`):

```go
type PatrolDigest struct {
    Date        string              `json:"date"`
    TotalCycles int                 `json:"total_cycles"`
    ByRole      map[string]int      `json:"by_role"` // deacon, witness, refinery
    Cycles      []PatrolCycleEntry  `json:"cycles"`
}

type PatrolCycleEntry struct {
    ID          string    `json:"id"`
    Role        string    `json:"role"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    CreatedAt   time.Time `json:"created_at"`
    ClosedAt    time.Time `json:"closed_at,omitempty"`
}
```

## Related

- [doctor](doctor.md) — the `PatrolMoleculesExist`, `PatrolHooksWired`,
  `PatrolNotStuck`, `PatrolPluginsAccessible`, `PatrolPluginDrift`
  check batch (`doctor.go:218-226`) verifies patrol system health;
  `gt patrol` is the operational counterpart.
- [feed](feed.md) — feed surfaces patrol lifecycle events via the
  activity/event streams (`🦉 patrol_started`); patrol writes to the
  same streams as a side effect of wisp creation.
- [audit](audit.md) — audit records every patrol create/close as a
  workspace event; `patrol digest` then aggregates those into
  permanent daily summaries.
- [log](log.md) — town log records patrol cycle output when patrol
  wisps write to `logs/town.log`.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Notes / open questions

- `GetRole`, `PatrolConfig`, `autoSpawnPatrol`, `findActivePatrol`,
  `forceCloseDescendants`, and `buildRefineryPatrolVars` all live in
  other files in `internal/cmd/`. A dedicated "patrol machinery"
  note would consolidate the call graph.
- `constants.MolDeaconPatrol`, `MolWitnessPatrol`, `MolRefineryPatrol`
  live in `internal/constants/` — these are the canonical formula
  names used throughout the patrol system.
- `patrol digest` relies on `bd list` returning proper JSON and on
  the `ephemeral` field being present on issues. A schema change
  to `bd` would silently break digest generation.
- The two-stage `queryPatrolDigests` call in `deletePatrolDigests`
  (`patrol.go:363`) re-runs the same query used in `runPatrolDigest`.
  There's a race window where new digests created between the two
  calls would be deleted but not included in the aggregate.
- `extractPatrolRole` only matches `mol-<role>-patrol` titles;
  wisp-style `gt-wisp-*` digests fall through to return `"patrol"`
  (`patrol.go:269-270`). Those cycles lose their per-role counts.
