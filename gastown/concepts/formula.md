---
title: formula (concept)
type: concept
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/formula/doc.go
  - /home/kimberly/repos/gastown/internal/formula/types.go
  - /home/kimberly/repos/gastown/internal/formula/parser.go
  - /home/kimberly/repos/gastown/internal/cmd/formula.go
tags: [concept, formula, molecule, toml, template, static, workflow-definition]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
---

# formula

A **formula** is a TOML file that declaratively describes a
multi-step workflow for Gas Town agents. It defines what the
work is (steps, legs, aspects, templates), what inputs the
work takes, what variables substitute into the work's text,
what agent runs it, what dependencies exist between
sub-units, and — crucially — how the work should be shaped
when it runs (parallel, sequential, synthesized, expanded).

A formula is **a template**. It is static data on disk. When
that template is executed — "run" — it produces a **molecule**,
which is a running bead-tracked instance of the formula. The
formula/molecule split is the same as class/instance in an
object-oriented language: the formula is the definition, the
molecule is the thing that actually runs.

> **Formula is to molecule what a class is to an instance.**

A formula lives on disk. A molecule lives as a bead (or a tree
of wisps) in the beads database, reflecting execution state.

See [molecule concept](molecule.md) for the runtime side of
this pair.

## Why formulas exist

Gas Town agents are not general-purpose; they are
bead-driven workers following repeatable patterns. A
"security audit" is not a free-form job — it is a
convoy of parallel analyses (SAST, dependency audit,
secrets scan, ...) synthesized into a report. A "release"
is a workflow with test → build → publish steps in that
order. A patrol cycle is a loop of checks executed on a
schedule. These patterns recur across rigs and across
agents, and each pattern has a shape that can be captured
in data instead of code.

Before formulas, agents would need per-workflow code paths:
a `gt security-audit` command, a `gt release` command, etc.
Formulas let operators define new workflows by writing a
TOML file and dropping it in `.beads/formulas/`. No code
changes. The shape of the workflow lives in the data; the
runner (molecule executor) is a single generic engine.

## The four formula types

Formula types are declared in
`/home/kimberly/repos/gastown/internal/formula/types.go:16-25`:

- **`convoy`** — parallel legs with an optional synthesis
  step. Used when a task breaks down into independent
  analyses that later combine. Each leg becomes a separate
  polecat worktree.
- **`workflow`** — sequential steps with explicit
  dependencies forming a DAG. Steps can optionally share a
  `needs` set and mark themselves `parallel = true` to run
  concurrently. The default shape for "do this, then that."
- **`expansion`** — a template for step generation.
  Expansion formulas do not run directly; they are consumed
  by a `compose.expand` rule in another formula, which
  replaces one of its steps with the expansion's template
  steps.
- **`aspect`** — multi-aspect parallel analysis, like
  convoy but for read-only work: no code commits are
  expected from aspect legs. Used for review-only
  workflows.

The type can be omitted from TOML; `inferType`
(`parser.go:40-57`) fills it in based on content: `extends`
or `steps` ⇒ workflow, `legs` ⇒ convoy, `template` ⇒
expansion, `aspects` ⇒ aspect.

## Anatomy of a formula file

From the package doc
(`/home/kimberly/repos/gastown/internal/formula/doc.go`), a
typical workflow formula:

```toml
formula = "release"
type = "workflow"
description = "Build, test, and publish."

[[steps]]
id = "test"
title = "Run Tests"

[[steps]]
id = "build"
title = "Build"
needs = ["test"]

[[steps]]
id = "publish"
title = "Publish"
needs = ["build"]
```

And a convoy formula:

```toml
formula = "security-audit"
type = "convoy"

[[legs]]
id = "sast"
title = "Static Analysis"
focus = "Find code vulnerabilities"

[[legs]]
id = "deps"
title = "Dependency Audit"
focus = "Check for vulnerable dependencies"

[synthesis]
title = "Combine Findings"
depends_on = ["sast", "deps"]
```

Formulas can also declare:

