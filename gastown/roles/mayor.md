---
title: mayor (role)
type: role
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/mayor/
  - /home/kimberly/repos/gastown/internal/cmd/mayor.go
tags: [role, agent, mayor, town-level, orchestrator, acp]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
---

# mayor (role)

The Mayor is Gas Town's town-level orchestrator agent — the LLM
persona that sits at the top of the agent hierarchy and acts as the
human Overseer's "Chief of Staff" for cross-rig coordination,
escalation handling, and town-wide decision making. Where polecats
build features in a single rig and the Deacon runs mechanical
patrols, the Mayor reasons across rigs, talks to the human, and
owns judgement calls that cannot be reduced to a script.

**Also known as:** Chief of Staff, town agent.

## Purpose

A Gas Town town has many rigs, each running its own agents. Most
day-to-day work happens at the rig level — a polecat picks up a
bead, builds the change, the Refinery merges it. But several
problems must be handled at the town scope:

- **Cross-rig coordination.** Work that spans multiple rigs (a
  protocol bump, a shared infra change, a coordinated rollout)
  has no natural home at the rig level.
- **Escalations.** When a polecat hits something it cannot solve
  — a NEEDS_RECOVERY verdict from
  [`gt polecat check-recovery`](../commands/polecat.md), a
  redispatch loop the Deacon has given up on, a stranded convoy
  with tracked-but-not-ready issues — the Mayor is the human-in-
  the-loop's nominated proxy.
- **Conversation with the Overseer.** The Mayor is the agent the
  human talks to first. `"mayor"` resolves as a mail/nudge address
  to this agent (`internal/cmd/mayor.go:39`).
- **Owning the town.** Town-level state — `mayor/town.json`,
  `mayor/config.json`, the rig registry, the merge queue
  configuration — is conceptually under the Mayor's stewardship,
  even when individual write paths run elsewhere.

Without the Mayor there would be no agent role responsible for
cross-rig judgement. The Deacon runs deterministic patrols; the
Witness keeps a single rig honest; the Refinery merges. None of
those are conversational, and none of them think across rig
boundaries.

## Behavior / responsibilities

- Receive escalations from the Deacon (redispatch loops,
  stale-hook recoveries with partial work) and from witnesses
  (NEEDS_RECOVERY polecats with unmerged work).
- Process Mayor-routed mail and callbacks via the
  [`gt callbacks`](../commands/callbacks.md) inbox.
- Make judgement calls about which polecats to nuke vs. recover,
  which convoys to dispatch, which slings to retry.
- Talk to the Overseer about anything the agents cannot resolve
  on their own.
- In ACP mode: review polecat worker diffs in an IDE before they
  are GC'd. The ACP cleanup veto (`internal/mayor/cleanup.go:144`)
  is what makes this safe.

## Lifecycle

A Mayor exists in exactly one of three modes at any time: not
running, running in tmux mode (interactive), or running in ACP mode
(headless, IDE-driven). The transitions are managed by
[`gt mayor`](../commands/mayor.md):

- **Cold start (tmux mode).** [`gt mayor start`](../commands/mayor.md)
  → `Manager.StartTMUX` creates the `hq-mayor` tmux session under
  `<townRoot>/mayor/`, sends a startup beacon with topic
  `cold-start`, and the Mayor begins its work loop.
- **Cold start (ACP mode).** [`gt mayor acp`](../commands/mayor.md)
  → `Manager.StartACP` runs the agent in headless mode connected
  via the Agent Client Protocol. The agent must support ACP (e.g.
  `opencode`); Claude is rejected with a clear error.
- **Mode transition.** Only `gt mayor attach` is allowed to
  upgrade an ACP Mayor into a tmux Mayor. The CLI gracefully
  stops the ACP proxy first, then attaches to (or respawns) the
  tmux session.
- **Self-healing attach.** If the tmux session exists but the
  agent inside has died, `gt mayor attach` rebuilds the startup
  command, kills the pane processes, and respawns the pane in
  place — the same session ID is preserved.
- **Stop.** [`gt mayor stop`](../commands/mayor.md) sends a
  best-effort `C-c`, waits 100 ms, then hard-kills the session
  and all descendant processes. The ACP sidecar files are not
  touched (they belong to the ACP lifecycle).
- **Boot triage.** [`gt boot`](../commands/boot.md) checks the
  Mayor on every daemon tick and may nudge or restart it.

