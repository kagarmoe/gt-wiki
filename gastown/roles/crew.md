---
title: crew (role)
type: role
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/crew/
  - /home/kimberly/repos/gastown/internal/cmd/crew.go
tags: [role, agent, crew, persistent, user-managed, full-clone, rig-level]
---

# crew (role)

A crew worker is a persistent, user-managed agent workspace inside
a single rig — a long-lived development environment that a human
or a long-running agent session can return to over many days,
across many tasks, with its own full clone of the project repo.
Where a polecat is an ephemeral worker the witness destroys after
one task, a crew member is a desk in a building: it has a name, it
keeps its tools, and the only thing that removes it is the user
explicitly removing it.

**Also known as:** crew worker, crew member.

## Purpose

Polecats and crew solve different problems. Polecats are good at
"I have a single bead, run an agent against it, then tear
everything down." But many real workflows don't fit that shape:

- Long-running development sessions where the agent accumulates
  context across days and tasks.
- Human developers who want a persistent agent companion in a
  workspace they own.
- Manual experiments and exploration where the workspace state
  matters more than the bead.
- Workflows where the same project clone is touched repeatedly
  and a fresh worktree per task would lose useful state.

Crew workspaces exist so that Gas Town can host these long-lived,
user-managed sessions without forcing them through the
witness/refinery cleanup machinery designed for ephemeral work.
The defining property is that the crew's workspace **stays until
the user removes it**.

Without the crew role there would be no way to run an agent
against a project for an extended period without subjecting it to
polecat-style auto-nuke.

## Behavior / responsibilities

- Hold a full git clone of a single rig's project repository at
  `<rig>/crew/<name>/`.
- Provide a stable identity (name, mailbox, beads agent) that
  survives session restarts, agent crashes, and machine reboots.
- Run an interactive agent session in a tmux window the user can
  attach to and detach from at will.
- Receive and process mail/nudges directed at the crew member's
  address.
- Pull from origin via [`gt crew pristine`](../commands/crew.md)
  to stay in sync with the rig's default branch.
- Persist work indefinitely until the user explicitly removes the
  workspace.

A crew worker does NOT participate in the
witness → done → nuke loop. It does not auto-clean. It does not
get garbage-collected. The Witness leaves it alone.

## Lifecycle

A crew worker has two lifecycles in series: the **workspace**
lifecycle (the directory and clone) and the **session** lifecycle
(the tmux session and agent process). They are decoupled.

**Workspace lifecycle:**

- **Add.** [`gt crew add <name>`](../commands/crew.md) clones the
  rig's repo into `<rig>/crew/<name>/`, optionally creates a
  `crew/<name>` feature branch, copies overlay files, runs setup
  hooks, sets up shared beads, installs runtime settings.
- **Pristine.** [`gt crew pristine`](../commands/crew.md) brings
  the workspace up to date with origin via `git pull --rebase`.
- **Rename.** [`gt crew rename <old> <new>`](../commands/crew.md)
  renames the workspace directory, preserving state.
- **Refresh.** [`gt crew refresh <name>`](../commands/crew.md)
  sends the crew member a handoff mail in their own inbox, then
  restarts the session so the new session picks up the handoff.
- **Remove.** [`gt crew remove <name>`](../commands/crew.md)
  deletes the workspace. With `--purge`, also deletes the agent
  bead, unassigns any beads currently assigned to the crew
  member, and clears the inbox.

**Session lifecycle:**

- **Start.** [`gt crew start`](../commands/crew.md) creates the
  tmux session, runs the agent with the crew workspace as cwd,
  and starts a nudge poller for queued mail delivery. Optionally
  resumes a previous Claude session via `--resume[=<id>]`.
- **At.** [`gt crew at <name>`](../commands/crew.md) attaches to
  the session, creating it if necessary. Auto-discovers the rig
  from the current directory if called bare.
- **Stop.** [`gt crew stop`](../commands/crew.md) kills the tmux
  session and all descendant processes; the workspace remains.
- **Restart.** Stop + start.

The workspace persists across all session start/stop/restart
cycles. The agent's persistent identity (agent bead, mailbox,
shared beads) persists across workspace renames.