- `[inputs]` — user-supplied parameters with `type`,
  `required`, `required_unless`, `default`.
- `[vars]` — variables for `{{template}}` substitution in
  step text. Either a bare `name = "default"` shorthand or
  a full `[vars.name]` table.
- `[prompts]` — named prompt templates used by legs.
- `[output]` — where synthesis output gets written.
- `extends = [...]` — parent formulas to inherit steps from.
- `[compose.expand]` — replace a target step with the
  template steps from an expansion formula.
- `pour = true` — materialize the steps as sub-wisps with
  checkpoint recovery (see the `Pour` flag in types.go).
- `agent = "claude"` — default agent for all legs.
- `review_only = true` — analysis-only, no code commits.

## Where formulas live

Three search paths, checked in order (from the
`gt formula` parent command's Long help,
`formula.go:57-60`):

1. **`.beads/formulas/`** — project/rig-level.
2. **`~/.beads/formulas/`** — user-level.
3. **`$GT_ROOT/.beads/formulas/`** — orchestrator/town-level.

Additionally, the
[`internal/formula` package](../packages/formula.md) embeds
a canonical set of formulas into the gt binary via
`//go:embed formulas/*.formula.toml`
(`embed.go:16-17`), so a fresh install has a working set
immediately. `gt install` provisions them into
`.beads/formulas/` via `ProvisionFormulas`
(`embed.go:139-199`), respecting existing files.

Rig-level **formula overlays** at
`<rig>/formula-overlays/<name>.toml` can replace, append
to, or skip individual steps of a formula without forking
the whole file. Town-level overlays at
`<townRoot>/formula-overlays/<name>.toml` exist too; when
both are present for a formula, the rig overlay takes full
precedence (not merged). See `overlay.go:43-62`.

## Lifecycle of a formula

A formula has a simple authoring lifecycle (the runtime is
what the [molecule](molecule.md) page covers):

1. **Author** — write a `.formula.toml` file in
   `.beads/formulas/` (or edit an embedded default).
2. **Parse** — `formula.ParseFile` reads + decodes TOML.
3. **Infer type** (`parser.go:40-57`) — fill `type` from
   content if unset.
4. **Validate** — type-specific validation rules
   (`parser.go:60-210`) check required fields, unique IDs,
   valid dependency references, and cycle-free dependency
   graphs.
5. **Resolve** — `formula.Resolve(f, searchPaths)`
   (`parser.go:508-599`) merges parent formulas from
   `extends` and applies `compose.expand` rules.
6. **Template-variable validation** — optional
   `ValidateTemplateVariables`
   (`variable_validation.go:63-160`) checks that every
   `{{var}}` reference is declared.
7. **Topological sort** — `TopologicalSort`
   (`parser.go:275-361`) produces execution order using
   Kahn's algorithm.
8. **Run** — `gt formula run <name>` pours the formula
   into a molecule and dispatches the work.

## Composition

A formula can build on other formulas in two ways:

- **Extends** — `extends = ["parent1", "parent2"]` at the
  top level causes `Resolve` to merge parent steps before
  the child's own steps. Child `vars` override parent
  `vars`; child description takes precedence; parents are
  merged in the order listed. Cycles in the extends chain
  are detected.
- **Compose.expand** — `[[compose.expand]]` blocks replace
  a named target step with the template steps of a named
  expansion formula. The first expanded step inherits the
  target's own needs; downstream steps that depended on the
  target are rewritten to depend on the last expanded step.
  Placeholders `{target}`, `{target.title}`,
  `{target.description}` are substituted from the replaced
  step.

Both are implemented in
`internal/formula/parser.go:508-703`.

## How formulas map to molecules

When `gt formula run <name>` executes, the flow is:

1. The formula is parsed and resolved.
2. A **convoy bead** is created (if the formula is a
   convoy type) plus per-leg beads. For workflow formulas,
   a molecule root bead is created with per-step wisps if
   `pour = true`.
