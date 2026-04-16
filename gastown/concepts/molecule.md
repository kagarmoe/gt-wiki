---
title: molecule (concept)
type: concept
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/internal/cmd/molecule.go
  - /home/kimberly/repos/gastown/internal/cmd/formula.go
  - /home/kimberly/repos/gastown/internal/formula/types.go
tags: [concept, molecule, formula, running-instance, workflow, polecat-safe, bead-tracked, step-dag]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# molecule

A **molecule** is a running instance of a [formula](formula.md).
If a formula is the static TOML template that defines what a
workflow does, a molecule is what you get when that template
is executed — a live, bead-tracked entity with an identity,
a current step, a record of which steps have completed, and
(for poured molecules) a tree of step wisps capturing
per-step state. Molecules are Gas Town's unit of "an agent
is doing a multi-step task."

> **A formula is to a molecule what a class is to an
> instance.**
> A formula is immutable data on disk. A molecule is a
> running bead in the beads database that was *poured* from
> that formula and exists for the duration of a task.

Molecules are fundamental: most of the multi-step agent
work in Gas Town is molecule-driven. Patrol loops are
molecules. Convoy-fed tasks are molecules. Polecat sessions
often have a molecule attached to their hook bead.
`gt sling` dispatching a formula onto an agent means
pouring a molecule and attaching it to that agent's hook.

## Why molecules exist

A formula alone is a definition, not a state. Running a
formula needs somewhere to track:

- Which step is currently executing?
- Which steps have completed?
- Which steps are ready to execute (dependencies met)?
- What's the acceptance criteria for the current step?
- If the agent session dies mid-flight, where do we resume?
- Which polecat is working on this step?
- What variables were supplied at run time?
- What is the overall root bead that ties everything
  together?

All of this is molecule state. The molecule is a bead (or a
tree of beads and wisps) in the beads database that answers
these questions. The formula is consulted to know what to
do next; the molecule is consulted to know where we are.

## Anatomy of a molecule

A molecule has several moving parts visible via the `gt mol`
(`gt molecule`) CLI surface, described in
`/home/kimberly/repos/gastown/internal/cmd/molecule.go:15-269`:

- **Root bead** — the anchor bead representing the molecule
  itself. Its ID is the `<root-issue-id>` argument to
  `gt mol progress`. Typically a permanent bead for durable
  molecules; for ephemeral patrol cycles, the whole molecule
  is wisp-shaped.
- **Step DAG** — the structure inherited from the formula's
  `[[steps]]` / `[[legs]]` / `[[template]]` definitions with
  their `needs` / `depends_on` wiring.
- **Step states** — which steps have completed, which are
  in progress, which are ready. For poured molecules
  (`formula.Pour = true`), each step is a sub-wisp that
  tracks its own state.
- **Attached polecat / hook bead** — when a molecule is
  attached to an agent's hook via `gt mol attach`, the
  polecat's pinned bead gets `attached_molecule` and
  `attached_at` fields set (`molecule.go:65-94`).
- **Digest / burn shape** — on completion, a molecule is
  either **squashed** (compressed into a permanent digest
  bead) or **burned** (discarded without a digest). Default
  for wisps is burn; patrol-style molecules often squash.

## Poured vs inline molecules

A formula's `Pour` flag (`types.go:34`) controls whether
its molecule is materialized as sub-wisps or kept inline:

- **Poured molecule** (`pour = true`): each step is a
  separate wisp in the beads database. The molecule can
  checkpoint — if the agent session dies mid-step, the next
  session can look at the wisp tree, see which steps
  completed, and resume from the next ready step. Poured
  molecules are durable across session boundaries.
- **Inline molecule** (`pour = false`, the default): the
  steps execute at the root level without creating
  sub-wisps. Simpler shape, less overhead, but no per-step
  checkpoint — if the session dies, the next attempt
  starts over.

Patrol cycles and multi-hour convoy legs typically use
`pour = true`. Lightweight single-use workflows use the
default inline shape.

## The `gt mol` CLI surface

From `/home/kimberly/repos/gastown/internal/cmd/molecule.go`,
eleven subcommands are registered (`molecule.go:253-267`):

