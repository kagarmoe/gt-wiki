---
title: internal/formula
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/formula/doc.go
  - /home/kimberly/repos/gastown/internal/formula/types.go
  - /home/kimberly/repos/gastown/internal/formula/parser.go
  - /home/kimberly/repos/gastown/internal/formula/embed.go
  - /home/kimberly/repos/gastown/internal/formula/overlay.go
  - /home/kimberly/repos/gastown/internal/formula/variable_validation.go
tags: [package, agent-runtime, formula, molecule, toml, dag, embed, topological-sort, overlay]
---

# internal/formula

The formula package parses, validates, and plans execution for
**formula TOML files** — the static workflow templates that
define what a Gas Town molecule will do when it runs. Four
formula types, dependency-graph validation with cycle detection,
Kahn-algorithm topological sort, ready-step computation, four
composition mechanisms (`extends`, `compose.expand`, step
overrides via overlay files, template variable validation),
embedded-file distribution with integrity tracking, and a
health-check pass that tells operators which formulas are ok /
outdated / modified / missing.

**Go package path:** `github.com/steveyegge/gastown/internal/formula`
**File count:** 6 non-test `.go` files.
`parser.go` (703), `embed.go` (357), `types.go` (235),
`variable_validation.go` (161), `overlay.go` (151),
`doc.go` (128). Plus an embedded `formulas/` directory of
`*.formula.toml` files that ship inside the binary.
**Concept:** [formula concept](../concepts/formula.md) — what
a formula IS in Gas Town, and its relationship to the
[molecule concept](../concepts/molecule.md) (formula = template;
molecule = running instance).
**CLI command:** [`gt formula`](../commands/formula.md) — four
subcommands (`list`, `show`, `run`, `create`). `list`/`show`
are pure passthroughs to `bd formula`; `run` is the
heavyweight path.
**Imports (notable):** `github.com/BurntSushi/toml` (all TOML
decoding); `embed` (for the distribution bundle);
`crypto/sha256` / `encoding/hex` (for install tracking).
**Imported by:** `internal/cmd/formula.go` (the CLI), the
`internal/cmd/sling.go` dispatch path (pouring a formula into a
molecule), `internal/cmd/convoy.go` (convoy-type formulas
create convoys), the Deacon's patrol dispatcher, and the
`internal/rig` package (patrol-molecule seeding).

## What it actually does

Six files. Three concerns: **the data shape and invariants**
(types.go, parser.go, variable_validation.go), **distribution
and update** (embed.go), and **rig/town-local customization**
(overlay.go). `doc.go` is an extensive package-level godoc that
explains the shape of the four formula types in one place.

### types.go — the data model

Source: `/home/kimberly/repos/gastown/internal/formula/types.go`.

**The four formula types** (`types.go:16-25`):

- **`TypeConvoy`** (`"convoy"`) — parallel execution of
  independent **legs** with an optional synthesis step that
  combines the leg outputs. Used for multi-aspect work where
  each leg is a separate polecat in a separate worktree.
- **`TypeWorkflow`** (`"workflow"`) — sequential steps with
  explicit `needs` dependencies forming a DAG. Steps can
  optionally run in parallel by marking `parallel = true` and
  sharing the same `needs`.
- **`TypeExpansion`** (`"expansion"`) — template-based step
  generation. Holds a `[[template]]` list that another
  formula can expand into as part of a `compose.expand` rule.
- **`TypeAspect`** (`"aspect"`) — multi-aspect parallel
  analysis. Similar to convoy but for read-only analysis
  workflows: no code commits expected from aspect legs.

`FormulaType.IsValid()` (`types.go:178-184`) rejects anything
outside this set; `inferType()` in parser.go fills it in from
content when the TOML doesn't set it explicitly.

**The `Formula` struct** (`types.go:28-58`) — a superset of all
type-specific fields:

- **Common:** `Name` (from TOML field `formula`), `Description`,
  `Type`, `Version`, `Pour`, `Agent`, `ReviewOnly`.
- **Convoy-specific:** `Inputs map[string]Input`,
  `Prompts map[string]string`, `Output *Output`, `Legs []Leg`,
  `Synthesis *Synthesis`.
- **Workflow-specific:** `Steps []Step`, `Vars map[string]Var`.
- **Composition:** `Extends []string`, `Compose *ComposeRules`.
- **Expansion:** `Template []Template`.
- **Aspect:** `Aspects []Aspect`.

Key domain flags on `Formula`:

