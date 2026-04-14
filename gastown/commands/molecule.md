---
title: gt molecule
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/molecule.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, molecule, polecat-safe, hook, dag, agent-workflow]
---

# gt molecule

Agent-side [molecule](../concepts/molecule.md) workflow commands —
attach/detach molecules to an agent's hook, show progress through a
step DAG, and compress completed molecules into digests. A molecule
is a running instance of a static [formula](../concepts/formula.md)
template. The parent `Use` value is actually `mol`, with
`molecule` as an alias (`molecule.go:16-17`).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`molecule.go:18`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`molecule.go:19`)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/molecule.go:15-269`
for the cobra surface. Implementation helpers (`runMoleculeAttach`,
`runMoleculeProgress`, `runMoleculeSquash`, …) live in sibling files
`molecule_*.go` that aren't read here.

### Invocation

```
gt mol <subcommand> [args]
gt molecule <subcommand> [args]   # alias
```

`moleculeCmd.RunE = requireSubcommand` (`molecule.go:21`) — bare
`gt mol` prints help and errors. The Long text at `molecule.go:22-43`
groups subcommands by purpose: viewing, working, lifecycle.

### Subcommands

Eleven subcommands, registered at `molecule.go:253-267`:

| subcommand | source line | one-liner |
|---|---|---|
| `status [target]` | `:131-151` | What's slung on an agent's hook (target defaults to current agent). |
| `current [identity]` | `:153-176` | What the agent should be working on (breadcrumb lookup). |
| `progress <root-issue-id>` | `:47-63` | Show step completion for an instantiated molecule; Witness-facing. |
| `dag` | (sibling file) | Render the molecule step DAG. |
| `attach [pinned-bead-id] <molecule-id>` | `:65-81` | Attach molecule to a pinned/handoff bead (auto-detects pinned-bead from cwd). |
| `detach <pinned-bead-id>` | `:83-94` | Clear `attached_molecule`/`attached_at` fields. |
| `attachment <pinned-bead-id>` | `:96-105` | Show which molecule is attached. |
| `attach-from-mail <mail-id>` | `:107-129` | Read `attached_molecule:` field from a mail body, attach, mark read. |
| `burn [target]` | `:179-193` | Discard current molecule without a digest (default for wisps). |
| `squash [target]` | `:195-210` | Compress into a permanent digest. Used by patrol cycles. |
| `step done` | (sibling file, added via `moleculeStepCmd`) | Complete current step and auto-continue. |

The `step` subcommand is itself a group (`molecule.go:212-228`) with
`moleculeStepDoneCmd` as its only registered child (`:254-255`).

### Flags

- `--json` on `status`, `current`, `progress`, `attachment`, `burn`,
  `squash` (single shared `moleculeJSON` var — setting it on one
  subcommand sets it for all, `molecule.go:233-248`).
- `--jitter <duration>` on `squash` (`:249`): sleep a random duration
  0..value before squashing to reduce concurrent Dolt lock contention.
- `--summary <text>` on `squash` (`:250`): optional summary for the
  digest (e.g. patrol observations).
- `--no-digest` on `squash` (`:251`): skip digest bead creation (for
  patrol molecules that run frequently).

### Relation to other commands

- [hook](hook.md) — `gt hook` is an alias for `gt mol status`
  (per `molecule.go:28`). A polecat's "hook" is literally its pinned
  bead with an attached molecule.
- [sling](sling.md) — dispatches `mol-xxx target` to pour a formula
  and attach the molecule on a target polecat (`molecule.go:41-42`).
- [formula](formula.md) — formulas bake into molecule instances when
  slung.
- [convoy](convoy.md) — molecules are tracked by convoys via the
  `--molecule` flag on `convoy create`.
- [mountain](mountain.md) — mountains are convoys with the `mountain`
  label, separate from molecule step execution but feed it.
- [done](done.md) — polecats call `gt done` after completing the
  current step, which does the `mol step done` work plus merge-queue
  handoff.

## Notes / open questions

- **`Use = "mol"`, alias = `molecule`** — the file is named
  `molecule.go` and the var is `moleculeCmd`, but the canonical CLI
  invocation is `gt mol`. `gt molecule` works via the alias
  (`molecule.go:17`). The help tree under `gt help` will show `mol`.
- **Shared `moleculeJSON` flag variable.** Re-using the same package
  var across six subcommands is unusual cobra style — tests running
  subcommands sequentially could see the flag leak across calls.
- **"Agent-specific operations only"** (`molecule.go:257`): this
  hints at a sibling `bd mol …` or other invocation surface for
  molecule-author operations (creating molecule definitions, editing
  step DAGs). Not implemented in this file — see `bd mol wisp list`
  used by [ready](ready.md)'s wisp filter.
- **Wisps** (`molecule.go:191`): wisps are ephemeral molecules where
  `burn` is the default completion. The wisp/digest distinction
  matters for audit trail shape.
