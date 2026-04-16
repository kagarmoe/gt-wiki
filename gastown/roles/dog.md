---
title: dog (role)
type: role
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/internal/dog/
  - /home/kimberly/repos/gastown/internal/cmd/dog.go
tags: [role, agent, dog, kennel, cross-rig, infrastructure, deacon-helper]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# dog (role)

A dog is a reusable, cross-rig infrastructure worker — the
Deacon's helper pack for cleanup, migration, and town-level work
that has to touch multiple rigs at once. Dogs live in a kennel at
`<townRoot>/deacon/dogs/` and each dog has a working tree into
every configured rig in the town. Where a polecat builds a single
feature in a single rig and is then nuked, a dog cleans up
multiple rigs and returns to idle, ready for the next assignment.

**Also known as:** Deacon's dogs, the kennel, the pack.

## Purpose

Polecats are good at "build this one feature in this one rig."
But many tasks don't fit that shape:

- "Sync this config across every rig in the town."
- "Run a migration script that touches three rigs."
- "Garbage-collect orphan branches from every rig."
- "Feed this stranded convoy that contains beads from two rigs."
- "Apply a town-wide refactoring."

These are infrastructure tasks. They need cross-rig context, they
need the same worker to operate on multiple working trees, and
they need to NOT be torn down after each task because creating a
worker with worktrees into every rig is expensive.

The dog role exists so that:

- Cross-rig work has a dedicated worker shape.
- The Deacon has a helper pack it can dispatch from.
- Workers with expensive setup (multi-rig worktree creation) can
  be reused.
- Plugin execution has a target that knows how to discover work
  via mail.

The framing in the [`gt dog`](../commands/dog.md) Long help is
canonical: **"Polecats (cats) build features. One rig. Ephemeral
sessions. Dogs clean up messes. Cross-rig. Reusable."**

Without the dog role there would be no agent shape that can
operate on multiple rigs from a single workspace, and the Deacon
would have to either reach into rigs directly (breaking
encapsulation) or sling polecats into many rigs separately.

## Behavior / responsibilities

- Live in the kennel at `<townRoot>/deacon/dogs/<name>/` with a
  worktree per rig at `<dog>/<rigname>/`.
- Wait for work assignment via mail; do nothing until told.
- Execute assigned work (formula, molecule, plugin, or bead) and
  call [`gt dog done`](../commands/dog.md) on completion to
  return to idle.
- Receive plugin dispatches from the Deacon and read full
  instructions from mail (NOT from local plugin files).
- Be reusable: complete work, return to idle, accept the next
  assignment without being torn down.
- Be reset (NOT deleted) by the Deacon's lifecycle commands.
  Identity persists across remove/recreate cycles via the dog
  agent bead.

A dog does not pick its own work. It does not search for things
to do. The startup prompt is explicit: "If your hook is empty and
you have no mail, WAIT — the dispatcher is still setting up your
assignment. Do NOT search for work, scan directories, or take
autonomous action."
(`internal/dog/session_manager.go:113`).

## Lifecycle

A dog has a kennel lifecycle (the directory + per-rig worktrees)
and a session lifecycle (the tmux session + agent process).
They are decoupled: a dog can exist in the kennel without a
session, and a session can be created and destroyed many times
over a single dog's life.

**Kennel lifecycle:**

- **Add.** [`gt dog add <name>`](../commands/dog.md) creates the
  kennel directory, fans out a worktree creation into every
  configured rig (one branch per dog/rig:
  `dog/<name>-<rig>-<millis>`), creates the agent bead, and
  leaves the dog in `idle` state.
- **Remove.** [`gt dog remove <name>`](../commands/dog.md) tears
  down all worktrees, removes the kennel directory, and
  **resets** the agent bead (preserves persistent identity for
  re-add).
