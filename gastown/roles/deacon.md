---
title: deacon (role)
type: role
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/deacon/
  - /home/kimberly/repos/gastown/internal/cmd/deacon.go
tags: [role, agent, deacon, watchdog, town-level, patrol, heartbeat]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
---

# deacon (role)

The Deacon is Gas Town's town-level watchdog agent — the LLM
persona that runs mechanical patrols, watches the health of every
agent in the town, redispatches recovered beads, feeds stranded
convoys, and stands ready to be poked by the daemon when something
needs attention. The Deacon is the Mayor's mechanical sibling: where
the Mayor handles judgement and conversation, the Deacon runs
deterministic loops and escalates to the Mayor only when those
loops give up.

**Also known as:** the watchdog, the patroller.

## Purpose

A town with many rigs and many polecats produces a steady drip of
mechanical maintenance work: stale hooks to unstick, dead sessions
to clean up, recovered beads to re-dispatch, stranded convoys to
feed, hung dogs to consider clearing. Without a dedicated agent for
this work, every other agent would have to spend cycles on
housekeeping, and the human Overseer would get woken up for
problems that have obvious mechanical answers.

The Deacon exists so that:

- Routine maintenance never blocks on human attention.
- The town has a single auditable place where mechanical patrol
  decisions happen.
- The daemon has a known liveness signal (the heartbeat) that is
  cheap to read.
- Pause/resume can be applied cleanly across restarts via a file
  that survives Deacon death.

Without the Deacon there would be no agent role responsible for
running the patrol cycle. The Mayor is the wrong shape for it
(judgement-oriented, conversational); polecats and crew are
rig-scoped; dogs do the cross-rig work the Deacon dispatches.

## Behavior / responsibilities

- Touch the heartbeat file at the start of every wake cycle so
  the daemon knows the Deacon is alive.
- Read the pause file before any action; if paused, do nothing.
- Run patrol cycles: scan for stale hooks
  ([`gt deacon stale-hooks`](../commands/deacon.md)), redispatch
  recovered beads ([`gt deacon redispatch`](../commands/deacon.md)),
  feed stranded convoys
  ([`gt deacon feed-stranded`](../commands/deacon.md)), clean up
  orphan processes
  ([`gt deacon cleanup-orphans`](../commands/deacon.md)), scan for
  zombie sessions
  ([`gt deacon zombie-scan`](../commands/deacon.md)).
- Health-check other agents on demand
  ([`gt deacon health-check`](../commands/deacon.md)) and force-
  kill them when they're confirmed stuck
  ([`gt deacon force-kill`](../commands/deacon.md)).
- Escalate to the Mayor when a redispatch loop runs out of
  attempts.
- Dispatch dogs to do cross-rig infrastructure work (the
  `mol-convoy-feed` molecule is the standard route).
- Decide when a hung dog should be cleared — the dog Go code
  detects "hung" but defers the clearing decision to the Deacon
  per the ZFC principle (see [`dog` package](../packages/dog.md)).

## Lifecycle

A Deacon exists in exactly one of two states: not running, or
running in its `hq-deacon` tmux session. There is no headless mode.

