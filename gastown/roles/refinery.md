---
title: refinery (role)
type: role
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/refinery/
  - /home/kimberly/repos/gastown/internal/cmd/refinery.go
tags: [role, agent, refinery, per-rig, merge-queue, engineer, batch, bisect]
---

# refinery (role)

The Refinery is Gas Town's per-rig merge queue processor — the
LLM persona that serializes all merges to main for a single rig,
rebases work branches, runs quality gates, performs squash
merges, and if conflicts or test failures appear, spawns a fresh
polecat to re-implement the work. Where polecats write code and
the Witness watches their health, the Refinery is the chokepoint
through which all merged work must pass. There is exactly one
Refinery per rig.

**Also known as:** the merger, the Engineer (its internal type
name), the queue processor.

## Purpose

In a town where multiple polecats work in parallel on the same
rig, two problems arise that cannot be solved at the polecat
level:

- **Merge serialization.** Two polecats that both finished their
  work at nearly the same time cannot safely both push to main
  — one merge might invalidate the other's tests, introduce
  conflicts, or race the push. Somebody has to decide the order.
- **Conflict recovery.** When a polecat's branch can no longer
  merge cleanly (because main has moved, another merge landed
  first, or a test started failing on the rebased code), the
  polecat that wrote it is already gone. The code exists but
  has no owner. Somebody has to decide whether to re-implement,
  reject, or escalate.

The Refinery exists so that:

- Merges happen in a single, deterministic order per rig.
- Every merge runs the same quality gates before landing.
- When conflicts or failures occur, the response is
  programmatic (spawn a fresh polecat to rework, or escalate)
  rather than requiring human attention.
- Merge-queue health (stale claims, orphaned branches, blocked
  MRs) is observable to operators.

Without the Refinery, every polecat would have to serialise its
own merges, know every other polecat's state, and handle
conflict recovery independently — a coordination problem that
does not scale.

## Behavior / responsibilities

From the CLI Long help at
`/home/kimberly/repos/gastown/internal/cmd/refinery.go:34-48`:

> The Refinery serializes all merges to main for a rig:
>   - Receives MRs submitted by polecats (via gt done)
>   - Rebases work branches onto latest main
>   - Runs validation (tests, builds, checks)
>   - Merges to main when clear
>   - If conflict: spawns FRESH polecat to re-implement
>     (original is gone)
>
> Work flows: Polecat completes → gt done → MR in queue →
> Refinery merges. The polecat is already nuked by the time the
> Refinery processes.

In practice, the Refinery's day-to-day responsibilities are:

- Poll the merge queue (`ListReadyMRs` / `Engineer` in the
  [`refinery` package](../packages/refinery.md)).
- Claim an MR via `ClaimMR` so no other Refinery worker picks
  it up. Claims have a 30-minute default TTL.
- Fetch the source branch, rebase onto the target, run the
  configured quality gates (pre-merge and post-squash phases).
- If `batch_mode` is enabled, assemble a batch of up to 5 MRs,
  stack them on target, run gates once, and on failure use
  bisection to identify culprits without penalising innocents.
- Acquire the main-push slot via the beads-backed distributed
  lock so concurrent Refinery workers don't race pushes.
- Perform the squash merge, push, and run post-merge cleanup
  (close the MR bead with reason `merged`, force-close the
  source issue with `Merged in <MR-ID>`, sync crew workspaces,
  prune stale remote refs).
- If the merge closes the last tracked issue in a convoy, fire
  the convoy completion path — close the convoy and land the
  swarm.
- On conflict, create a synthesis task bead that spawns a fresh
  polecat to re-implement the work with the conflict as context.
- On repeated failures, escalate to the Mayor.

## Lifecycle

A Refinery exists in exactly one of two states: not running, or
running in its `<rig-prefix>-refinery` tmux session. There is no
headless mode.