- **`Pour bool`** (`types.go:34`) — when `true`, the steps are
  **materialized as sub-wisps with checkpoint recovery** instead
  of running inline. Default `false` (inline/root-only). This is
  the "poured" vs "inline" distinction that `gt formula run`'s
  help text mentions: a poured formula creates per-step wisps so
  a molecule can recover mid-flight; inline formulas are
  executed all-at-once at the root level. See the
  [molecule concept](../concepts/molecule.md) for how this maps
  to runtime state.
- **`Agent string`** (`types.go:35`) — default agent (claude,
  gemini, codex, etc.) for all legs. Per-leg overrides live on
  `Leg.Agent` (GH#2118).
- **`ReviewOnly bool`** (`types.go:36`) — marks the formula as
  analysis-only: no code commits expected. Legs inherit this
  unless overridden (gt-kvf).

**The component types:**

- `Leg` (`types.go:104-111`) — `ID`, `Title`, `Focus`,
  `Description`, `Agent`, `ReviewOnly`. Focus is the
  single-sentence "what are you looking at" field that drives
  the leg's prompt.
- `Synthesis` (`types.go:114-118`) — the combining step of a
  convoy formula. Has `DependsOn []string` referencing leg IDs;
  validated to only point at existing legs.
- `Step` (`types.go:121-128`) — workflow step with `ID`, `Title`,
  `Description`, `Needs []string`, `Parallel bool`,
  `Acceptance string` (exit criteria for Ralph loop mode).
- `Template` (`types.go:131-137`) — expansion template with
  `Needs` and `Acceptance` (propagated to the generated step).
- `Aspect` (`types.go:80-85`) — `ID`, `Title`, `Focus`,
  `Description`. Simpler than a Leg.
- `Input` (`types.go:88-94`) — formula input parameter: `Type`,
  `Required`, `RequiredUnless []string`, `Default`.
- `Output` (`types.go:97-101`) — where synthesis output gets
  written: `Directory`, `LegPattern`, `Synthesis`.
- `Var` (`types.go:142-175`) — custom TOML unmarshaller: a var
  can be a bare string (used as default value) or a full table
  `[vars.wisp_type] description=... required=... default=...`.

**Composition** (`types.go:61-77`):

- `ComposeRules` has `Expand []*ExpandRule` and a reserved
  `Aspects []string` (future work).
- `ExpandRule` has `Target string` (the step ID to replace) and
  `With string` (the expansion formula to replace it with).

**`GetDependencies`** (`types.go:190-211`) — returns the
dependency list for a step/template/synthesis by ID. For convoy
legs the dependency list is empty (legs are parallel); for
`synthesis` it's `Synthesis.DependsOn`.

**`GetAllIDs`** (`types.go:214-235`) — returns every step/leg/
template/aspect ID in the formula, used by external tools that
need to enumerate work units.

### parser.go — parse, validate, sort, resolve

Source: `/home/kimberly/repos/gastown/internal/formula/parser.go`.

**Parsing:**

- `ParseFile(path)` (`parser.go:14-20`) — read + `Parse`.
- `Parse(data)` (`parser.go:23-37`) — TOML decode, `inferType`,
  `Validate`.

**Type inference** (`parser.go:40-57`): when `type` isn't set,
infer from content. `Extends` or `Steps` ⇒ workflow; `Legs` ⇒
convoy; `Template` ⇒ expansion; `Aspects` ⇒ aspect.

**Validation** (`parser.go:60-210`) is type-specific:

- `validateConvoy` (`:85-121`) — at least one leg; unique IDs;
  `Synthesis.DependsOn` references only existing legs;
  `Input.RequiredUnless` references valid input keys.
- `validateWorkflow` (`:123-156`) — at least one step (OR
  `Extends` is set and steps come from the parent); unique IDs;
  every `Needs` reference points at an existing step; cycle
  detection via `checkCycles`.
- `validateExpansion` (`:158-190`) — at least one template;
  unique IDs; cycle detection via `checkExpansionCycles`.
- `validateAspect` (`:192-210`) — at least one aspect; unique
  IDs.

**Cycle detection** (`checkDependencyCycles`, `:231-270`): DFS
with a visited set and an in-stack set, iterating IDs in sorted
order for deterministic error reporting. Returns
`"cycle detected involving: <id>"`.

**Topological sort** (`TopologicalSort`, `parser.go:275-361`):
Kahn's algorithm for workflow/expansion. Parallel formula types
(convoy, aspect) short-circuit to "all items, original order."
The final length check catches cycles missed by `checkCycles`
(defensive — should not fire in practice).

**Ready-step computation** (`ReadySteps`, `parser.go:365-418`):
Takes a `map[string]bool` of completed IDs and returns the set
of steps whose dependencies are all satisfied. For convoy/aspect
"ready" just means "not completed." This is the core primitive a
runner uses to drive execution.

**Parallel-aware ready steps** (`ParallelReadySteps`,
`parser.go:435-469`): extends `ReadySteps` with the
`Step.Parallel` flag — returns either all parallel-ready steps
for concurrent execution, or a single non-parallel step for
sequential execution.

**Lookup helpers:** `GetStep`, `GetLeg`, `GetTemplate`,
`GetAspect` (`parser.go:421-499`) — linear scans by ID,
returning `nil` for misses.

**Composition: `extends` and `compose.expand`** (`Resolve`,
`parser.go:508-599`):

`Resolve(formula, searchPaths)` recursively merges parent
formulas named in `Extends` and applies `Compose.Expand` rules.

1. **Cycle detection** across the extends chain via a `chain`
   slice passed through recursive calls.
2. **Short-circuit** if no `Extends` and no `Compose`: return
   the formula as-is after validation.
3. **Merge parents in order** (`parser.go:547-569`): load each
   parent via `loadFormulaByName`, recursively resolve it, then
   inherit its `Vars` (child overrides) and `Steps` (parent
   steps come first in the ordering). Parent `Description` is
   used as a fallback.
4. **Apply child vars and steps** (`parser.go:572-581`): child
   vars override inherited ones; child steps are appended after
   parent steps.
5. **Apply compose.expand rules** (`parser.go:583-593`):
   `applyExpandRule` loads the named expansion formula, replaces
   the target step with the expansion's template steps, and
   rewrites any subsequent step's `Needs` that referenced the
   replaced target to point at the last expanded step.
6. **Re-validate** the merged formula before returning.

**`applyExpandRule`** (`parser.go:623-694`):

- Loads the named expansion formula; enforces it's
  `TypeExpansion`.
- Builds new `Step` values from each `Template`, running each
  string field through `expandPlaceholders` which substitutes
  `{target}`, `{target.title}`, `{target.description}` with
  values from the target step being replaced (`parser.go:698-703`).
- The first expanded step inherits the target's own `Needs` if
  the template didn't specify any.
- Rewrites `Needs` on following steps so anything that depended
  on the replaced target now depends on the last step of the
  expansion.

**`loadFormulaByName`** (`parser.go:602-618`): tries the embedded
FS first (`GetEmbeddedFormulaContent`), then walks
`searchPaths`, joining with `name + ".formula.toml"`. Errors
clearly say "not found in embedded FS or search paths."

### embed.go — distribution and health

Source: `/home/kimberly/repos/gastown/internal/formula/embed.go`.

**`//go:embed formulas/*.formula.toml`** (`embed.go:16-17`): the
`formulas/` subdirectory is the source-of-truth set of canonical
formulas, baked into the gt binary. They are provisioned to
`.beads/formulas/` on install and can be updated later.

- `GetEmbeddedFormulaContent(name)` (`embed.go:50-61`): return
  the raw bytes for an embedded formula by name, with or without
  the `.formula.toml` suffix.
- `getEmbeddedFormulas()` (`embed.go:76-94`): `filename →
  sha256` for every embedded formula.

**`InstalledRecord`** (`embed.go:21-23`): JSON file
`.beads/formulas/.installed.json` with a
`map[filename]sha256` tracking what the install-time hash was.
This is the "we installed this" fingerprint — user modifications
are detected as "current file hash doesn't match either the
embedded hash or the installed hash."

**`ProvisionFormulas(beadsPath)`** (`embed.go:139-199`): called
during `gt install`. Creates `.beads/formulas/`, writes every
embedded formula that doesn't already exist, and records the
install-time hash in `.installed.json`. User files are NEVER
overwritten — a pre-existing `.beads/formulas/<name>.formula.toml`
is silently skipped.

**`CheckFormulaHealth(beadsPath)`** (`embed.go:203-272`):
classifies every embedded formula into one of seven states:

- **`ok`** — current hash matches embedded hash.
- **`outdated`** — current matches installed hash, but embedded
  hash has changed (safe to update — user hasn't modified).
- **`modified`** — tracked file whose current hash differs from
  both installed and embedded (user changed it — don't
  overwrite).
- **`missing`** — was installed, now deleted.
- **`new`** — embedded but never installed.
- **`untracked`** — file exists but not in `.installed.json`
  (e.g. from an older gt version — safe to update).
- **`error`** — file could not be read (permission, etc.).

**`UpdateFormulas(beadsPath)`** (`embed.go:277-357`): update
safe-to-update formulas (`outdated`, `missing`, `untracked`),
skip `modified` ones, return counts for each category. This is
the conservative path: a user's customization is preserved even
through a gt binary upgrade.

### overlay.go — rig/town step overrides

Source: `/home/kimberly/repos/gastown/internal/formula/overlay.go`.

Formulas ship as static templates, but some steps need to be
customized per-rig or per-town without forking the formula. The
overlay system lets operators drop a TOML file into
`<townRoot>/formula-overlays/<name>.toml` (town-level) or
`<townRoot>/<rigName>/formula-overlays/<name>.toml` (rig-level)
that replaces, appends to, or skips individual steps of the
named formula.

- `OverrideMode` (`overlay.go:12-21`): `replace`, `append`, or
  `skip`.
- `StepOverride` (`overlay.go:24-28`): `StepID`, `Mode`,
  `Description`.
- `FormulaOverlay` (`overlay.go:31-33`): a slice of
  `StepOverride` under the `[[step-overrides]]` TOML table.

**`LoadFormulaOverlay(formulaName, townRoot, rigName)`**
(`overlay.go:43-62`): **rig-level takes FULL precedence over
town-level** (not merged). If a rig-level overlay exists, only
that one is used. This is different from the directive overlay
pattern, where town and rig are concatenated. The asymmetry is
intentional: formula overlays replace individual steps, and a
rig operator who writes `<rig>/formula-overlays/<name>.toml` is
explicitly overriding the town default.

**`ApplyOverlays(f, overlay)`** (`overlay.go:99-151`): mutates
the formula in place:

- `ModeReplace`: swap the step's description entirely.
- `ModeAppend`: concatenate with a newline separator.
- `ModeSkip`: remove the step and propagate its `Needs` to any
  downstream step that depended on the removed step. This
  preserves dependency semantics — skipping a middle step
  doesn't break the DAG; downstream steps just depend on
  whatever the skipped step depended on.

Returns warnings for overlay entries that reference step IDs
not present in the formula (`stale override`). The caller
surfaces these to the operator without failing the load.

### variable_validation.go — template variable check

Source:
`/home/kimberly/repos/gastown/internal/formula/variable_validation.go`.

Formulas use `{{variable}}` placeholders in step text that are
rendered at runtime. This file validates that every variable
the formula references is declared in `[vars]` or `[inputs]`.

- `variablePattern` (`:12`) — regex matching
  `\{\{([a-zA-Z_][a-zA-Z0-9_]*)\}\}`.
- `ExtractTemplateVariables(text)` (`:17-41`): dedup + sorted
  variable names, excluding Handlebars keywords (`else`,
  `this`, `range`, etc.) via `isHandlebarsKeyword` (`:45-53`).
- **`ValidateTemplateVariables`** (`:63-160`): walks every text
  field of the formula (Description, Steps, Legs, Synthesis,
  Template, Aspects, Prompts, Inputs, Output), extracts all
  `{{var}}` references, and errors if any are undefined. The
  error message nudges the operator to add the missing var
  with `default=""` for computed values.

The bug this guards against is a real one: formulas that
reference computed variables like `{{ready_count}}` without
declaring them in `[vars]` cause `bd mol wisp` to fail with
`missing required variables` at runtime. Declaring them with
`default=""` makes the failure loud and local instead of
silent and distant.

## Notable design choices

- **Four formula types in one struct.** The `Formula` struct is
  a superset rather than a sum type. Type-specific fields are
  empty for the other types. This keeps parser + validator code
  in one place but means consumers must respect the `Type` when
  reading fields.
- **Kahn's algorithm for topo sort.** Deterministic, cycle-aware,
  and the canonical choice for a dependency graph where ties
  should be broken by insertion order. The fallback `cycle
  detected in dependencies` check defends against logic errors
  in the validator.
- **Embedded defaults with hash-tracked updates.** The embed
  pattern means formulas ship with the binary (so a fresh
  install has a working set immediately) but are
  user-overridable (because `.beads/formulas/` files take
  precedence over embedded ones). The `.installed.json` record
  is the memory that separates "user modified" from "we
  updated."
- **Rig-level overlays are full replacement, not merge.** The
  comment at `overlay.go:47-48` is explicit. The trade-off:
  writing a rig overlay implies the operator reviewed the town
  default and decided rig-specific behavior matters. Merging
  would let the operator accidentally inherit town overrides
  they intended to replace.
- **`ModeSkip` preserves dependency semantics.** A naive skip
  would orphan downstream steps that referenced the removed
  step. `ApplyOverlays` walks the step list and rewrites the
  needs so the skip is DAG-safe.
- **Template-variable validation catches a real runtime class
  of bug.** `bd mol wisp` rejects formulas with undefined
  `{{variable}}` references with a cryptic error. Adding
  validation at parse time makes the failure loud at
  authoring time.
- **Convoy formulas are a formula type, not a convoy bead.**
  The word "convoy" is overloaded: a *convoy formula type* is a
  TOML shape with parallel legs; a *convoy bead* is a work
  dispatch bead with tracked issues. A convoy formula can
  create a convoy bead (when `gt formula run` picks up a
  `TypeConvoy` formula), but the two are distinct concepts.
  See the [convoy concept](../concepts/convoy.md).

## Related wiki pages

- [formula concept](../concepts/formula.md) — what a formula
  IS in Gas Town.
- [molecule concept](../concepts/molecule.md) — what a running
  instance of a formula is called, and why the distinction
  matters.
- [`gt formula`](../commands/formula.md) — CLI surface.
- [`gt molecule`](../commands/molecule.md) / `gt mol` — the
  runtime side of molecules.
- [`internal/rig`](rig.md) — `AddRig` seeds patrol molecules
  from formulas. `formula-overlays/` directory lives under
  the rig path.
- [`internal/plugin`](plugin.md) — plugins are a distinct
  concept: plugin.md files with TOML frontmatter are Deacon
  patrol tasks, not molecule templates. Both use TOML, both
  live in directories under the rig, but they're different
  systems.
- [`internal/convoy`](convoy.md) — convoy formulas dispatch
  into convoy beads.
- [convoy-launch workflow](../workflows/convoy-launch.md) —
  `gt formula run` on a convoy-type formula is one of the
  triggers.
- [wisp concept](../concepts/wisp.md) — `Pour=true` formulas
  materialize steps as wisps.
- [`gt install`](../commands/install.md) — calls
  `ProvisionFormulas` during town setup.
- [`gt doctor`](../commands/doctor.md) — `CheckFormulaHealth`
  is one of the health checks it consumes.
- [`internal/beads`](beads.md) — the beads SDK exposes
  `bd formula list/show/run/create` which the CLI wraps.

## Notes / open questions

- **`TypeAspect` is underexplored.** It behaves like convoy
  for parallel execution but is marked analysis-only. Whether
  the difference is enforced at runtime (no merge request, no
  git push) or just metadata is worth confirming.
- **`Pour` = false semantics.** The comment says "inline/root-
  only" but doesn't define what "inline" means at execution
  time. A root-only formula probably executes its steps as a
  single flat set instead of creating per-step wisps. The
  molecule concept page should explain the distinction from
  the runtime side.
- **Composition is one-way.** `extends` means "my steps come
  after my parents' steps in the final ordering." There's no
  insertion point mechanism — a child can't say "insert my
  steps between step A and step B." `compose.expand` is the
  closest thing but operates on an existing step rather than
  injecting a sibling.
- **`compose.aspects` is a parked field.** The TOML reads it
  but `resolveChain` doesn't act on it (`parser.go:594`). The
  comment says "future work."
- **`TemplateNeeds == nil` inherits target.** The first
  expanded step's behavior when its template's `Needs` is
  empty is subtle: it inherits the target step's own needs so
  the expansion slots into the DAG where the target was.
  Non-first expanded steps with empty `Needs` *don't* inherit
  anything — they become roots of the expansion's internal
  DAG. This may or may not be the intended behavior for
  multi-step expansions.
- **`.installed.json` drift.** If two gt binaries of different
  versions run against the same `.beads/formulas/`, the older
  one sees some formulas as `untracked` that the newer version
  installed. Safe to update in both directions, but the
  classification depends on which binary last wrote
  `.installed.json`.
- **No formula-level test exists for
  `ValidateTemplateVariables` + overlays interaction.** An
  overlay that replaces a step with new `{{variable}}`
  references might introduce undefined variables; overlay
  validation doesn't re-run template variable checks.
