---
title: witness (role)
type: role
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/witness/
  - /home/kimberly/repos/gastown/internal/cmd/witness.go
tags: [role, agent, witness, per-rig, polecat-monitor, patrol, mountain-eater]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# witness (role)

The Witness is Gas Town's per-rig polecat health monitor — the
LLM persona that patrols a single rig, watches over its polecats,
detects stalled or crashed workers, nudges unresponsive sessions,
nukes zombies, escalates recoveries to the Mayor, and tracks
convoy failures for the Mountain-Eater auto-skip path. There is
exactly one Witness per rig. The Deacon monitors all Witnesses.

**Also known as:** the polecat monitor, the sentinel.

## Purpose

Polecats in Gas Town are ephemeral workers. They spawn on demand,
do one unit of work, call `gt done`, and exit. In a healthy world
the Witness would have nothing to do — polecats handoff cleanly
and nuke themselves after work. But polecats crash, hang, leak
sandboxes, miss their `gt done`, and occasionally get stuck in
states the polecat itself cannot escape.

The Witness exists so that:

- Polecat failures are detected per rig without the Deacon
  needing to scan every rig every tick.
- The "self-cleaning model" (polecats nuke themselves) has a
  fallback for the cases where self-cleaning didn't happen.
- Orphan beads (stuck in `hooked` state with no live polecat)
  are routinely reset and re-dispatched.
- Convoys with the `mountain` label can auto-skip issues that
  polecats repeatedly fail on, rather than infinite-looping on
  an impossible bead.
- Recovery escalations (NEEDS_RECOVERY) have a consistent home
  — the Witness escalates to the Mayor via
  [`internal/mayor`](../packages/mayor.md).

Without the Witness there would be no agent role responsible
for per-rig polecat health. The Deacon is town-scoped and can't
watch every polecat in every rig. The Refinery handles merges,
not workers. The polecats themselves can't observe their own
stuckness.

## Behavior / responsibilities

From the CLI Long help at
`/home/kimberly/repos/gastown/internal/cmd/witness.go:29-43`:

> The Witness patrols a single rig, watching over its polecats:
>   - Detects stalled polecats (crashed or stuck mid-work)
>   - Nudges unresponsive sessions back to life
>   - Cleans up zombie polecats (finished but failed to exit)
>   - Nukes sandboxes when polecats complete via 'gt done'
>
> The Witness does NOT force session cycles or interrupt working
> polecats. Polecats manage their own sessions (via gt handoff).
> The Witness handles failures and edge cases only.
>
> One Witness per rig. The Deacon monitors all Witnesses.

Concretely, the Witness agent runs the `mol-witness-patrol`
molecule, which calls into the following
[`internal/witness`](../packages/witness.md) library functions:

- `DetectZombiePolecats` — finds polecats whose session is gone
  or whose agent is dead. Classifies each as active-work-stale
  or orphan and decides whether to restart (for recoverable hook
  beads) or nuke.
- `DetectStalledPolecats` — finds polecats that are alive but
  stuck mid-work (hung, not progressing).
- `DetectOrphanedBeads` — finds beads in `hooked` state whose
  polecat is long gone; resets them so they can be re-dispatched.
- `DetectOrphanedMolecules` — finds parent molecules whose
  children are all closed but the parent never got closed.
- `DiscoverCompletions` — sweeps beads for `exit_type` signals
  from polecats whose `POLECAT_DONE` mail was lost.
- `HandlePolecatDone` / `HandleMerged` / `HandleMergeFailed` /
  `HandleLifecycleShutdown` / `HandleHelp` / `HandleSwarmStart`
  — one handler per protocol message type, driven by mail
  received in the Witness's inbox.
- `NukePolecat` / `AutoNukeIfClean` — the hard-cleanup
  primitives the Witness invokes via `gt polecat nuke`.
- `ShouldBlockRespawn` / `RecordBeadRespawn` — the spawn-count
  circuit breaker that prevents respawn storms.
