---
title: gt synthesis
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/synthesis.go
  - /home/kimberly/repos/gastown/internal/formula/
tags: [command, work, synthesis, convoy, formula, legs]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt synthesis

Manage synthesis steps for convoy formulas — the final step that
combines outputs from all parallel convoy legs into a unified
deliverable. Checks leg completion, collects leg outputs, creates
a synthesis bead, and slings it to a target polecat.

**Parent:** [gt](../binaries/gt.md) (root command), alias `synth`
**Group:** `GroupWork` ("Work Management") (`synthesis.go:31`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/synthesis.go`
(713 lines). Uses `internal/formula` to parse formula files and
shells to `bd show/create/dep` for bead operations.

### Invocation

```
gt synthesis <subcommand> <convoy-id> [flags]
gt synth <subcommand> <convoy-id> [flags]    # alias
```

`synthesisCmd.RunE = requireSubcommand` (`:33`).

### Subcommands

Registered at `synthesis.go:102-104`:

| subcommand | source | one-liner |
|---|---|---|
| `start <convoy-id>` | `:50-68`, `runSynthesisStart :131-241` | Verify legs complete, collect outputs, create synthesis bead, sling to polecat. |
| `status <convoy-id>` | `:70-82`, `runSynthesisStatus :244-314` | Show synthesis readiness and leg outputs. |
| `close <convoy-id>` | `:84-92`, `runSynthesisClose :317-371` | Close a convoy after synthesis is complete. |

### `start` pipeline (`runSynthesisStart`, `synthesis.go:131-241`)

1. `getConvoyMeta(convoyID)` (`:135`, helper at `:374-440`) — reads
   `bd show <id> --json`, extracts title/status plus
   `formula`/`formula_path`/`review_id` parsed from description
   lines. Fetches tracked leg issues via `getTrackedIssues`.
2. **Load formula.** Prefer `meta.FormulaPath` → `formula.ParseFile`;
   otherwise `findFormula(meta.Formula)` (`:621-653`) which searches
   `.beads/formulas`, `~/.beads/formulas`, `$GT_ROOT/.beads/formulas`
   for `<name>.formula.{toml,json}`.
3. `collectLegOutputs(meta, f)` (`:443-501`) — for each tracked leg
   issue, fetch `getIssueDetails(legID)` and mark `allComplete =
   false` unless status is `closed`. If the formula defines output
   paths, also scan `expandOutputPath(f.Output.Directory,
   f.Output.LegPattern, meta.ReviewID, leg.ID)` for each leg and
   embed file contents.
4. Refuse unless `allComplete || synthesisForce` (`:175-186`).
5. **Determine review ID** (`:188-195`): `--review-id` flag →
   `meta.ReviewID` → strip `hq-cv-` prefix from convoy ID.
6. **Determine target rig** (`:197-210`): `--rig` flag →
   `findCurrentRig(townRoot)` → fallback `"gastown"`.
7. `--dry-run` → print what would happen and return (`:212-222`).
8. `createSynthesisBead` (`:515-608`) — builds a description with
   convoy ID, review ID, synthesis instructions (from
   `f.Synthesis.Description`), all leg outputs, and a destination
   output path; guards against flag-like titles via
   `beads.IsFlagLikeTitle` (`:560-562`); creates via `bd create
   --type=task --title ... --description ... --json`; adds a
   `tracks` dep from the convoy to the synthesis bead (non-fatal
   if it fails, `:602-605`).
9. `slingSynthesis(synthesisID, targetRig)` (`:611-618`) — shells
   to `gt sling <bead> <rig>`.

### `status` (`runSynthesisStatus`, `synthesis.go:244-314`)

Reads convoy meta, loads formula (best-effort), collects leg
outputs, and prints:

- Convoy header (emoji 🚚, ID, title, status).
- Formula name if set.
- Per-leg status with `✓`/`○` and optional `(output: ✓)` if an
  output file exists.
- Synthesis readiness: `Ready - all legs complete` with a
  `gt synthesis start <id>` hint, or `Waiting - X/Y legs complete`.
- Synthesis config from the formula: title, output path.

### `close` (`runSynthesisClose`, `synthesis.go:317-371`)

1. Read convoy via `bd show <id> --json`.
2. `ensureKnownConvoyStatus(status)` (helper in convoy.go).
3. Idempotent: if already `convoyStatusClosed`, report and return.
4. Shell `bd close <id> --reason="synthesis complete"`, appending
   `--session=<id>` from `runtime.SessionIDFromEnv()` when set.
5. A TODO comment (`:367-368`) notes a planned notification hook:
   parse `Notify: <address>` from description and send mail.

### Exported helpers

- `CheckSynthesisReady(convoyID)` (`:657-665`) — returns
  `allComplete` from a zero-formula `collectLegOutputs`. Public API.
- `TriggerSynthesisIfReady(convoyID, targetRig)` (`:669-713`) —
  convenience wrapper: if ready, re-does meta fetch + formula load +
  `createSynthesisBead` + `slingSynthesis`. Doc comment says this
  is meant to be called by the witness when a leg completes.

### Flags (on `synthesisStartCmd` only, `:95-99`)

| flag | default | notes |
|---|---|---|
| `--rig <name>` | `""` | Target rig for synthesis polecat (default: current). |
| `--dry-run` | `false` | Preview. |
| `--force` | `false` | Start even if some legs incomplete. |
| `--review-id <id>` | `""` | Override review ID for output paths. |

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Tracking dep-add silently discarded:** `synthesis.go:605` — `_ = depCmd.Run()` discards the error from adding the convoy-tracks-synthesis dependency. If this fails, the synthesis bead exists but is disconnected from the convoy tracking graph. **Absent** — convoy status display won't show the synthesis step.
- **Formula parse errors silently ignored:** `synthesis.go:255,258` — `f, _ = formula.ParseFile(...)` discards parse errors. If the formula file is malformed, the synthesis proceeds without formula guidance (leg naming, ordering, etc. fall back to defaults). **Absent** — synthesis runs without formula context, potentially producing unexpected structure.

## Notes / open questions

- **Unclear at first glance, structural once read.** "Synthesis"
  is specifically the merge step for a multi-leg convoy — think
  literature review with parallel reviewers producing summaries
  that a synthesizer then combines. Not related to software
  synthesis, schema generation, or anything toolchain-y.
- **Formula `Output` schema.** The code references
  `f.Output.Directory`, `f.Output.LegPattern`, `f.Output.Synthesis`
  with `{{review_id}}` and `{{leg.id}}` templates. Documenting the
  formula TOML/JSON shape would be a good separate page (flagged for
  the eventual `../data-type/` or `../config/` folder — not here).
- **Convoy-description parsing is fragile.** `getConvoyMeta`
  (`:374-440`) scans description lines for `formula:`/
  `formula_path:`/`review_id:` keys. Any description formatting
  change could break formula linkage.
- **Three paths into `createSynthesisBead`** — explicit `start`,
  `TriggerSynthesisIfReady`, and any future witness hook. The title
  safety guard against flag-like titles (`:560-562`) suggests this
  has bitten before (probably `--force` etc.).
- **Related commands.** [convoy](convoy.md) (the tracking unit),
  [formula](formula.md) (defines the leg + synthesis schema),
  [sling](sling.md) (dispatches the synthesis bead),
  [close](close.md) (convoy closure side), [changelog](changelog.md)
  (the sibling "what did we just land" output surface).