- **Refresh.** [`gt dog refresh`](../commands/dog.md) (via the
  package's `Refresh` method) — recreates all worktrees with
  fresh branches. Refuses if the dog is currently working.

**Session lifecycle:**

- **Call.** [`gt dog call`](../commands/dog.md) wakes an idle dog
  by creating its tmux session.
- **Dispatch.** [`gt dog dispatch --plugin <name>`](../commands/dog.md)
  finds the plugin, marks an idle dog as working, sends mail
  with full plugin instructions, and returns immediately
  (non-blocking). The dog discovers the work from its mail and
  executes.
- **Done.** [`gt dog done`](../commands/dog.md) clears the dog's
  work assignment, sets state back to idle, and terminates the
  session.
- **Clear.** [`gt dog clear <name>`](../commands/dog.md) is a
  manual override the Deacon uses during patrol to clear a dog
  that timed out or crashed.

The dog tmux session is named `hq-dog-<name>` — deliberately not
`hq-deacon-...` to avoid tmux prefix-match collisions with the
actual `hq-deacon` session.

## Interactions with other roles

- **Deacon** — the [`deacon` role](deacon.md) owns the kennel
  and dispatches work to dogs. The Deacon's `feed-stranded`
  patrol primitive dispatches dogs to feed stranded convoys via
  `gt sling deacon/dogs <mol-convoy-feed>`. The Deacon also
  decides when hung dogs need clearing (per the ZFC principle —
  Go detects hung, the agent decides).
- **Polecats** — the structural counterpart. The
  [`polecat` role](polecat.md) builds features in one rig
  ephemerally; the dog cleans up across rigs reusably.
- **Mayor** — independent. The [`mayor` role](mayor.md) does
  not directly dispatch dogs (that's the Deacon's job).
- **Witness / Refinery** — independent. Dogs are not part of the
  per-rig polecat lifecycle.
- **Crew** — independent.
- **Boot** — `<kennel>/boot/` is "a special dog" by directory
  layout but is NOT a dog as far as this package is concerned.
  Boot uses `.boot-status.json` instead of `.dog.json`, has no
  `AgentDog` registration, and `dog.Manager.Get("boot")` returns
  `ErrDogNotFound`.

## Runtime substrate

A dog is physically:

- A directory `<townRoot>/deacon/dogs/<name>/`.
- One subdirectory per rig at
  `<townRoot>/deacon/dogs/<name>/<rigname>/`, each of which is a
  git worktree of that rig's bare repo (`.repo.git`) or
  `mayor/rig`.
- A `.dog.json` state file with `name`, `state` (idle/working),
  `last_active`, `work`, `work_started_at`, `worktrees` map,
  and timestamps.
- A `.dog.lock` flock file for serialising load-modify-save on
  the state file.
- A persistent dog agent bead in beads.

When a dog has a session, it adds a `hq-dog-<name>` tmux session
running the agent in the dog's directory. The session's identity
env vars come from `config.AgentEnv` and the startup beacon has
sender `"deacon"` and topic `"assigned"`.

Dogs have **no persistent state in beads** beyond the agent bead
itself. All work tracking is via mail and the dog's own state
file. Dogs are invisible to [`gt agents list`](../commands/agents.md)
because there is no `AgentDog` entry in the `AgentType` enum.

The implementation is in [`internal/dog`](../packages/dog.md).

## Related wiki pages

- [internal/dog](../packages/dog.md) — Go package
  implementation.
- [gt dog](../commands/dog.md) — CLI entry point with 9
  subcommands.
- [deacon role](deacon.md) — the dog's manager.
- [polecat role](polecat.md) — the cats vs dogs counterpart.
- [mayor role](mayor.md) — independent town-level agent.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — `RigsConfig`,
  agent env resolution.
- [`deacon` package](../packages/deacon.md) — feed-stranded
  dispatches dogs.
- [gt sling](../commands/sling.md) — primary work dispatch path
  used to feed dogs (`gt sling deacon/dogs <molecule>`).
- [gt boot](../commands/boot.md) — the "special dog" that isn't
  managed by this package despite living in the kennel.
- [gt plugin](../commands/plugin.md) — the plugin authoring
  surface that produces the work dogs run via dispatch.

## Notes / open questions

- Dogs are deliberately invisible to `gt agents list` /
  `gt agents menu`. The only way to enumerate them is
  `gt dog list` or to look in the kennel directly. Operators
  using the agents menu to navigate will never see a running
  dog session there.
- "Hung" is a state the Go health checker can detect but does
  not auto-resolve by default. The Deacon agent decides whether
  to clear a hung dog. This is the same ZFC pattern that
  appears in the deacon package: Go surfaces data, the agent
  decides.
- A dog has worktrees into EVERY rig, not just the rigs it has
  work in. This means adding a new rig to the town does NOT
  retroactively give existing dogs worktrees into it — they
  would need to be refreshed (via the package's `Refresh`
  method) to pick it up.
- The "cats vs dogs" naming is more than a joke: it captures the
  semantic split between feature workers and infrastructure
  workers. Anyone reading the codebase should treat it as a
  load-bearing distinction, not a cute label.
- Boot being "a special dog" but not actually a dog is a small
  trap. Anyone implementing dog-related logic must remember
  that `dog.Manager.List()` will never include boot, and that
  `<kennel>/boot/` exists but is owned by a different
  subsystem.
