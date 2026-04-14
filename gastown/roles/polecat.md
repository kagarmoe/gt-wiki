---
title: polecat (role)
type: role
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/polecat/
  - /home/kimberly/repos/gastown/internal/cmd/polecat.go
tags: [role, agent, polecat, ephemeral, witness-managed, persistent-identity, worktree, rig-level]
---

# polecat (role)

A polecat is Gas Town's primary feature-building agent — an LLM
worker with a **persistent identity** (a permanent agent bead, a
CV chain accumulating work history across assignments, a stable
themed name like `furiosa` or `obsidian`) but **ephemeral
sessions** (the tmux session is created when work arrives and
killed when work completes) and **a sandbox that lives across
sessions but dies on nuke** (a git worktree the polecat operates
in). Polecats are watched by the Witness and are the workers `gt
sling` and `gt convoy` dispatch beads to.

**Also known as:** cat, worker, ephemeral worker.

## Purpose

Most of the actual code Gas Town agents produce is built by
polecats. They exist so that:

- Bead-driven work has a worker shape: take a bead, build the
  change, push it, submit to the merge queue, exit.
- Workers can be spun up and torn down cheaply per-task without
  losing identity history.
- Multiple polecats can run in parallel in a single rig, each in
  their own git worktree, without stepping on each other.
- Failure modes have a watchdog: the Witness observes polecats,
  detects stalls, runs recoveries, and nukes when safe.
- The persistent identity allows a CV chain — every assignment a
  polecat completes accumulates on its agent bead, so its work
  history is available to future schedulers.

The framing in the [`gt polecat`](../commands/polecat.md) Long
help is canonical: **"Polecats have PERSISTENT IDENTITY but
EPHEMERAL SESSIONS. Each polecat has a permanent agent bead and
CV chain that accumulates work history across assignments."** And:
**"Cats build features. Dogs clean up messes."**

Without the polecat role there would be no agent shape that
combines persistent identity with cheap, parallelisable per-task
sandboxes. Crew workers are persistent but heavyweight (full
clones, manual lifecycle); dogs are reusable but cross-rig and
infrastructure-focused.

## Behavior / responsibilities

- Hold a stable identity (themed name, agent bead, CV chain)
  scoped to a single rig.
- Accept work via [`gt sling`](../commands/sling.md): a bead
  arrives, the polecat hooks it, transitions to the `working`
  state, and starts a tmux session in its sandbox worktree.
- Build the change in the sandbox: branch off, edit, commit,
  push.
- Submit the work to the merge queue and call
  [`gt done`](../commands/done.md), which writes a heartbeat
  with `state="exiting"`, signals completion, and the Witness
  takes over to nuke the sandbox.
- Touch the session heartbeat file regularly (heartbeat v2,
  with agent-reported state, so the Witness can distinguish
  "actively working" from "waiting at prompt" from "in gt done
  flow" from "self-reported stuck").
- Self-report stuck via [`gt heartbeat`](../commands/heartbeat.md)
  with `state="stuck"` if the agent realises it cannot proceed.
- Cycle through states: working → done (transient) → nuked, with
  the persistent identity rolling forward to the next assignment
  via `ReuseIdlePolecat` if the polecat was set up as part of an
  idle pool.

A polecat does NOT do cross-rig work, town-level coordination, or
infrastructure tasks. Those are dog and Mayor concerns.

## Lifecycle

The polecat lifecycle has three layered timescales: the persistent
identity (longest-lived), the sandbox worktree (until nuke), and
the tmux session (until current task completes). The
[`gt polecat`](../commands/polecat.md) Long help enumerates the
working states:

- **Working** — session active, hook bead set, agent doing
  assigned work.
- **Stalled** — session crashed mid-work; needs Witness
  intervention. Detected condition.
- **Zombie** — `gt done` was called but cleanup failed. Detected
  condition (StateDone persisting too long).
- **Nuked** — session ended, identity persists with
  `agent_state=nuked`, ready for the next assignment.

Plus an additional state from the persistent-polecat pool:

- **Idle** — work completed, session killed, sandbox preserved
  for reuse. Created by `gt polecat pool-init`.

**Spawn paths:**

- [`gt sling <bead> <rig>`](../commands/sling.md) — the primary
  entry. Allocates a name from the rig's themed pool, creates a
  worktree, hooks the bead, starts the session.
- [`gt polecat identity add`](../commands/polecat.md) — manual
  identity creation without an immediate session.
- `pool-init` — bulk creation of N polecats in idle state for a
  ready pool.
- `ReuseIdlePolecat` (in
  [`internal/polecat`](../packages/polecat.md)) — gives an
  existing idle polecat a new task.

**Run.** The agent works in the sandbox, touching its heartbeat
file (`<townRoot>/.runtime/heartbeats/<sessionName>.json`) on
each cycle. The Witness reads the heartbeat to know whether the
polecat is alive and what state it's in.

**Completion.** The polecat calls
[`gt done`](../commands/done.md) which:

1. Writes a heartbeat with `state="exiting"`.
2. Pushes the branch.
3. Submits to the merge queue.
4. Exits the agent process.

The Witness sees the exiting state and runs the nuke.

**Nuke.** [`gt polecat nuke`](../commands/polecat.md) is the
canonical destruction path. It's a 7-step protocol (kill tmux,
burn molecules, push branch as a guardrail, delete worktree,
delete LOCAL branch only, reset agent bead for reuse, purge
closed ephemeral beads). Local branch only because the
**Refinery** owns remote branch cleanup after the merge queue
commits — racing to delete the remote branch would break the
merge.