- **Cold start.** [`gt refinery start`](../commands/refinery.md)
  calls `refinery.NewManager(r).Start(false, agentOverride)`,
  which detects and kills zombie sessions, resolves (or
  auto-repairs) the `<rig>/refinery/rig` worktree from the
  shared `.repo.git` bare repo, loads runtime settings,
  generates a UUID run ID for GASTA tracing, creates the tmux
  session with `NewSessionWithCommand` (avoiding the send-keys
  race from gh#280), writes the `GT_REFINERY=1` env var plus
  the standard agent identity vars, applies the rig theme,
  accepts startup dialogs, waits for Claude to reach a ready
  prompt, starts the nudge-queue poller, and records the
  agent instantiation event.
- **Foreground mode is deprecated.** The `--foreground` flag on
  `gt refinery start` is still live in the CLI but
  `Manager.Start(true, ...)` hard-rejects it — refinery no
  longer has a Go-side foreground loop; the Claude agent drives
  the merge cycle.
- **Patrol loop.** Inside the tmux session, the Refinery agent
  polls the queue, claims MRs, and merges them. The claim
  protocol uses `GT_REFINERY_WORKER` for worker ID (defaulting
  to `refinery-1`) so multiple Refinery instances can cooperate
  via `refinery claim` / `refinery release`.
- **Stop.** [`gt refinery stop`](../commands/refinery.md) kills
  the tmux session via `Manager.Stop()`. State is ZFC — no
  state file to clean up. In-flight claims time out after 30
  minutes and become eligible for re-claim by another Refinery
  worker.
- **Restart.** Compose: stop + start. The worktree survives the
  restart; the queue state lives in beads and is untouched.
- **Boot triage.** The Deacon's patrol loop
  ([`gt deacon health-check`](../commands/deacon.md)) monitors
  Refineries alongside Witnesses. A Refinery detected as
  unhealthy gets nudged or force-killed by the Deacon.

One Refinery per rig. The session name is
`<rig-prefix>-refinery` via
`session.RefinerySessionName(session.PrefixFor(rigName))`.

## Interactions with other roles

- **Polecats** — the [`polecat` role](polecat.md) writes the
  code the Refinery merges. By the time an MR reaches the
  Refinery's queue, the polecat is already gone
  (nuked by `gt done`), so the Refinery never talks to the
  original author. On conflict, the Refinery creates a fresh
  polecat via a synthesis task bead.
- **Witness** — the [`witness` role](witness.md) and the
  Refinery share a rig but don't directly interact. The Witness
  handles polecat health; the Refinery handles merge health.
  Both escalate to the Mayor independently.
- **Mayor** — the [`mayor` role](mayor.md) is the Refinery's
  escalation target for exhausted retries. The Refinery's
  `MaxRetryCount` (default 5) is the threshold before a
  conflict gets escalated rather than auto-synthesised.
- **Deacon** — the [`deacon` role](deacon.md) runs health checks
  that monitor Refinery liveness alongside Witnesses. A Refinery
  that's hung or crashed will be detected by the Deacon and
  either nudged or force-killed.
- **Crew** — the [`crew` role](crew.md) workspaces are synced
  by the Refinery after every merge via `syncCrewWorkspaces`
  so the human's checkout stays current with merged work.
- **Other Refineries (parallel workers)** — when multiple
  Refinery workers cooperate (via `GT_REFINERY_WORKER` IDs),
  they coordinate via the `ClaimMR` / `ReleaseMR` protocol and
  the beads-backed main-push slot. No direct messaging; all
  coordination is through bead state.
- **Dogs** — the [`dog` role](dog.md) may be dispatched by the
  Deacon to do pre/post merge infrastructure tasks (e.g. wisp
  cleanup), but dogs and Refineries don't share state.

## Runtime substrate

The Refinery is a Claude (or compatible) process running in a
dedicated `<rig-prefix>-refinery` tmux session under
`<rig>/refinery/rig/`. The working directory is a git worktree
that shares the bare repo with polecats and mayor, so rebases
and fast-forwards see the same object database.

Auto-repair: if the `refinery/rig` worktree is missing, the
start path attempts to recreate it from the shared `.repo.git`
via `git worktree add`, falling back to `<rig>/mayor/rig` only
if the bare repo is absent. The fallback-to-mayor path is
correct for rigs using a standard `.git` clone (e.g. beads),
but for rigs with a bare-repo layout the auto-repair is the
preferred outcome.

State the Refinery's codepath owns on disk:

- The `refinery/rig` worktree (shared with bare repo).
- Runtime settings at `<rig>/refinery/settings/` via
  `runtime.EnsureSettingsForRole`.

State the Refinery DOES NOT own:

- No state file for running status (ZFC — tmux session is
  truth).
- No state file for the queue (beads is truth).
- No state file for claims (beads fields on the MR are truth).

The implementation is in
[`internal/refinery`](../packages/refinery.md).

## Related wiki pages

- [internal/refinery](../packages/refinery.md) — Go package
  implementation (Manager, Engineer, Batcher, Scorer).
- [gt refinery](../commands/refinery.md) — CLI entry point
  with 11 subcommands (start/stop/restart/attach/status/queue/
  ready/blocked/unclaimed/claim/release).
- [witness role](witness.md) — sibling per-rig agent.
- [mayor role](mayor.md) — escalation target.
- [deacon role](deacon.md) — health watchdog.
- [polecat role](polecat.md) — upstream of the merge queue.
- [crew role](crew.md) — downstream of every merge.
- [wisp concept](../concepts/wisp.md) — MRs are wisps stored
  in the shared `wisps` table; this is why
  `beads.ListMergeRequests` returns what it does.
- [convoy concept](../concepts/convoy.md) — the Refinery's
  post-merge path closes completed convoys.
- [convoy-launch workflow](../workflows/convoy-launch.md) — the
  landing phase is driven by the Refinery.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — role-agent
  resolution, runtime settings, merge queue config.
- [`beads` package](../packages/beads.md) — merge-slot protocol,
  MR fields parser, queue queries.
- [gt done](../commands/done.md) — the polecat-side completion
  command that submits MRs to the queue.
- [gt sling](../commands/sling.md) — used to respawn fresh
  polecats for conflict recovery.

## Notes / open questions

- **The Refinery never talks to the original polecat.** By the
  time an MR is processed, the polecat is already gone. All
  "spawn a fresh polecat to rework" flows go through synthesis
  beads and `gt sling`. This is the "original is gone" line in
  the CLI Long help.
- **The Refinery runs without a dedicated agent config file of
  its own.** Role resolution goes through `ResolveRoleAgentConfig`
  like the other per-rig agents; the `role_agents` mapping is
  in the rig config.
- **Parallel-worker support exists but isn't used by default.**
  The `GT_REFINERY_WORKER` env var + `claim`/`release`
  subcommands are wired up, but `getWorkerID()` defaults to
  `refinery-1` with no deduplication. Two processes with no env
  var set will both be `refinery-1` and step on each other's
  claims.
- **Batch-then-bisect is opt-in.** With `batch.max_batch_size
  <= 1`, the Engineer falls through to sequential single-MR
  processing. The batch mode exists but isn't the default.
- **The Refinery is the point where wisps become "permanent"
  work.** Until an MR is merged, its source issue is still
  ephemeral in the queue's view. After merge, the source issue
  is force-closed with `Merged in <MR-ID>` and the work is
  permanent. This is the transition from "in progress" to
  "durable".