- **Cold start.** [`gt deacon start`](../commands/deacon.md)
  creates the `hq-deacon` tmux session under `<townRoot>/deacon/`,
  injects the startup prompt
  `"I am Deacon. Start patrol: run gt deacon heartbeat, then check
  gt hook. If no hook, create mol-deacon-patrol wisp and execute
  it."` and lets the agent take over. The startup path uses
  `NewSessionWithCommand` to avoid a documented send-keys race
  (gh#280) and installs an auto-respawn hook so tmux keeps the
  pane alive across crashes.
- **Heartbeat loop.** Inside its patrol cycle the Deacon agent
  calls [`gt deacon heartbeat`](../commands/deacon.md), which goes
  through `deacon.Touch` / `deacon.TouchWithAction` to update
  `<townRoot>/deacon/heartbeat.json`. The daemon reads the file
  mtime and decides whether to poke. **A paused Deacon refuses
  to update its heartbeat**, which means a paused Deacon will
  eventually be flagged as very-stale by the daemon.
- **Boot triage.** [`gt boot`](../commands/boot.md) checks the
  Deacon on every daemon tick and starts one if missing
  (`boot.go:305-323` calls `deacon.NewManager(townRoot).Start("")`).
- **Pause / resume.** [`gt deacon pause`](../commands/deacon.md)
  writes a pause file at
  `<townRoot>/.runtime/deacon/paused.json`. The file persists
  across restarts. The Deacon agent must check it before every
  patrol action.
  [`gt deacon resume`](../commands/deacon.md) removes it.
- **Stop.** [`gt deacon stop`](../commands/deacon.md) sends a
  best-effort `C-c`, sleeps 100 ms, and hard-kills the session
  with all descendants. Tmux's auto-respawn hook will NOT bring
  the Deacon back after a Stop.
- **Restart.** Compose: stop + start. The Deacon's persistent
  state (heartbeat file, pause file, redispatch state, feed-
  stranded state, health-check state) all survives restarts.

## Interactions with other roles

- **Mayor** — the [`mayor` role](mayor.md) is the Deacon's
  judgement-oriented sibling. Mechanical loops that exhaust
  retry budgets escalate to the Mayor via mail
  (`REDISPATCH_FAILED`, etc.). The Mayor and Deacon do not share
  state but share the `hq-` session prefix.
- **Polecats** — the Deacon does not directly manage polecats,
  but it runs the redispatch loop that re-slings recovered
  polecat work and the stale-hook scan that frees beads stuck
  on dead polecats. The [`polecat` role](polecat.md) is the
  primary target of dispatched work.
- **Witnesses** — per-rig Witnesses are the source of
  RECOVERED_BEAD mail that triggers `Redispatch`. Witnesses also
  detect stalled polecats; the Deacon's stale-hook scan is the
  town-level safety net for cases the Witness misses.
- **Refinery** — independent of Deacon in normal flow.
- **Crew** — `assigneeToWorktreePath` in
  `internal/deacon/stale_hooks.go` knows about both polecat and
  crew worktree layouts, so the stale-hook scan covers
  [`crew` role](crew.md) workers as well.
- **Dogs** — the [`dog` role](dog.md) is the Deacon's helper
  pack. The Deacon owns the dog kennel, dispatches dogs to feed
  convoys, and decides when hung dogs need clearing. Dogs are
  invisible to `gt agents`; the Deacon is their only manager.
- **Boot** — Boot is the daemon's heartbeat trigger; it brings
  the Deacon up if missing and reads the heartbeat file to
  decide whether to nudge.

## Runtime substrate

The Deacon is a Claude (or compatible) process running in a
dedicated `hq-deacon` tmux session under `<townRoot>/deacon/`. The
session name is rig-less because there is exactly one Deacon per
town. Identity environment variables are written via
`config.AgentEnv` and the session has the auto-respawn hook
installed so tmux keeps the pane alive across crashes.

State the Deacon owns on disk:

- `<townRoot>/deacon/heartbeat.json` — wake-cycle liveness signal
  read by the daemon.
- `<townRoot>/deacon/.deacon-heartbeat` — legacy mtime marker for
  pre-JSON shell scripts.
- `<townRoot>/.runtime/deacon/paused.json` — pause file (note
  this is in `.runtime/` not `deacon/` — different lifetime).
- `<townRoot>/deacon/feed-stranded-state.json` — per-convoy feed
  cooldown tracking.
- `<townRoot>/deacon/redispatch-state.json` — per-bead
  redispatch attempt counters.
- `<townRoot>/deacon/health-check-state.json` — per-agent failure
  counts and force-kill cooldowns.

The Deacon also owns the dog kennel at `<townRoot>/deacon/dogs/`
(see the [`dog` role](dog.md) and [`dog` package](../packages/dog.md)).

The implementation is in [`internal/deacon`](../packages/deacon.md).

## Related wiki pages

- [internal/deacon](../packages/deacon.md) — Go package
  implementation.
- [gt deacon](../commands/deacon.md) — CLI entry point with 15
  subcommands.
- [mayor role](mayor.md) — sibling town-level agent.
- [dog role](dog.md) — the Deacon's helper pack.
- [polecat role](polecat.md) — primary patrol subjects.
- [crew role](crew.md) — also covered by stale-hook scans.
- [`session` package](../packages/session.md) — tmux substrate.
- [`config` package](../packages/config.md) — `AgentEnv`,
  `LoadOperationalConfig` for the deacon thresholds.
- [`beads` package](../packages/beads.md) — bead queries used by
  redispatch and stale-hook scanning.
- [gt boot](../commands/boot.md) — daemon-level Deacon watchdog.
- [gt convoy](../commands/convoy.md), [gt sling](../commands/sling.md)
  — the Deacon's primary dispatch tools.
- [gt heartbeat](../commands/heartbeat.md) — lower-level heartbeat
  primitive that the Deacon's wrapper builds on.

## Notes / open questions

- The Deacon is the only agent with a structured heartbeat file
  the daemon reads. The Mayor has no equivalent — Mayor liveness
  is checked from outside (Boot, `gt mayor status`) rather than
  self-reported.
- The Deacon's patrol primitives are deliberately split between
  Go (mechanical: rate limits, state files, shell-outs) and the
  Deacon agent itself (judgement: which hung dog to clear, when
  to escalate). The `feed_stranded.go` `needs_attention` action
  is a clean example of Go surfacing data without classifying it.
- A paused Deacon eventually becomes very-stale on the daemon's
  view because it can't update its heartbeat. Whether the daemon
  then attempts to poke a paused Deacon, and how the paused
  Deacon should respond, is not visible from this package alone.
- The Deacon is the Mayor's natural escalation target for
  mechanical patrol failures, but the reverse direction — the
  Mayor handing work to the Deacon — does not have a dedicated
  protocol. Mail flows in both directions but is generic.
