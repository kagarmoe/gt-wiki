---
title: gt dog
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/dog.go
tags: [command, agents, dog, kennel, infrastructure, cross-rig, deacon, dispatch]
---

# gt dog

Manage [dogs](../roles/dog.md) — reusable, cross-rig infrastructure
workers that live in the kennel at `~/gt/deacon/dogs/` and are
dispatched by the [Deacon](../roles/deacon.md). This page documents
the `gt dog` CLI surface; see the [Dog role page](../roles/dog.md)
for persona details.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`dog.go:48`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no
**Alias:** `gt dogs`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/dog.go`
(1225 lines). Registration: `dog.go:256-299` attaches nine
subcommands to `dogCmd` and registers the parent on `rootCmd`.

The parent `dogCmd` (`dog.go:45-65`) has no `RunE`, so `gt dog`
with no arguments prints help.

### Invocation

```
gt dog add <name>
gt dog remove <name>... | --all [--force]
gt dog list [--json]
gt dog call [name] | --all
gt dog done [name]
gt dog clear <name> [--force]
gt dog status [name] [--json]
gt dog dispatch --plugin <name> [--rig <r>] [--dog <name>] [--create] [--dry-run] [--json]
gt dog health-check [name] [--json] [--auto-clear] [--max-inactivity <d>]
```

### Cats vs Dogs (from the parent Long help, `dog.go:50-64`)

The in-file framing is explicit:

> Polecats (cats) build features. One rig. Ephemeral sessions (one
> task, then nuked).
> Dogs clean up messes. Cross-rig. Reusable (multiple tasks,
> eventually recycled).

Dogs are managed by the Deacon for town-level work:
infrastructure tasks (rebuilding, syncing, migrations), cleanup
operations (orphan branches, stale files), and cross-rig work.
Each dog has worktrees into every configured rig, enabling
cross-project operations. Dogs return to idle state after
completing work (unlike cats).

The kennel is `~/gt/deacon/dogs/`. The Deacon dispatches work to
dogs.

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `add <name>` | `dogAddCmd` | `dog.go:67-80` | `runDogAdd` (`dog.go:317-358`) |
| `remove` (alias `rm`?) | `dogRemoveCmd` | `dog.go:82-105` | `runDogRemove` (`dog.go:360-439`) |
| `list` (alias `ls`) | `dogListCmd` | `dog.go:107-120` | `runDogList` (`dog.go:441-…`) |
| `call [name]` | `dogCallCmd` | `dog.go:122-139` | `runDogCall` |
| `done [name]` | `dogDoneCmd` | `dog.go:141-158` | `runDogDone` |
| `clear <name>` | `dogClearCmd` | `dog.go:160-176` | `runDogClear` |
| `status [name]` | `dogStatusCmd` | `dog.go:178-199` | `runDogStatus` |
| `dispatch --plugin <n>` | `dogDispatchCmd` | `dog.go:201-226` | `runDogDispatch` |
| `health-check [name]` | `dogHealthCheckCmd` | `dog.go:228-254` | `runDogHealthCheck` |

`dog list` aliases to `dog ls` (`dog.go:109`). `remove` has no
declared alias.

### Per-subcommand behavior

**`dog add <name>`** — `runDogAdd` (`dog.go:317-358`). Validates
that the name contains no `/`, `\`, `.`, or spaces
(`dog.go:321-323`). Loads a `dog.Manager` via `getDogManager`
(`dog.go:302-315`) which reads `<townRoot>/mayor/rigs.json` to
know which rigs to create worktrees for. Calls `mgr.Add(name)` to
create the dog, prints the kennel path and all per-rig worktree
paths, then creates an **agent bead** for the dog via
`beads.New(townRoot).CreateDogAgentBead(name, "deacon/dogs/<name>")`.
Bead creation failure is non-fatal — the dog exists with a
warning.

**`dog remove`** — `runDogRemove` (`dog.go:360-439`). Either takes
one or more names or `--all` (validated in `dogRemoveCmd.Args` at
`dog.go:95-103`). Refuses to remove a dog in
`dog.StateWorking` unless `--force` is passed. For each removed
dog, calls `mgr.Remove(name)` (tears down worktrees and
directory), then calls `b.ResetDogAgentBead(name)` to reset the
agent bead (preserves persistent identity — the bead is reset
rather than deleted). Failures are accumulated and reported at
the end; returns a non-zero error when any removal fails.

**`dog list`** — `runDogList`. Prints each dog's name, state
(`idle`/`working`), current work, and last-active timestamp.
With `--json`, emits the list as JSON.

**`dog call [name]`** — wakes an idle dog. With `--all`, wakes
all idle dogs. With no arguments, wakes one idle dog if
available. Updates last-active and can trigger session creation
for the dog's worktrees (per `dog.go:131-132`).

**`dog done [name]`** — marks a dog as done and returns it to
idle. Without a name, auto-detects the current dog from the
working directory (must be run from within a dog's worktree).

**`dog clear <name>`** — resets a stuck dog to idle. By default
refuses if the tmux session still exists; `--force` overrides.
The long help notes: "The Deacon uses this during patrol to
clear dogs that have timed out" (`dog.go:166-167`).

**`dog status [name]`** — detailed status for one dog (state,
current work, worktree paths per rig, last-active), or a pack
summary (totals, idle/working counts, pack health) without a
name. `--json` for machine-readable output.

**`dog dispatch --plugin <name>`** — dispatches plugin execution.
Per `dog.go:205-216`:

> 1. Finds the plugin definition (plugin.md)
> 2. Assigns work to an idle dog (marks as working)
> 3. Sends mail with plugin instructions to the dog
> 4. Returns immediately (non-blocking)
> The dog discovers the work via its mail inbox and executes the
> plugin instructions. On completion, the dog sends DOG_DONE mail
> to deacon/.

`--plugin` is required (`dog.go:280` — `MarkFlagRequired`).
`--rig` limits plugin search to a specific rig; `--dog` targets a
specific dog; `--create` creates a new dog if none are idle;
`--dry-run` previews without executing; `--json` for output.

**`dog health-check [name]`** — per `dog.go:233-244`:

> Detects:
>   - Zombies: state=working but tmux session or agent process is dead
>   - Hung: agent alive but no tmux activity for too long
>   - Orphans: dog idle but tmux session still exists
>
> With --auto-clear, zombies are automatically returned to idle
> state. Hung dogs are reported only (Deacon decides per ZFC
> principle).
>
> Exit codes:
>   0 = all healthy
>   1 = error
>   2 = needs attention

The `--max-inactivity` flag controls the hung threshold (default
`10m`). Auto-clear only affects zombies, not hung dogs — hung
determination is deferred to the Deacon agent.

### Dog states

From the references in this file (`dog.StateWorking`,
`dog.StateIdle` — imports `internal/dog` at `dog.go:15`), a dog
exists in one of two states: `idle` or `working`. State
transitions are driven by `call` (→ wake), `dispatch` (→
working), `done` / `clear` (→ idle), and `health-check
--auto-clear` (zombies → idle).

### Flags

Parent-level flags: none. Each subcommand has its own flags:

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--json` | bool | `false` | `list`, `status`, `dispatch`, `health-check` | `dog.go:258,271,278,283` |
| `-f, --force` | bool | `false` | `remove`, `clear` | `dog.go:261,268` |
| `--all` | bool | `false` | `remove`, `call` | `dog.go:262,265` |
| `--plugin <n>` | string | `""` (required) | `dispatch` | `dog.go:274,280` |
| `--rig <r>` | string | `""` | `dispatch` | `dog.go:275` |
| `--dog <n>` | string | `""` | `dispatch` | `dog.go:276` |
| `--create` | bool | `false` | `dispatch` | `dog.go:277` |
| `-n, --dry-run` | bool | `false` | `dispatch` | `dog.go:279` |
| `--auto-clear` | bool | `false` | `health-check` | `dog.go:284` |
| `--max-inactivity <d>` | duration | `10m` | `health-check` | `dog.go:285` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [deacon.md](deacon.md) — the Deacon is the primary caller of
  `gt dog dispatch`, `gt dog clear`, and `gt dog health-check`.
  The Deacon's `feed-stranded` subcommand dispatches dogs via
  `gt sling`. See also `deacon.go:279-305`.
