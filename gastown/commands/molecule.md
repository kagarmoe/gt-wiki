---
title: gt molecule
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/molecule.go
  - /home/kimberly/repos/gastown/internal/cmd/molecule_await_event.go
  - /home/kimberly/repos/gastown/internal/cmd/molecule_await_signal.go
  - /home/kimberly/repos/gastown/internal/cmd/molecule_emit_event.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, molecule, polecat-safe, hook, dag, agent-workflow]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift, wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
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

Sixteen total subcommands, registered across `molecule.go` and
three sibling files via per-file `init()` blocks. (Phase 3 Batch 1c
wiki-stale correction: the Phase 2 page body listed only 11 and
noted `step done` as the `step` group's "only registered child";
re-read at gastown HEAD `9f962c4a` surfaced three more sibling
files that register additional `mol step` subcommands plus an
`await-signal` shortcut directly on `mol`.)

Registered directly on `moleculeCmd` in `molecule.go:258-267`:

| subcommand | source line | one-liner |
|---|---|---|
| `status [target]` | `:131-151` | What's slung on an agent's hook (target defaults to current agent). |
| `current [identity]` | `:153-176` | What the agent should be working on (breadcrumb lookup). |
| `progress <root-issue-id>` | `:47-63` | Show step completion for an instantiated molecule; Witness-facing. |
| `dag` | (sibling file `molecule_dag.go`) | Render the molecule step DAG. |
| `attach [pinned-bead-id] <molecule-id>` | `:65-81` | Attach molecule to a pinned/handoff bead (auto-detects pinned-bead from cwd). |
| `detach <pinned-bead-id>` | `:83-94` | Clear `attached_molecule`/`attached_at` fields. |
| `attachment <pinned-bead-id>` | `:96-105` | Show which molecule is attached. |
| `attach-from-mail <mail-id>` | `:107-129` | Read `attached_molecule:` field from a mail body, attach, mark read. |
| `burn [target]` | `:179-193` | Discard current molecule without a digest (default for wisps). |
| `squash [target]` | `:195-210` | Compress into a permanent digest. Used by patrol cycles. |

Plus a sibling-file-registered shortcut directly on `moleculeCmd`
(added via `molecule_await_signal.go`):

| subcommand | source | one-liner |
|---|---|---|
| `await-signal` (shortcut) | `molecule_await_signal.go:81-132` | Direct shortcut equivalent to `gt mol step await-signal`; registered on `moleculeCmd` via `moleculeAwaitSignalShortcutCmd` at `molecule_await_signal.go:132`. |

Registered on the `step` group (`molecule.go:212-228`, added to
parent at `molecule.go:255`). The parent `molecule.go` registers
only `moleculeStepDoneCmd` (`:254`); three sibling files each
register one additional step subcommand in their own `init()`:

| subcommand | source | one-liner |
|---|---|---|
| `step done` | `molecule_step.go:19-52` (added at `molecule.go:254`) | Complete current step and auto-continue to next. |
| `step await-event` | `molecule_await_event.go:34-115` (added at `:115`) | Wait for file-based events in `~/gt/events/<channel>/`, with optional backoff and cleanup. Dedicated per-channel event directory path (distinct from the activity-feed path used by `await-signal`). |
| `step await-signal` | `molecule_await_signal.go:31-113` (added at `:113`) | Wait for a generic beads activity-feed signal with optional backoff and agent-bead tracking. |
| `step emit-event` | `molecule_emit_event.go:18-64` (added at `:64`) | Emit a file-based event to `~/gt/events/<channel>/` for a consumer running `mol step await-event`. |

All four sibling-file subcommands and the `await-signal` shortcut
were present at gastown `v1.0.0` — this is not post-release churn.

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

## Docs claim

### Source

- `/home/kimberly/repos/gastown/internal/cmd/molecule.go:22-43` —
  `moleculeCmd.Long` categorised subcommand listing.

### Verbatim

> `Agent-specific molecule workflow operations.`
>
> `These commands operate on YOUR hook and YOUR attached molecules.`
> `Use 'gt hook' to see what's on your hook (alias for 'gt mol status').`
>
> `VIEWING YOUR WORK:`
> `  gt hook              Show what's on your hook`
> `  gt mol current       Show what you should be working on`
> `  gt mol progress      Show execution progress`
>
> `WORKING ON STEPS:`
> `  gt mol step done     Complete current step (auto-continues)`
>
> `LIFECYCLE:`
> `  gt mol attach        Attach molecule to your hook`
> `  gt mol detach        Detach molecule from your hook`
> `  gt mol burn          Discard attached molecule (no record)`
> `  gt mol squash        Compress to digest (permanent record)`
>
> `TO DISPATCH WORK (with molecules):`
> `  gt sling mol-xxx target   # Pour formula + sling to agent`
> `  gt formulas               # List available formulas`

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Parent `Long` hand-maintained subcommand categories omit four step-group members and the `await-signal` shortcut