| subcommand | line | purpose |
|---|---|---|
| `status [target]` | `:131-151` | What's slung on an agent's hook |
| `current [identity]` | `:153-176` | What the agent should be working on |
| `progress <root>` | `:47-63` | Step completion for a specific molecule |
| `dag` | (sibling file) | Render the step DAG |
| `attach [bead] <mol>` | `:65-81` | Attach a molecule to a pinned bead |
| `detach <bead>` | `:83-94` | Clear the attachment |
| `attachment <bead>` | `:96-105` | Show which molecule is attached |
| `attach-from-mail <mail>` | `:107-129` | Attach from a mail body field |
| `burn [target]` | `:179-193` | Discard without a digest |
| `squash [target]` | `:195-210` | Compress into a permanent digest |
| `step done` | `:212-228` + sibling | Complete current step, auto-continue |

The parent `Use` is actually `"mol"`, with `"molecule"` as
an alias (`molecule.go:16-17`). `gt help` shows `mol` in
the command tree. The command is
**`AnnotationPolecatSafe: "true"`** (`molecule.go:19`) —
polecats are allowed to call it while hooked to a molecule.

## Lifecycle of a molecule

1. **Pour** — `gt formula run <name>` (or `gt sling <bead>
   <rig>` with a formula-based dispatch) creates the
   molecule root bead from the formula. For convoy
   formulas, a convoy bead is created with per-leg beads
   tracked; for workflow formulas, a root molecule bead is
   created with optional per-step wisps.
2. **Attach** — the molecule is attached to an agent's
   pinned bead (hook bead) via `gt mol attach`. The
   pinned bead gets `attached_molecule` and `attached_at`
   fields. The polecat now "has a hook."
3. **Execute** — the agent (polecat) runs the current
   step. The formula's step DAG and the molecule's step
   state determine what "current step" means. The agent
   consults `gt mol current` to know what to do and
   `gt mol progress` to see the overall state.
4. **Step done** — `gt mol step done` marks the current
   step as complete and auto-advances. For poured
   molecules, the sub-wisp is closed; for inline
   molecules, the root state updates.
5. **Complete** — when all steps are done (or when the
   agent calls `gt done` as a polecat shortcut), the
   molecule is ready for its end-state transition.
6. **Squash or burn** — `gt mol squash` compresses the
   completed molecule into a permanent digest bead. Patrol
   molecules use `--jitter` to reduce Dolt contention when
   many are squashing at once and `--no-digest` to skip
   digest creation for high-frequency patrols. `gt mol burn`
   discards without a digest — the default for wisp-shaped
   ephemeral molecules.

## Molecule step wisps

When a molecule is poured (`pour = true`), each step
becomes a sub-wisp in the [wisps table](wisp.md). Molecule
step wisps are one of the canonical wisp categories listed
in the wisp concept page — ephemeral beads with a TTL, a
promotion criterion, and a reaper sweep. If a molecule
step wisp accumulates comments or is referenced by a
permanent bead, it gets promoted and moves to `issues`.
Otherwise it compacts away when the molecule finishes and
its audit trail is no longer interesting.

This is why molecules work with the broader wisp model:
step state is, by design, ephemeral. The permanent record
of what a molecule did is the squashed digest bead, not
the per-step wisps.

## `gt mol current` and the breadcrumb lookup