- [plugin.md](plugin.md) — `gt dog dispatch --plugin <name>`
  looks up plugin definitions; plugin authoring and listing live
  under `gt plugin`.
- [sling.md](sling.md) — an alternate dispatch path used by the
  Deacon's `feed-stranded`; slings dispatch work to polecats and
  dogs via bead IDs rather than plugins.
- [agents.md](agents.md) — dogs do *not* appear in `gt agents
  list` (no `AgentDog` type in the `AgentType` enum at
  `agents.go:24-33`). Listing dogs is only possible via `gt dog
  list`.
- [boot.md](boot.md) — Boot is described in its Long help as "a
  special dog", lives under `deacon/dogs/boot/`, but is *not*
  registered in the dog kennel manager.
- [convoy.md](convoy.md) — dogs are a dispatch target for
  stranded convoys via `gt deacon feed-stranded`.
- [formula.md](formula.md), [molecule.md](molecule.md),
  [mountain.md](mountain.md) — adjacent work-recipe primitives.
  Dogs execute plugins; polecats execute formulas.
- [cat.md](cat.md) — utility helper (unrelated to cats/polecats
  despite the name).

## Notes / open questions

- **Dogs are invisible to `gt agents`.** Because `AgentType` in
  [agents.md](agents.md) has no `AgentDog` entry, dogs are a
  separate world from the mayor/deacon/witness/refinery/crew/
  polecat taxonomy. Anyone using `gt agents menu` to navigate
  will never see a running dog session there — they have to use
  `gt dog list` or attach to the dog's tmux session directly.
- **Agent bead lifecycle** — `add` creates an agent bead,
  `remove` *resets* it rather than deleting, preserving
  persistent identity across add/remove cycles.
  `CreateDogAgentBead` and `ResetDogAgentBead` are both in the
  [`internal/beads`](../packages/beads.md) package but only called from this file.
- **`dispatch` requires `--plugin`, so there is no "dispatch by
  bead" mode.** Bead-driven dispatch goes through `gt sling`
  instead. This is a real split: `gt dog dispatch --plugin` is
  for operator/Deacon-driven plugin runs, and `gt sling <bead>
  <rig>` is for bead-driven assignment.
- **`dispatch --create` is an escape hatch.** If no dog is idle
  and `--create` is set, a new dog is created and immediately
  dispatched. There is no flag-level cap on how many dogs may be
  created this way.
- **Health-check is split**: Go code decides zombies (dead
  session) but delegates the hung decision to the Deacon "per
  ZFC principle" (`dog.go:236-240`). The `--max-inactivity`
  threshold is the knob the Go code uses to *detect* hung, but
  clearing is manual.
- **No `dog attach`.** Unlike [mayor.md](mayor.md) and
  [deacon.md](deacon.md), there is no `gt dog attach`
  subcommand. Operators attach to a dog's tmux session with
  plain `tmux attach` / the [agents.md](agents.md) menu — except
  the menu won't list dogs, as noted above.
- **Role/concept page pending.** The Dog persona (watchdog's
  reusable worker, cross-rig cleanup, kennel) is pending
  Batch 6 — see `gastown/roles/dog.md` (not yet created).
