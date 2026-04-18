---
title: gt compact
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/compact.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, wisp, ttl, bd-wrapper, compaction]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt compact

Apply TTL-based compaction policy to ephemeral wisps: promote
stuck/valued ones to permanent beads, delete expired closed ones, and
sweep orphaned `wisp_dependencies` rows.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`compact.go:55`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/compact.go:53-485`.
This is a wisp-level compactor; it operates on ephemeral beads only.
**It is distinct from `gt dolt compact`** (Dolt database compaction,
the data-plane operation referenced in
[CLAUDE.md](../../CLAUDE.md)) and from `bd delete` in general.

### Invocation

```
gt compact [--dry-run] [--verbose|-v] [--json] [--rig <name>]
```

### Behavior

`runCompact` (`compact.go:176-277`):

1. **Get cwd and resolve town root** via `beads.FindTownRoot(workDir)`
   (`compact.go:179-186`).
2. **Determine rig** from `--rig` flag, falling back to `GT_RIG` env
   var (`compact.go:187-190`).
3. **Load TTLs** via `loadTTLConfig(townRoot, rigName)`. Three layers
   (`compact.go:86-155`):
   - Layer 1: hardcoded defaults in `defaultTTLs` map
     (`compact.go:26-35`): `heartbeat`/`ping` 6h, `patrol`/`gc_report`
     24h, `recovery`/`error`/`escalation` 7d, `default` 24h.
   - Layer 2: rig wisp config — `wisp.NewConfig(...).Get("wisp_ttl")`
     reads `map[string]interface{}` and parses string durations via
     `time.ParseDuration`.
   - Layer 2b: rig identity bead labels matching `wisp_ttl_*:value`
     via `beads.ParseWispTTLKey` and `beads.RigBeadIDWithPrefix("gt",
     rigName)` (`compact.go:131-155`).
4. **List wisps** via `listWisps(bd)` → `bd list --json --all -n 0`
   (`compact.go:194-199`, `compact.go:302-329`). Stdout is passed
   through `extractJSONArray` (`compact.go:335-341`) to strip any
   non-JSON prefix before the `[` character — this is a tolerance
   hack for bd emitting warnings before the JSON array, which broke
   parsing when wisp titles contained emoji.
5. **Filter to ephemeral only** (`compact.go:321-326`).
6. **Decision loop** (`compact.go:207-258`) for each wisp:
   - Compute `age` via `wispAge` (uses `UpdatedAt` or `CreatedAt`,
     `compact.go:471-484`).
   - `ttl` via `getTTL(ttls, w.WispType)` — falls back to `"default"`.
   - `shouldPromote = hasComments(w) || hasKeepLabel(w)` — comments
     or a `keep`/`gt:keep` label means this wisp proved valuable and
     should be saved.
   - `isMoleculeStep = w.Parent != ""` — wisps that are steps of a
     molecule are **never** promoted.
   - **Non-closed wisps** (`compact.go:222-243`):
     - Promote if `shouldPromote && !isMoleculeStep`.
     - Else if past TTL: promote with reason `"open past TTL"` or
       `"stuck in_progress past TTL"`, or (if molecule step) delete
       with `"molecule step past TTL"`.
     - Else skip.
   - **Closed wisps** (`compact.go:244-257`):
     - Promote if `shouldPromote && !isMoleculeStep`.
     - Else if past TTL: delete with `"TTL expired"`.
     - Else skip.
7. **Clean orphaned `wisp_dependencies`** unless `--dry-run`
   (`compact.go:264-266`, `compact.go:283-298`). Runs a single bd SQL
   statement:

   ```sql
   DELETE FROM wisp_dependencies WHERE
     NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.issue_id)
     OR NOT EXISTS (SELECT 1 FROM wisps WHERE id = wisp_dependencies.depends_on_id)
   ```

   Parses `"OK, N rows affected"` to populate
   `result.OrphanedWispDeps`.
8. **Emit results** — JSON via `json.NewEncoder(os.Stdout)` or
   pretty-printed summary via `printCompactSummary`
   (`compact.go:269-276`, `compact.go:402-435`).

### Promote vs delete (`promoteWisp`, `deleteWisp`)

- **Promote** (`compact.go:344-372`): `bd update <id> --persistent`
  (sets `ephemeral=false`), followed by `bd comment <id> "Promoted
  from Level 0: <reason>"`. The comment is best-effort (errors
  ignored).
- **Delete** (`compact.go:375-400`): `bd delete <id> --force`. The
  comment notes this is safe because "Dolt AS OF preserves history".

Both skip their side-effect entirely in `--dry-run` mode.

### Subcommands

None.

### Flags

Defined in `init()` at `compact.go:77-84`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--dry-run` | — | bool | `false` | Preview compaction without making changes |
| `--verbose` | `-v` | bool | `false` | Show each wisp decision |
| `--json` | — | bool | `false` | Output results as JSON |
| `--rig` | — | string | `""` | Compact a specific rig (default: current rig) |

### Result shape (`compactResult`, `compact.go:38-51`)

```go
type compactResult struct {
    Promoted         []compactAction
    Deleted          []compactAction
    Skipped          int
    OrphanedWispDeps int
    Errors           []string
}
type compactAction struct {
    ID, Title, Reason, WispType string
}
```

### Related commands

- [changelog](changelog.md) — filters out wisps/molecule steps from
  its output since those are `gt compact`'s territory.
- [close](close.md) — sibling bd-wrapper; closing a wisp doesn't
  delete it, `gt compact` does the eventual sweep.
- [cleanup](cleanup.md) — unrelated: that's for orphan Claude
  processes. The `compact` / `cleanup` / `dolt cleanup` split is a
  known naming ambiguity.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Promotion comment failure swallowed:** `compact.go:365` — after promoting a wisp via `bd update --persistent`, the follow-up `bd comment` at `compact.go:365` discards its error with `_, _ = bd.Run(...)`. The promotion succeeds but the audit trail comment is silently lost. **Absent** — no indication the comment failed.
- **Orphaned wisp_deps cleanup error collected but not fatal:** `compact.go:289-290` — the SQL DELETE for orphaned wisp dependencies appends to `result.Errors` on failure but doesn't stop compaction. **Present** — error surfaces in JSON/summary output.
- **Rig bead TTL override silently ignored on error:** `compact.go:137-139` — `bd.Show(rigBeadID)` failure returns early with no warning, falling back to defaults. **Absent** — custom TTL overrides silently don't apply if the rig bead is unreachable.

## Notes / open questions

- **`isReferenced` is defined but not consulted.** `compact.go:456-458`
  defines a helper that checks `DependentCount` and `DependencyCount`,
  but `runCompact` never calls it. Dead code, or pending wiring.
- **`WISP-COMPACTION-POLICY.md` design doc** is referenced on
  `compact.go:26` ("Default TTLs per wisp type (from design doc ...)").
  Not yet ingested into the wiki.
- **Bd SQL for the orphan sweep** — the command embeds a raw SQL
  query and parses bd's `"OK, N rows affected"` output format. This
  couples `gt compact` to a specific bd output string; any change in
  bd's SQL wrapper output would silently zero out `OrphanedWispDeps`
  without failing (the `Sscanf` error is ignored).
- **Promotion comment is best-effort.** `promoteWisp` ignores the
  error from the `bd comment` call (`compact.go:364`). So a promoted
  wisp might end up without the audit-trail comment if bd's comment
  subsystem flakes.