There is **one Mayor per machine.** Mayor's session name is the
constant `"hq-mayor"` with no rig component, so multi-town setups
require container or VM isolation.

## Interactions with other roles

- **Deacon** — the Mayor's sibling town-level agent. The
  [`deacon` role](deacon.md) handles mechanical patrols and
  escalates judgement-required problems to the Mayor via mail.
  Mayor and Deacon do not share state but share the `hq-` session
  prefix.
- **Polecats** — the [`polecat` role](polecat.md) escalates
  NEEDS_RECOVERY situations to the Mayor. While a Mayor runs in
  ACP mode, polecat worker workspaces cannot be garbage-collected
  (the cleanup veto), so the Mayor can review their diffs.
- **Witness** — per-rig Witnesses send escalations to the Mayor
  when they cannot resolve a polecat's state safely.
- **Refinery** — per-rig merge agents do not directly talk to the
  Mayor in normal flow; failures bubble up via beads and mail.
- **Crew** — the [`crew` role](crew.md) is human-managed and
  routes through the Mayor for cross-rig coordination as needed.
- **Dogs** — the [`dog` role](dog.md) is dispatched by the Deacon,
  not by the Mayor directly.
- **Boot** — Boot is the daemon's heartbeat trigger and brings
  the Mayor up via `mayor.NewManager(townRoot).Start("")` if
  missing.
- **Overseer (human)** — the Mayor is the human's nominated proxy
  for routine town-level decisions. Conversations with the human
  pass through Mayor.

## Runtime substrate

In **tmux mode** the Mayor is a Claude (or compatible) process
running in a dedicated `hq-mayor` tmux session under
`<townRoot>/mayor/`. The session name is rig-less because there is
exactly one Mayor per town. Identity environment variables are
written via `config.AgentEnv` and the `CLAUDE_CONFIG_DIR` is
resolved from `mayor/accounts.json`.

In **ACP mode** the Mayor is a headless agent process attached to
its IDE / ACP client over stdio, with no tmux session at all. The
liveness signal is a PID file at
`<townRoot>/mayor/mayor-acp.pid` and a sidecar agent name at
`mayor-acp.agent`. While the PID is alive,
`mayor.IsACPActive(townRoot)` returns true and the cleanup veto
(`mayor.CleanupVetoChecker`) blocks polecat GC.

The Mayor's working directory is always `<townRoot>/mayor/`. Town-
level files (`town.json`, `config.json`, `accounts.json`,
`overseer.json`, ACP sidecars) all live under that directory and
the Mayor reads/writes them in place.

The implementation is in [`internal/mayor`](../packages/mayor.md).

## Related wiki pages

- [internal/mayor](../packages/mayor.md) — Go package
  implementation.
- [gt mayor](../commands/mayor.md) — CLI entry point.
- [deacon role](deacon.md) — sibling town-level agent.
- [polecat role](polecat.md) — workers Mayor escalates to / from.
- [crew role](crew.md) — user-managed counterpart.
- [`session` package](../packages/session.md) — tmux substrate
  (Mayor uses `MayorSessionName`, `StartSession`,
  `FormatStartupBeacon`).
- [`config` package](../packages/config.md) — `AgentEnv`,
  `ResolveAccountConfigDir`, `RuntimeConfigSupportsACP`.
- [`acp` package](../packages/go-packages.md) — ACP proxy /
  propeller subsystem the headless Mayor uses.
- [gt boot](../commands/boot.md) — daemon-level Mayor watchdog.
- [gt callbacks](../commands/callbacks.md) — the Mayor's mail
  inbox processor.
- [gt escalate](../commands/escalate.md) — the standard escalation
  primitive that lands work in Mayor's queue.

## Notes / open questions

- The Mayor is the only agent in Gas Town that can run in two
  fundamentally different runtime substrates (tmux vs. ACP). The
  invariant that they are mutually exclusive is enforced by
  `Manager.Start` and `Manager.StartACP` both checking the other
  mode first.
- The Mayor does not own a heartbeat file the way the Deacon
  does. Liveness is checked from outside (Boot, `gt mayor
  status`) rather than self-reported.
- "Mayor as Chief of Staff" framing comes from the
  [`gt mayor`](../commands/mayor.md) Long help and the architecture
  docs, not from the package code itself. The package code only
  knows about sessions, modes, and PIDs.
- A future role page for the Overseer (the human) is implied but
  not yet created. The Overseer is the Mayor's principal.