## Interactions with other roles

- **Polecats** — the [`polecat` role](polecat.md) is the
  structural counterpart: same rig scope, same general shape, but
  ephemeral instead of persistent and witness-managed instead of
  user-managed. The
  [`gt crew`](../commands/crew.md) Long help is explicit:
  "Polecats: Ephemeral sessions. Witness-managed. Auto-nuked
  after work. / Crew: Persistent. User-managed. Stays until you
  remove it."
- **Witness** — does NOT manage crew. Crew is invisible to the
  Witness's nuke loop.
- **Mayor** — the [`mayor` role](mayor.md) is the natural
  escalation target for cross-rig crew coordination. Crew member
  CLAUDE.md / SessionStart hook wires crew into the same mail /
  nudge addressing the Mayor uses.
- **Deacon** — the [`deacon` role](deacon.md) covers crew in its
  stale-hook scan (the `assigneeToWorktreePath` helper in
  `internal/deacon/stale_hooks.go` knows about both polecat and
  crew worktree layouts). So if a crew member abandons a hooked
  bead and dies, the Deacon will eventually unhook it.
- **Refinery** — independent. Crew members merge their own work
  via the same merge-queue surface anyone else uses; the
  Refinery does not own crew branches.
- **Dogs** — independent.
- **The user (Overseer)** — the principal. Crew exists for the
  user's benefit, not the agents'.

## Runtime substrate

A crew worker is physically a directory at `<rig>/crew/<name>/`
containing a full git clone of the rig's project repository. Not
a worktree — a full clone with its own `.git/`. Inside it lives a
mail directory, a state file (`state.json`), runtime settings,
overlay files copied from the rig, and any setup-hook outputs.

The crew's tmux session is named via
`session.CrewSessionName(rigPrefix, name)` and runs in the crew
clone as cwd. The session's identity environment variables are
seeded via tmux `-e` flags before the shell starts (NOT via
send-keys after) to prevent parent env vars from leaking in
(gh#1289). The session has crew-cycle keybindings (`C-b n` /
`C-b p`) bound for crew-to-crew navigation.

Persistent state belonging to a crew member:

- The clone directory itself (workspace, branch state, untracked
  files).
- `<rig>/crew/<name>/state.json` — manager state (name, branch,
  timestamps).
- `<rig>/crew/<name>/mail/` — mail inbox.
- `<rig>/.runtime/locks/crew-<name>.lock` — flock for serialising
  filesystem operations.
- The crew's agent bead in the rig's beads database.

The implementation is in [`internal/crew`](../packages/crew.md).

## Related wiki pages

- [internal/crew](../packages/crew.md) — Go package
  implementation.
- [gt crew](../commands/crew.md) — CLI entry point with 13
  subcommands.
- [polecat role](polecat.md) — the ephemeral counterpart.
- [mayor role](mayor.md) — cross-rig escalation target.
- [deacon role](deacon.md) — town-level watchdog covering crew
  in stale-hook scans.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — agent env, startup
  command builders.
- [`beads` package](../packages/beads.md) — shared beads setup,
  `ProvisionPrimeMDForWorktree`.
- [`nudge` package](../packages/nudge.md) — background poller
  for queued mail delivery.
- [gt worktree](../commands/worktree.md) — uses
  `detectCrewFromCwd` from this package.
- [gt mail](../commands/mail.md), [gt nudge](../commands/nudge.md)
  — delivery primitives that crew members consume.

## Notes / open questions

- "Crew worker" and "crew member" are used interchangeably in the
  docs and the code. The package uses `CrewWorker`; the CLI long
  help calls them "crew workers".
- A crew member is a single agent in a single rig. There is no
  cross-rig crew model — each rig has its own crew namespace. A
  human who works on three rigs has three independent crews.
- The full-clone (vs worktree) choice has cost: each crew member
  is a full project repo. For large repos this matters. But it
  also means crew members can be operated on independently of
  the rig's main repo, including being pushed to forks the rig
  doesn't know about.
- The crew role does not currently support a "headless" mode the
  way the [mayor role](mayor.md) does. A crew session is always
  a tmux session.
