---
title: gt formula
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/formula.go
  - /home/kimberly/repos/gastown/internal/cmd/formula_overlay.go
  - /home/kimberly/repos/gastown/internal/cmd/formula_overlay_show.go
  - /home/kimberly/repos/gastown/internal/cmd/formula_overlay_edit.go
  - /home/kimberly/repos/gastown/internal/cmd/formula_overlay_list.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, formula, molecule, workflow, bd-wrapper]
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

# gt formula

Manage [workflow formulas](../concepts/formula.md) — reusable
[molecule](../concepts/molecule.md) templates (TOML/JSON files) that
define workflow steps, variables, and composition rules. Parent to
five subcommands: `list`, `show`, `run`, `create`, `overlay`.

**Also known as:** `gt formulas` (alias, `formula.go:40`).
**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`formula.go:41`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/formula.go` (1267
lines). Parent command at `formula.go:38-66` is a pure
`requireSubcommand` router.

### Invocation

```
gt formula <subcommand> [args]
gt formulas <subcommand> [args]    # alias
```

### Concept (from the `Long` help, `formula.go:44-66`)

A **formula** is a TOML or JSON file that defines a workflow with
steps, variables, and composition rules. Formulas can be "poured"
(materialized into a molecule) or "wisped" (ephemeral patrol cycles).

**Search paths** (in order, `formula.go:57-60`):

1. `.beads/formulas/` (project)
2. `~/.beads/formulas/` (user)
3. `$GT_ROOT/.beads/formulas/` (orchestrator)

### Subcommands

Registered in `init()` at `formula.go:179-184`, plus the `overlay`
subtree registered in `formula_overlay.go:34-35`. Five total:

1. **`gt formula list`** (`formulaListCmd`, `formula.go:68-82`)

   Lists formulas from all search paths. Delegates to `bd formula
   list` via `runFormulaList` at `formula.go:188-198` — pure
   passthrough with an optional `--json` forwarded.

2. **`gt formula show <name>`** (`formulaShowCmd`, `formula.go:84-100`)

   Displays detailed information about a formula: metadata,
   variables, steps, composition rules. Delegates to `bd formula
   show <name>` via `runFormulaShow` at `formula.go:201-212` —
   again a passthrough with optional `--json`.

3. **`gt formula run [name]`** (`formulaRunCmd`, `formula.go:102-138`)

   The only non-passthrough subcommand. Pours a formula to create a
   molecule and dispatches the work. `runFormulaRun` at
   `formula.go:217+` handles rig detection, default-formula lookup
   from rig config, PR-based workflows, and (per the comment at
   `formula.go:214-216`) for convoy-type formulas "creates a convoy
   bead, creates leg beads, and slings each leg to a separate
   polecat with leg-specific prompts."

   **Agent precedence** (from `Long`, `formula.go:123-127`, highest
   to lowest):
   1. Per-leg `agent` field in formula TOML.
   2. `--agent` CLI flag.
   3. Formula-level `agent` field in formula TOML.
   4. Rig/town default agent.

4. **`gt formula create <name>`** (`formulaCreateCmd`,
   `formula.go:140-159`)

   Creates a starter formula file in `.beads/formulas/`. The
   `--type` flag selects between three templates: `task` (default —
   single-step), `workflow` (multi-step with dependencies), `patrol`
   (repeating patrol cycle for wisps).

5. **`gt formula overlay`** (`formulaOverlayCmd`,
   `formula_overlay.go:7-32`)

   Manage formula overlays — per-formula step overrides stored as
   TOML files. The parent command is a `requireSubcommand` router
   with three children. Overlays customize formula steps via
   `replace`, `append`, or `skip` modes and are applied at prime
   time when formula steps are displayed.

   **File layout** (from `Long` text, `formula_overlay.go:20-24`):
   - Town-level: `<townRoot>/formula-overlays/<formula>.toml`
   - Rig-level: `<townRoot>/<rig>/formula-overlays/<formula>.toml`

   **Resolution:** if a rig-level overlay exists it takes full
   precedence; town-level is NOT merged
   (`formula_overlay_show.go:66-73`).

   **Children:**

   **5a. `gt formula overlay show <formula>`**
   (`formulaOverlayShowCmd`, `formula_overlay_show.go:14-26`)

   Displays the resolved overlay for a formula with source
   annotation (which file provides it, how many step overrides).
   Loads and validates the overlay via
   `formula.LoadFormulaOverlay(formulaName, townRoot, rigName)`
   (`formula_overlay_show.go:50-53`). Prints the raw TOML file
   content to stdout (`formula_overlay_show.go:81-88`).
   `resolveOverlayContext` at `formula_overlay_show.go:94-109`
   resolves town root and auto-detects rig from cwd.

   Flag: `--rig <name>` (default: auto-detect from cwd).

   **5b. `gt formula overlay edit <formula>`**
   (`formulaOverlayEditCmd`, `formula_overlay_edit.go:13-29`)

   Opens the overlay file in `$EDITOR` (fallback: `vi`). Creates
   the directory and file with a commented template if they do not
   exist (`formula_overlay_edit.go:64-78`). By default edits the
   rig-level overlay; `--town` forces the town-level overlay
   (`formula_overlay_edit.go:52-56`). After editing, validates via
   `formula.LoadFormulaOverlay` and prints a warning on parse
   errors (`formula_overlay_edit.go:96-98`). Changes take effect
   at next `gt prime`.

   Flags: `--rig <name>`, `--town` (bool).

   **5c. `gt formula overlay list`**
   (`formulaOverlayListCmd`, `formula_overlay_list.go:13-23`)

   Lists all overlay files across town and rig levels. Scans the
   town-level `formula-overlays/` directory and every rig's
   `formula-overlays/` directory (detecting rigs by the presence
   of `config.json`, `formula_overlay_list.go:64-65`). Output is a
   tabular listing of scope, formula name, and file path
   (`formula_overlay_list.go:92-96`).

   No flags.

### Flags

Per-subcommand, defined in `init()` at `formula.go:161-185`:

**`list`:**

| flag | type | default | description |
|------|------|---------|-------------|
| `--json` | bool | `false` | Output as JSON |

**`show`:**

| flag | type | default | description |
|------|------|---------|-------------|
| `--json` | bool | `false` | Output as JSON |

**`run`:**

| flag | type | default | description |
|------|------|---------|-------------|
| `--pr` | int | `0` | GitHub PR number to run formula on |
| `--rig` | string | `""` | Target rig (default: current or gastown) |
| `--dry-run` | bool | `false` | Preview execution without running |
| `--agent` | string | `""` | Override agent/runtime for all legs (e.g., gemini, codex, claude-haiku) |
| `--files` | []string | `nil` | Files to pass to formula legs (available as `{{.files}}` in templates) |

**`create`:**

| flag | type | default | description |
|------|------|---------|-------------|
| `--type` | string | `"task"` | Formula type: `task`, `workflow`, or `patrol` |

### Related commands

- [convoy](convoy.md) — `gt formula run` on a convoy-type formula
  creates a convoy bead and slings legs to polecats (`formula.go:214-216`).
  Formulas are one of the primary upstream sources of convoy creation.
- [bead](bead.md) — `list` and `show` subcommands shell out to
  `bd formula list` and `bd formula show`, so formulas are actually
  a bd concept that gt exposes.
- [mq](mq.md) — merge-queue formulas are part of the refinery loop
  that consumes the work formulas dispatch.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Formula file parse errors silently ignored in `formula run`:** `formula.go:633,640` — dependency-add commands via `BdCmd("dep", "add", ...)` at lines 633 and 640 discard errors with `_ = ...Run()`. If the dep-add fails, the tracking relation between convoy and synthesis or between synthesis and leg beads is missing. **Absent** — the workflow runs but convoy tracking is broken silently.
- **Comment on synthesis failure silently ignored:** `formula.go:690` — `_ = commentCmd.Run()` discards the error from adding a comment to a step bead after a sling failure. **Absent** — audit trail gap.

## Notes / open questions

- **`list` and `show` are pure passthroughs** — no gt-specific logic
  above `bd formula list/show`. If these ever acquire gt-local
  features (caching, merging multiple sources), the passthrough
  pattern would need revisiting.
- **`run` is the heavyweight subcommand** — the rest of the
  1267-line file is dominated by `runFormulaRun` and its helpers.
  A deeper pass on the run logic (molecule pouring, leg creation,
  polecat slinging) is a follow-up.
- **`runFormulaCreate` needs a template source** — imports include
  `text/template`, suggesting file-template rendering from baked-in
  strings. Not yet traced.
- **`encoding/base32` + `crypto/rand` imports** (`formula.go:6-8`)
  imply formula IDs are generated client-side with random suffixes.
  Worth confirming.
- **`NoOptDefVal`-style defaults** — unlike [convoy](convoy.md)'s
  `--notify`, formula's flags don't use that pattern; all flags
  require explicit values.