**Recovery.** If the polecat dies mid-work, the Witness or the
Deacon's stale-hook scan unhooks the bead and the Deacon
redispatches it to a fresh polecat. After
`DefaultMaxRedispatches = 3` failures the Deacon escalates the
bead to the Mayor.

## Interactions with other roles

- **Witness** — the per-rig watchdog. The
  [witness role (pending)](witness.md) is the polecat's
  immediate guardian: detects stalls, runs recoveries, nukes
  sandboxes, observes heartbeat v2 state.
- **Refinery** — the per-rig merge agent. The polecat's `gt
  done` submits to the merge queue; the
  [refinery role (pending)](refinery.md) processes the MR and
  cleans up the remote branch after merge. The polecat's nuke
  deliberately never deletes the remote branch to avoid racing
  the Refinery.
- **Mayor** — escalation target for NEEDS_RECOVERY verdicts and
  for redispatch loops that have given up. The
  [`mayor` role](mayor.md)'s ACP cleanup veto specifically
  protects polecat workspaces from being garbage-collected
  while a Mayor is reviewing their diffs.
- **Deacon** — runs the town-level redispatch and stale-hook
  patrols that operate on polecat work beads. The
  [`deacon` role](deacon.md) does NOT directly manage polecats
  but mechanically re-slings beads and unsticks hooks.
- **Crew** — the [`crew` role](crew.md) is the structural
  counterpart at the rig level: same scope, opposite
  semantics.
- **Dogs** — the [`dog` role](dog.md) handles cross-rig
  cleanup the polecats can't.
- **Sling and convoy** — `gt sling` is the spawn primitive;
  `gt convoy` batches work upstream of sling.

## Runtime substrate

A polecat is physically:

- **Identity:** an agent bead in the rig's beads database
  containing the CV chain, work history,
  `agent_state` (working/idle/done/nuked), `cleanup_status`,
  `hook_bead`, and `active_mr`.
- **Sandbox:** a git worktree of the rig's bare repo at
  `<rig>/polecats/<name>/<rigname>/` (new layout) or
  `<rig>/polecats/<name>/` (old layout). Each polecat gets its
  own worktree on its own branch.
- **Mailbox:** `<rig>/polecats/<name>/mail/`.
- **Lock:** `<rig>/.runtime/locks/polecat-<name>.lock`
  (per-polecat flock).
- **Heartbeat:** `<townRoot>/.runtime/heartbeats/<sessionName>.json`
  (heartbeat v2 with agent-reported state, see
  [`polecat` package](../packages/polecat.md) `heartbeat.go`).
- **Tmux session:** named `gt-<rig>-<name>` via
  `session.PolecatSessionName(prefix, name)`. Created on spawn,
  killed on done/nuke. Crew sessions share the same naming
  shape; downstream code distinguishes by directory layout.

The name comes from the rig's themed name pool — `mad-max` by
default, with `minerals` and `wasteland` as built-in alternatives,
and custom themes loaded from
`<townRoot>/themes/*.txt`. Reserved infra names (`witness`,
`mayor`, `deacon`, `refinery`, `crew`, `polecats`) are filtered
out of every theme so a polecat is never named after a town-level
agent.

The implementation is in [`internal/polecat`](../packages/polecat.md).

## Related wiki pages

- [internal/polecat](../packages/polecat.md) — Go package
  implementation (~2400 lines manager + 880 lines session
  manager + namepool + heartbeat + types).
- [gt polecat](../commands/polecat.md) — CLI entry point.
- [gt sling](../commands/sling.md) — primary spawn path.
- [gt done](../commands/done.md) — polecat-only completion
  primitive.
- [gt heartbeat](../commands/heartbeat.md) — agent-side
  heartbeat write.
- [crew role](crew.md) — the persistent counterpart.
- [dog role](dog.md) — the cross-rig counterpart.
- [mayor role](mayor.md) — escalation target.
- [deacon role](deacon.md) — town-level patroller of polecat
  work.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — agent env, runtime
  config, startup command builders.
- [`beads` package](../packages/beads.md) — agent beads, hook
  beads, `ResetAgentBeadForReuse`.
- [gt convoy](../commands/convoy.md) — upstream batching.

## Notes / open questions

- "Polecat" the role and "polecat" the command share a name with
  "polecat-safe" — a separate concept that gates which `gt`
  subcommands a polecat session is allowed to run. The
  polecat-safe annotation lives on each command and is
  enforced by the proxy server, not by this package.
- The persistent-identity / ephemeral-session split is the
  hardest part of the model to internalise. A polecat is a
  worker that survives the death of its current session because
  the agent bead and (until nuke) the worktree both persist.
  The same polecat name can carry many sessions over its life.
- Heartbeat v2 (gt-3vr5) is what makes the Witness's job
  tractable. Without agent-reported state, the Witness would
  have to infer "is this polecat done, idle, or stuck?" from
  timestamps alone. With agent-reported state, the Witness only
  needs the freshness check.
- The Witness role page is pending. Currently the Witness shows
  up as a participant in the polecat's lifecycle but is not
  documented at the role level.
- The Refinery role page is pending. Currently the Refinery
  shows up as the owner of remote branch cleanup but is not
  documented at the role level.
- Pool-initialised idle polecats have `agent_state="idle"`,
  which is not one of the four states listed in the polecat
  Long help (working / stalled / zombie / nuked). It's a
  pre-spawn state that exists specifically for the
  persistent-pool model.
