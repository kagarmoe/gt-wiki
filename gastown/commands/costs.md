---
title: gt costs
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/costs.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, claude, cost-tracking, beads-exempt]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt costs

Reports Claude Code session costs by scanning transcript files and
applying per-model pricing. Has three modes: live (default), ledger
query, and append-only record/digest.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/costs.go:45-68`)
**Beads-exempt:** **yes** — `costs` is in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:61`; the beads check
is skipped for this command so `costs record` can run from a session
Stop hook without requiring `bd`.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/costs.go:45-end`.

### Invocation

```
gt costs [--today|--week|--by-role|--by-rig|--json|-v]
gt costs record --session <name> [--work-item <bead>]
gt costs digest [--yesterday|--date YYYY-MM-DD] [--dry-run]
```

### Behavior

#### `gt costs` (default — live view)

`runCosts` (`costs.go:216-224`) branches:

- If any of `--today/--week/--by-role/--by-rig` is set, calls
  `runCostsFromLedger` (`costs.go:295-373`), which queries beads events
  and/or ephemeral wisps.
- Otherwise calls `runLiveCosts` (`costs.go:226-293`), which:
  1. Lists tmux sessions via `tmux.NewTmux().ListSessions()`
     (`costs.go:229-233`).
  2. Filters to known Gas Town sessions via `session.IsKnownSession`
     (`costs.go:239-242`).
  3. Parses session name to `role/rig/worker` via `parseSessionName`
     (`costs.go:244`).
  4. For each session, looks up its working directory
     (`getTmuxSessionWorkDir`) and scans that path for a Claude Code
     transcript file (`extractCostFromWorkDir`, `costs.go:246-264`).
  5. Checks whether the agent appears to be running
     (`t.IsAgentRunning`, `costs.go:266-267`).
  6. Totals across sessions and emits JSON or human output
     (`costs.go:281-292`).

Transcript files live under `$CLAUDE_CONFIG_DIR/projects/` (default
`~/.claude/projects/`) per the Long description (`costs.go:50-54`).
Token counts come from `TranscriptUsage` structs
(`costs.go:182-196`) and are priced against a `modelPricing` map
keyed by model ID (`costs.go:200-214`): Opus 4.5, Sonnet 4, Haiku 3.5,
and a Sonnet-priced `"default"` fallback.

#### `gt costs record` — called by Claude Code Stop hook

`costsRecordCmd` (`costs.go:70-89`). From the Long text
(`costs.go:72-87`): reads token usage from the Claude Code transcript
and appends a cost entry to `~/.gt/costs.jsonl`. It is an append
operation that "never fails due to database availability" — which is
why `costs` is beads-exempt.

Flags (`costs.go:121-122`): `--session <name>` and `--work-item <bead>`.

#### `gt costs digest` — called by Deacon patrol

`costsDigestCmd` (`costs.go:91-108`). Aggregates entries from
`~/.gt/costs.jsonl` for a target date, creates a permanent
"Cost Report YYYY-MM-DD" digest bead, then removes the source log
entries. Flags (`costs.go:126-128`):

- `--yesterday` — digest yesterday's costs (default for patrol)
- `--date YYYY-MM-DD` — digest a specific date
- `--dry-run` — preview without making changes

#### `runCostsFromLedger`

`costs.go:295-373`. Query rules:

- `--today` → `querySessionCostEntries(now)` (today's ephemeral wisps,
  `costs.go:300-306`)
- `--week` → `queryDigestBeads(7)` plus today's wisps
  (`costs.go:307-317`)
- `--by-role`/`--by-rig` without time filter → default to today
  (`costs.go:318-324`)
- Otherwise → `querySessionEvents` scans both town-level and all
  per-rig beads locations for `session.ended` events
  (`costs.go:405-end`).

### Data shapes

From `costs.go:132-161`:

```go
type SessionCost struct { Session, Role, Rig, Worker string; Cost float64; Running bool }
type CostEntry   struct { SessionID, Role, Rig, Worker string; CostUSD float64; StartedAt, EndedAt time.Time; WorkItem string }
type CostsOutput struct { Sessions []SessionCost; Total float64; ByRole, ByRig map[string]float64; Period string }
```

### Flags (top level)

`costs.go:112-118`:

| Flag              | Type   | Default | Purpose                                     |
|-------------------|--------|---------|---------------------------------------------|
| `--json`          | bool   | `false` | Output as JSON                              |
| `--today`         | bool   | `false` | Today's total from session events           |
| `--week`          | bool   | `false` | This week's total from session events       |
| `--by-role`       | bool   | `false` | Breakdown by role                           |
| `--by-rig`        | bool   | `false` | Breakdown by rig                            |
| `--verbose` / `-v`| bool   | `false` | Debug output for failures                   |

### Subcommands

- `record` (`costs.go:70-89`) — append session cost to
  `~/.gt/costs.jsonl` (Stop-hook fast path)
- `digest` (`costs.go:91-108`) — aggregate the log into a daily bead

## Related

- [feed](feed.md) — lives under the same Diag group and feeds the
  refinery/merge events that also get priced out via `cost_usd` payload.
- [metrics](metrics.md) — reads `~/.gt/cmd-usage.jsonl`; `costs` reads
  `~/.gt/costs.jsonl`. Both are local JSONL logs under `~/.gt/`.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents the
  full `beadsExemptCommands` list including `costs`.
- [README.md](README.md) — command tree index.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Session cost extraction failures silently zero out:**
  `extractCostFromWorkDir` errors at `costs.go:258-264` are printed
  only when `--verbose` is set. Without `-v`, sessions with failed
  transcript parsing appear with `$0.00` cost and no indication of
  failure. **Absent** — errors are hidden by default.
- **Session workdir lookup failures silently skip:** `getTmuxSessionWorkDir`
  errors at `costs.go:249-253` cause the session to be omitted from
  the report entirely (only visible with `-v`). **Absent.**
- **Week query swallows today's errors:** `querySessionCostEntries`
  error in the `--week` path at `costs.go:316` is discarded with
  `todayEntries, _ := ...`. Today's costs are silently missing from the
  weekly total. **Absent.**

## Notes / open questions

- `parseSessionName`, `getTmuxSessionWorkDir`, `extractCostFromWorkDir`,
  `outputCostsHuman`, `outputLedgerHuman`, `querySessionCostEntries`,
  `queryDigestBeads`, and `querySessionEventsFromLocation` are all in
  this file below the read window — worth a second pass for the detailed
  extraction logic.
- Pricing is hard-coded and dated "as of Jan 2025" in the source comment
  (`costs.go:198-199`), but the model IDs reference 2025/2026 releases.
  The table may drift behind actual Anthropic pricing.
- `~/.gt/costs.jsonl` is a local, per-developer file; multi-machine
  installations will not see each other's pre-digest logs.
- `CLAUDE_CONFIG_DIR` env var is honored for transcript discovery
  (`costs.go:51-53`) but the resolution code is in the helpers below
  the read window.
