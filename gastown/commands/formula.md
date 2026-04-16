---
title: gt formula
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/formula.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, formula, molecule, workflow, bd-wrapper]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [incomplete]
---

# gt formula

Manage [workflow formulas](../concepts/formula.md) — reusable
[molecule](../concepts/molecule.md) templates (TOML/JSON files) that
define workflow steps, variables, and composition rules. Parent to
four subcommands: `list`, `show`, `run`, `create`.

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

Registered in `init()` at `formula.go:179-184`. Four total:

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

## Coverage gaps

*Phase 4 audit (2026-04-16):* The `overlay` subcommand tree is entirely
missing from this page. Source: `formula_overlay.go` (parent),
`formula_overlay_edit.go`, `formula_overlay_list.go`,
`formula_overlay_show.go` (347 lines total). The page says "four
subcommands" but there are five top-level registrations (list, show, run,
create, overlay), and overlay itself has three children (show, edit, list).
The overlay feature manages per-formula step overrides via TOML files at
town-level and rig-level paths.

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