- **Claim source:** `moleculeCmd.Long` at
  `/home/kimberly/repos/gastown/internal/cmd/molecule.go:22-43`.
- **Docs claim:** the `Long` text enumerates five user-facing
  subcommands by category — `current`, `progress`, `step done`,
  `attach`, `detach`, `burn`, `squash` (plus references to
  out-of-group `gt hook`, `gt sling`, `gt formulas`). Under
  "WORKING ON STEPS" the only listed step subcommand is `step done`.
- **Code does:** the parent command has 16 subcommands at HEAD
  `9f962c4a` — the 10 registered directly in `molecule.go` (plus
  `step` group), plus a sibling-file shortcut
  `moleculeAwaitSignalShortcutCmd` at
  `/home/kimberly/repos/gastown/internal/cmd/molecule_await_signal.go:132`,
  plus three additional `step` subcommands (`step await-event`,
  `step await-signal`, `step emit-event`) registered by their own
  sibling files
  (`/home/kimberly/repos/gastown/internal/cmd/molecule_await_event.go:115`,
  `/home/kimberly/repos/gastown/internal/cmd/molecule_await_signal.go:113`,
  `/home/kimberly/repos/gastown/internal/cmd/molecule_emit_event.go:64`).
  None of these four are mentioned anywhere in the parent `Long`
  text. The sibling-file subcommands have full `Long` help of their
  own and are reachable via `gt mol step <name> --help`, but a user
  scanning `gt mol --help` (which prints `moleculeCmd.Long`) will
  not know they exist. `status`, `attachment`, `attach-from-mail`,
  and `dag` are also missing from the Long text, but are at least
  listed as cobra auto-`Available Commands:` output — the four
  step-group additions are doubly hidden because `gt mol --help`'s
  categorised list plus cobra's auto-list show only the parent
  subcommands, never stepping into `step`'s own children.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — edit `moleculeCmd.Long` at
  `molecule.go:22-43` to either name the missing subcommands in the
  existing categories (add a "SYNCHRONIZATION:" or similar heading
  for the `await-*`/`emit-*` family) or stop hand-maintaining the
  list and rely on cobra's auto-generated `Available Commands:`
  block. The `step` subcommand's own `Long` (at `molecule.go:214-224`
  in the `moleculeStepCmd` declaration) also needs a similar
  subcommand enumeration audit, though that's a follow-up, not this
  finding.
- **Release position:** `in-release` — `moleculeCmd.Long` is
  byte-identical at `v1.0.0:internal/cmd/molecule.go:22-43`, and
  `molecule_await_event.go`, `molecule_await_signal.go`, and
  `molecule_emit_event.go` all exist at v1.0.0 with their
  `AddCommand` registrations intact (verified via
  `git -C /home/kimberly/repos/gastown show v1.0.0:internal/cmd/<file>`).

### Phase 2 wiki body under-counted the subcommand set (wiki-stale)

- **Category:** `wiki-stale`
- **Severity:** `wrong`
- **Phase 2 root cause:** `phase-2-incomplete` (heuristic
  determination per Sweep 1 convention — the Phase 2 page was
  written by reading `molecule.go` in isolation and the "`step`
  subcommand is itself a group … with `moleculeStepDoneCmd` as its
  only registered child" claim contradicts sibling files
  `molecule_await_event.go`, `molecule_await_signal.go`, and
  `molecule_emit_event.go` that already existed at `v1.0.0`. Both
  Phase 2's HEAD and the current HEAD disagree with the wiki body
  the same way, so this is "wrong at Phase 2 time," not churn-induced
  drift.)
- **Phase 2 claim (removed):** the Phase 2 page body stated
  "Eleven subcommands, registered at `molecule.go:253-267`" and
  "The `step` subcommand is itself a group (`molecule.go:212-228`)
  with `moleculeStepDoneCmd` as its only registered child
  (`:254-255`)."
- **Current reality (2026-04-15, gastown HEAD `9f962c4a`):** 16
  user-facing subcommands total — 10 on `moleculeCmd` directly (same
  as Phase 2's count minus the parent `step` group), plus an
  `await-signal` shortcut registered on `moleculeCmd` by a sibling,
  plus the `step` group which has 4 children (`step done` plus
  `step await-event`, `step await-signal`, `step emit-event`).
  Sibling files confirmed via `ls internal/cmd/molecule_*.go` and
  `grep -n "AddCommand" internal/cmd/molecule_*.go`. Every sibling
  file present at `v1.0.0` with its AddCommand registrations
  unchanged.
- **Fix tier:** `wiki` — already fixed inline in the
  `### Subcommands` section of `## What it actually does` above (the
  table now enumerates all 16 subcommands across the four files and
  explicitly notes the sibling-file registrations).

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