3. The work is dispatched to polecats via `gt sling`. The
   per-leg agent (from `Leg.Agent` → `--agent` → formula
   `Agent` → rig default, per the precedence in the
   `formula.go:123-127` Long text) drives the spawn.
4. The molecule — the running instance — tracks execution
   state in beads while the formula file remains unchanged.

The formula stays static; the molecule is everything that
happens at runtime. Multiple molecules can exist
simultaneously from the same formula (two operators running
`gt formula run release` produce two independent molecules).

## Relationships with other concepts

- **[molecule](molecule.md)** — the running instance of a
  formula. Every molecule has exactly one formula it was
  poured from.
- **[wisp](wisp.md)** — when `pour = true`, each step
  becomes a wisp in the beads database so the molecule can
  checkpoint and recover.
- **[rig](rig.md)** — formulas are loaded from a rig's
  search paths; rig-level formula overlays live under the
  rig.
- **[convoy](convoy.md)** — `type = "convoy"` formulas
  dispatch into convoy beads. A convoy formula's legs are
  the convoy's tracked issues.
- **[directive](directive.md)** — directives override
  role behavior at prime time; formulas are workflow
  definitions, not role overrides. A formula's steps can
  run under a role whose behavior has been adjusted by a
  directive.

## How it's realized in the code

- **`internal/formula` package** — parsing, validation,
  composition, embedded distribution, overlays, template
  variable checks. See
  [`packages/formula.md`](../packages/formula.md).
- **`internal/cmd/formula.go`** — the `gt formula` CLI:
  `list`, `show`, `run`, `create`. The `list`/`show`
  subcommands are passthroughs to `bd formula`; `run` is
  the heavyweight gastown-specific path that pours into a
  molecule.
- **Embedded formulas** — `internal/formula/formulas/*.formula.toml`
  ship inside the binary, installed to `.beads/formulas/`
  by `gt install`.
- **bd formula** — the beads SDK owns the underlying
  formula storage and query primitives that `gt formula
  list/show` wrap.

## Related wiki pages

- [`internal/formula` package](../packages/formula.md)
- [`gt formula`](../commands/formula.md)
- [molecule concept](molecule.md)
- [wisp concept](../concepts/wisp.md)
- [convoy concept](convoy.md)
- [rig concept](rig.md)
- [directive concept](directive.md)
- [`gt molecule` / `gt mol`](../commands/molecule.md)
- [`internal/convoy`](../packages/convoy.md) — convoy
  formulas dispatch here.
- [`internal/rig`](../packages/rig.md) — `AddRig` seeds
  patrol molecules from formulas.

## Notes / open questions

- **"Pour" is a named verb.** A formula that is poured
  becomes a molecule with per-step wisps. Inline formulas
  execute "at the root level" without sub-wisps. The
  distinction matters for checkpoint recovery: a poured
  molecule can resume mid-flight; an inline one cannot.
- **Convoy formula ≠ convoy bead.** The word "convoy" is
  overloaded. `type = "convoy"` in a formula describes the
  parallel-legs-with-synthesis shape. A convoy bead is a
  work-dispatch bead that tracks a set of issues. They
  meet at `gt formula run` time, where a convoy formula
  can create a convoy bead. See
  [convoy concept](convoy.md).
- **Composition is append-only.** `extends` merges parent
  steps before child steps; there's no "insert between X
  and Y" mechanism. `compose.expand` is the richest shape
  available for reshaping an existing formula. For more
  nuanced surgery, overlays (`ModeReplace`, `ModeAppend`,
  `ModeSkip`) are the escape hatch — at the cost of
  per-rig/per-town operator state instead of versionable
  formula code.
- **Embedded formulas are version-tracked but not
  signed.** `.installed.json` stores the sha256 at install
  time, so user modifications are detected and preserved.
  There's no signature verification on the embedded
  formulas themselves.
- **Agent precedence has four levels.** Per-leg > CLI
  flag > formula-level > rig/town default
  (`formula.go:123-127`). This is a narrow override chain
  compared to the rig config's four-layer chain but covers
  the same "local beats global" intent.