- `TrackConvoyFailure` — the Mountain-Eater failure counter;
  auto-skips issues after 3 failures in a mountain-labelled
  convoy.
- `EscalateRecoveryNeeded` — mail-based escalation to the Mayor
  when the Witness can't recover safely on its own.

## Lifecycle

A Witness exists in exactly one of two states: not running, or
running in its `<rig-prefix>-witness` tmux session. There is no
headless mode.

- **Cold start.** [`gt witness start <rig>`](../commands/witness.md)
  calls `witness.NewManager(r).Start(false, agentOverride,
  envOverrides)`. The start path detects zombie sessions with
  TOCTOU mitigation — if the agent looks dead, it waits
  `ZombieKillGracePeriod`, re-checks, and aborts the kill if
  the agent started or the session was replaced in the interim.
  Loads role config via `LoadRoleDefinition`, builds a startup
  command through `buildWitnessStartCommand` (handling
  built-in Claude commands, non-Claude agents, and custom TOML
  `start_command` patterns), generates a UUID run ID, creates
  the session with `NewSessionWithCommand` (avoiding the
  send-keys race from gh#280), writes AgentEnv + role config env
  + CLI overrides, applies the rig theme, waits for Claude to
  be ready, accepts startup dialogs, starts the nudge poller,
  tracks the session PID, and records GASTA instantiation.
- **Foreground mode is vestigial.** The `--foreground` flag on
  `gt witness start` still parses, but `witness.Manager.Start`
  hard-rejects it — patrol logic has moved entirely to the
  `mol-witness-patrol` molecule driven by the Claude agent
  inside the tmux session.
- **Patrol loop.** Inside the session, the Witness agent runs
  `mol-witness-patrol` (the new home of patrol logic after
  the Go-side foreground loop was removed). Each patrol cycle
  invokes the detection functions in
  [`internal/witness`](../packages/witness.md), acts on the
  results via `gt polecat` subcommands, and optionally files
  wisps for cleanup work.
- **Stop.** [`gt witness stop <rig>`](../commands/witness.md)
  kills the tmux session with descendants. State is ZFC — no
  state file to clean up. The spawn-count state file survives
  restarts (that's the point — respawn budgets persist).
- **Restart.** Compose: stop + start. The spawn-count state,
  the operational config, and the rig config all survive the
  restart.
- **Boot triage.** The Deacon's health-check patrols monitor
  every Witness in the town. An unhealthy Witness gets nudged
  or force-killed by the Deacon.

One Witness per rig. Session name:
`<rig-prefix>-witness` via
`session.WitnessSessionName(session.PrefixFor(rigName))`.

## Interactions with other roles

- **Polecats** — the [`polecat` role](polecat.md) is the
  Witness's subject of observation. The Witness detects
  failures, issues `gt polecat nuke` / `gt polecat
  check-recovery` / `gt polecat git-state` via the patrol
  molecule, and watches polecat agent beads for state and
  `exit_type` signals.
- **Refinery** — the [`refinery` role](refinery.md) is the
  sibling per-rig agent. When a polecat completes with a
  pending MR, the Witness notifies the Refinery via
  `notifyRefineryMergeReady` / `MERGE_READY` protocol. They
  don't share state; the bead ecosystem is the communication
  medium.
- **Mayor** — the [`mayor` role](mayor.md) is the Witness's
  escalation target for NEEDS_RECOVERY situations. The
  Witness calls `EscalateRecoveryNeeded` (mail-backed) when a
  polecat has unmerged work and cannot be recovered mechanically.
- **Deacon** — the [`deacon` role](deacon.md) monitors all
  Witnesses. Per the Long help, "The Deacon monitors all
  Witnesses." When a Witness is unhealthy, the Deacon is the
  one that detects and responds. Witnesses also push
  `RECOVERED_BEAD` mail upstream to the Deacon for
  re-dispatching.
- **Crew** — independent of Witness in normal flow. Crew
  workers have their own stale-hook surface (handled by the
  Deacon's `stale-hooks` scan, not the Witness).
- **Dogs** — the [`dog` role](dog.md) may be dispatched to do
  cleanup work the Witness identifies, but Witnesses don't
  directly dispatch dogs; that's the Deacon's job.

## Runtime substrate

The Witness is a Claude (or compatible) process running in a
dedicated `<rig-prefix>-witness` tmux session under
`<rig>/witness/rig/` (preferred) or `<rig>/witness/` as fallback.
The working directory is whichever of those exists — see
`Manager.witnessDir()`.

State the Witness codepath owns on disk:

- `<townRoot>/witness/bead-respawn-counts.json` — spawn-count
  state. Cross-process serialized via a sibling `.flock` file.
  Survives restarts by design.
- Runtime settings at `<rig>/witness/settings/` via
  `runtime.EnsureSettingsForRole`.

State the Witness codepath shares with callers:

- The rig's beads database — agent bead state transitions
  (spawning → running → done → nuked), cleanup wisps, orphan
  resets.
- The in-memory `MessageDeduplicator` per Witness session —
  prevents restart-time re-reads from double-processing the
  same mail.

The implementation is in
[`internal/witness`](../packages/witness.md).

## Related wiki pages

- [internal/witness](../packages/witness.md) — Go package
  implementation.
- [gt witness](../commands/witness.md) — CLI entry point with
  5 subcommands.
- [refinery role](refinery.md) — sibling per-rig agent.
- [polecat role](polecat.md) — subject of observation.
- [mayor role](mayor.md) — escalation target.
- [deacon role](deacon.md) — upstream monitor.
- [wisp concept](../concepts/wisp.md) — cleanup wisps are the
  Witness's primary externalized output.
- [convoy concept](../concepts/convoy.md) — Mountain-Eater
  tracks failures per convoy.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — `AgentEnv`,
  operational config thresholds, `LoadRoleDefinition`.
- [`beads` package](../packages/beads.md) — `AgentState`,
  `AgentFields`, `RoleConfig`, `ExpandRolePattern`.
- [`mail` package](../packages/mail.md) — protocol message
  delivery.
- [`nudge` package](../packages/nudge.md) — alternative to mail
  for non-durable notifications.
- [`polecat` package](../packages/polecat.md) — the Witness's
  library for touching polecat state.
- [gt polecat](../commands/polecat.md) — commands the Witness
  invokes during patrol.

## Notes / open questions

- **The Witness does NOT force session cycles.** Per the Long
  help: "Polecats manage their own sessions (via gt handoff).
  The Witness handles failures and edge cases only." This is
  the cleanest statement of the separation: the polecat owns
  its own liveness; the Witness owns its failure recovery.
- **"There is no 'idle' state."** Also from the Long help:
  polecats either have work or don't exist. The Witness
  enforces this by nuking polecats whose work is complete but
  whose session didn't exit cleanly.
- **`DetectZombiePolecats` hands off to Mountain-Eater
  automatically.** `trackConvoyFailures` is called inside
  `DetectZombiePolecats`, so the Mountain-Eater is baked into
  the zombie sweep — it's not a separate patrol step.
- **Three completion-detection paths exist.** POLECAT_DONE mail,
  bead `exit_type` via `DiscoverCompletions`, and patrol sweeps
  via `DetectZombiePolecats`. When these disagree, the Witness's
  behaviour depends on which sweep ran first. The dedup layer
  prevents double-processing within a session but not across.
- **Spawn-count circuit breaker survives Witness restarts.**
  This is the primary defense against "clown show #22" — a
  buggy bead that keeps respawning workers forever. The
  `.flock` serialization means even a Witness restart mid-
  respawn doesn't reset the counter.
- **The Witness has no self-reported heartbeat.** Unlike the
  Deacon, the Witness's liveness is observed from outside (the
  Deacon's health-check scan). A future enhancement might add
  a per-Witness heartbeat file for tighter observability.