`gt mol current [identity]` (`molecule.go:153-176`) does a
**breadcrumb lookup**: given an identity (defaulting to the
caller's own), find what the agent *should be working on*
right now. The result is the current step of the currently
attached molecule, formatted as a prompt. Agents use this
to regain context after a session restart or a handoff —
"what was I doing?" answered authoritatively from the
beads database.

Combined with `gt mol status`, the agent has two
complementary views: **status** ("what's on my hook right
now") and **current** ("what should I be doing right now,
given what's on my hook").

## Attachment via mail

`gt mol attach-from-mail <mail-id>` (`molecule.go:107-129`)
is a specialized form where the molecule to attach is
named in a mail body's `attached_molecule:` field. The
caller reads the mail, parses the field, attaches, and
marks the mail as read. This is how Deacon-dispatched
molecules reach a dog or polecat: the mail IS the
attachment instruction.

## Relationships with other concepts

- **[formula](formula.md)** — every molecule has exactly
  one formula. The formula is static; the molecule is the
  dynamic state.
- **[wisp](wisp.md)** — molecule steps (when poured) are
  wisps. Molecule root beads can be permanent or
  wisp-shaped depending on whether the molecule is
  patrol-style or a real work unit.
- **[polecat](../roles/polecat.md)** — polecats host
  molecules via their hook bead. A polecat doing work
  usually has an attached molecule.
- **[convoy](convoy.md)** — convoy-type formulas pour
  molecules that create convoy beads; each leg bead is
  separately tracked.
- **[directive](directive.md)** — directives adjust
  *role behavior*, not molecule structure. A molecule
  runs under a role whose directive has already been
  loaded at prime time.
- **[rig](rig.md)** — a molecule runs in a rig context;
  `gt mol` commands resolve the rig from the caller's
  environment.
- **[identity](identity.md)** — `gt mol current` and
  `gt mol status` default their "target" to the caller's
  detected identity.

## How it's realized in the code

- **`internal/cmd/molecule.go`** — the cobra surface
  (`:15-269`). Sibling files `molecule_*.go` hold the
  actual `runMoleculeAttach` / `runMoleculeProgress` /
  `runMoleculeSquash` implementations.
- **`internal/formula`** — the formula-side definition
  (see [formula package](../packages/formula.md)).
- **`internal/beads`** — the bead SDK primitives for
  reading and writing molecule state
  (`attached_molecule`, `attached_at`, step wisps, digest
  beads).
- **`internal/polecat/session_manager.go`** — `hookIssue`
  records "this polecat owns this issue" when a sling +
  molecule-attach happens.
- **`internal/deacon`** — dispatches dogs with molecule
  attachments via mail (`attached_molecule:` field).
- **`internal/convoy`** — the convoy-launch path creates
  convoy beads that may have molecule formulas driving
  them.
- **`bd mol wisp list`** — the beads CLI command that
  filters to wisp-shaped molecules.

## Related wiki pages

- [formula concept](formula.md) — the static template
  side.
- [wisp concept](wisp.md) — molecule steps are wisps.
- [`gt molecule` / `gt mol`](../commands/molecule.md) —
  CLI reference.
- [`gt formula`](../commands/formula.md) — pour a formula
  into a molecule.
- [`gt sling`](../commands/sling.md) — dispatch with a
  formula creates a molecule on the target.
- [`gt done`](../commands/done.md) — polecat-side
  shortcut for marking the current step done and
  completing.
- [`gt hook`](../commands/hook.md) — alias for
  `gt mol status` (`molecule.go:28`).
- [`internal/polecat`](../packages/polecat.md) — polecat
  hooks and molecule attachments.
- [convoy concept](convoy.md) — convoy formulas produce
  convoy beads with associated molecules.
- [rig concept](rig.md) — molecules run in a rig
  context.
- [directive concept](directive.md) — affects the role a
  molecule runs under, but not the molecule itself.

## Notes / open questions

- **`Use = "mol"`, not `"molecule"`.** The canonical CLI
  invocation is `gt mol`; `gt molecule` works via an
  alias (`molecule.go:17`). The help tree shows `mol`.
  This is uncharacteristic for gastown, which usually
  prefers spelled-out subcommand names.
- **"Molecule" as a bead-type.** Beads have a custom type
  called `molecule` (registered in `constants.BeadsCustomTypes`
  alongside `agent`, `role`, `rig`, `convoy`). Root
  molecule beads use this type; step wisps have their own
  types.
- **Burn vs squash is not the same axis as pour vs
  inline.** Pour/inline is set at formula authoring time
  and controls runtime state shape. Burn/squash is chosen
  at molecule completion time and controls audit-trail
  shape. A poured molecule can be burned; an inline
  molecule can be squashed. They compose.
- **`moleculeJSON` is a shared package var.** The `--json`
  flag is implemented as a single variable reused across
  six subcommands (`molecule.go:233-248`). Cobra flag
  state across subcommands can leak if two subcommands
  run in the same process, which matters for tests.
- **Molecule-author operations are elsewhere.** This
  file has agent-side operations only (attach, status,
  progress, squash). Creating molecule definitions and
  editing step DAGs happens in `bd mol …` — outside the
  `gt` binary entirely.
- **Convoy-fed molecules are attached via dog dispatch.**
  The Deacon's `mol-convoy-feed` molecule is dispatched to
  a dog via mail, and the dog's first act is `gt mol
  attach-from-mail`. The mail pipeline is the attachment
  channel for multi-stage convoy work.
- **`gt mol step done` auto-continues.** After closing
  the current step wisp, it advances to the next ready
  step automatically. This is why polecats don't need to
  poll — finishing a step drops them straight into the
  next one without an explicit "advance" command.
