---
title: gt deacon
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/deacon.go
tags: [command, agents, deacon, tmux, patrol, watchdog, heartbeat, health-check, lifecycle]
---

# gt deacon

Lifecycle and operations commands for the
[Deacon](../roles/deacon.md) — the town-level watchdog that
receives mechanical heartbeats from the daemon and watches all
rigs. This page documents the `gt deacon` CLI subcommands only;
see the [Deacon role page](../roles/deacon.md) for persona
details and patrol semantics.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`deacon.go:35`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no
**Alias:** `gt dea`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/deacon.go`
(1644 lines). Registration: `deacon.go:396-468` attaches **15**
subcommands to `deaconCmd` and registers the parent on `rootCmd`.

The parent `deaconCmd` (`deacon.go:32-50`) uses `RunE:
requireSubcommand` — `gt deacon` alone prints the subcommand list.

### Invocation

```
gt deacon start    [--agent <alias>]
gt deacon stop
gt deacon attach   [--agent <alias>]
gt deacon status   [--json]
gt deacon restart  [--agent <alias>]
gt deacon heartbeat [action words...]
gt deacon health-check <agent> [--timeout=<d>] [--failures=<n>] [--cooldown=<d>]
gt deacon force-kill <agent> [--reason=<r>] [--skip-notify]
gt deacon health-state
gt deacon stale-hooks [--max-age=<d>] [--dry-run]
gt deacon pause [--reason=<r>]
gt deacon resume
gt deacon cleanup-orphans
gt deacon zombie-scan [--dry-run]
gt deacon redispatch <bead-id> [--rig <r>] [--max-attempts <n>] [--cooldown <d>]
gt deacon redispatch-state
gt deacon feed-stranded [--max-feeds <n>] [--cooldown <d>] [--json]
gt deacon feed-stranded-state
```

Role-shortcut note from the Long help at `deacon.go:49`: `"deacon"`
in mail/nudge addresses resolves to this agent.

### Subcommand groups

The 15 subcommands (17 if you count `deaconCmd.AddCommand` lines —
`heartbeat` and the two `-state` commands do count) break down into
lifecycle primitives, health/triage actions, and patrol helpers
called by the Deacon itself during its cycle.

**Lifecycle (`start` / `stop` / `attach` / `status` / `restart`):**

| subcommand | var | source | run-fn |
|---|---|---|---|
| `start` (alias `spawn`) | `deaconStartCmd` | `deacon.go:52-61` | `runDeaconStart` (`deacon.go:470-493`) |
| `stop`    | `deaconStopCmd`    | `deacon.go:63-70` | `runDeaconStop`    (`deacon.go:578-606`) |
| `attach` (alias `at`)  | `deaconAttachCmd`  | `deacon.go:72-81` | `runDeaconAttach`  (`deacon.go:608-629`) |
| `status`  | `deaconStatusCmd`  | `deacon.go:83-95` | `runDeaconStatus`  (`deacon.go:650-765`) |
| `restart` | `deaconRestartCmd` | `deacon.go:97-104`| `runDeaconRestart` (`deacon.go:767-795`) |

`runDeaconStart` calls `startDeaconSession` (`deacon.go:496-576`),
which creates the tmux session under `<townRoot>/deacon/` (so
`gt prime` detects the role correctly — `deacon.go:503-504`),
ensures runtime settings exist via `runtime.EnsureSettingsForRole`,
builds a startup beacon with the hard-coded prompt `"I am Deacon.
First run 'gt deacon heartbeat'. Then check gt hook, if empty
create mol-deacon-patrol wisp and execute it."` (`deacon.go:521`),
creates the session with the startup command directly (to avoid
the send-keys race documented at
`deacon.go:534`, issue #280), sets env vars from
`config.AgentEnv`, stores the pane id as `GT_PANE_ID` for
ZFC-compliant liveness checks (gt-qmsx — `deacon.go:551-554`),
applies the Deacon tmux theme, waits for Claude to start, and
accepts startup dialogs.

`runDeaconStop` tries a best-effort `C-c` interrupt, sleeps 100ms,
then `t.KillSessionWithProcesses` to ensure descendants are killed.

`runDeaconAttach` auto-starts the Deacon if missing, then calls the
shared `attachToTmuxSession` helper.

`runDeaconStatus` (`deacon.go:650-765`) gathers pause state
(`deacon.IsPaused`), tmux running state, and heartbeat state
(`deacon.ReadHeartbeat` → `HeartbeatStatus` JSON shape defined at
`deacon.go:640-648` with `timestamp`, `age_seconds`, `cycle`,
`last_action`, `fresh`, `stale`, `very_stale`). With `--json`, emits
a `DeaconStatusOutput` (`deacon.go:631-637`) combining all three.
Without `--json`, prints a pause block first if paused, then a
session block, then a heartbeat block with age, cycle, last action,
and health label (`fresh` / `stale` / `very stale`).

`runDeaconRestart` kills the session (with process kill), then
calls `runDeaconStart`.

**Heartbeat (`heartbeat`):**

`runDeaconHeartbeat` (`deacon.go:797-834`). Refuses to update if
the Deacon is paused — returns an error and prints the pause
reason. With no arguments, calls `deacon.Touch(townRoot)`. With
arguments, joins them and calls `deacon.TouchWithAction(townRoot,
action, 0, 0)`. The heartbeat signals liveness to the daemon so
the daemon does not poke (`deacon.go:113-115`).

**Health actions (`health-check` / `force-kill` / `health-state`):**

`runDeaconHealthCheck` (`deacon.go:838-…`). Takes a single agent
address, loads / creates state via
`deacon.LoadHealthCheckState(townRoot)`, checks cooldown, resolves
the agent to a bead ID and session name via `agentAddressToIDs`,
confirms the tmux session exists, then sends a HEALTH_CHECK nudge
via `t.NudgeSession` (using *immediate* delivery, not queued —
`deacon.go:882-886` explains that queued delivery would defer to
the next turn boundary and cause the 30s timeout to expire with
false negatives). Baseline is recorded after the nudge
(`deacon.go:891-896`). Exit codes documented at `deacon.go:136-139`:
`0` = responded/cooldown, `1` = error, `2` = should force-kill.

`runDeaconForceKill` (`deacon.go:966-…`) performs the protocol
documented at `deacon.go:152-163`: log intervention mail, kill
tmux session, update bead state to `"killed"`, optionally notify
mayor. Respects cooldown.

`runDeaconHealthState` (`deacon.go:1050-…`) prints the full health
check state: per-agent failure counts, last ping/response times,
force-kill history, cooldowns.

**Patrol helpers (`stale-hooks` / `cleanup-orphans` / `zombie-scan`
/ `redispatch` / `redispatch-state` / `feed-stranded` /
`feed-stranded-state`):**

- `stale-hooks` (`deacon.go:187-201`, `runDeaconStaleHooks` at
  `deacon.go:1178-…`): finds beads stuck in `hooked` status older
  than `--max-age` (default 1h) whose assignee agent is gone, and
  unhooks them. `--dry-run` previews.
- `cleanup-orphans` (`deacon.go:232-251`, `runDeaconCleanupOrphans`
  at `deacon.go:1337-…`): kills orphaned Claude subagent processes
  that have TTY `"?"` (no controlling terminal). Spawned by Task
  tool and accumulate.
- `zombie-scan` (`deacon.go:253-276`, `runDeaconZombieScan` at
  `deacon.go:1392-…`): detects Claude processes that are *not* the
  pane PID of any active tmux session and older than 60 seconds,
  cleaning "ghost" processes from dead sessions.
  `SuggestFor: []string{"orphan-scan", "orphan_scan", "orphan"}`
  at `deacon.go:255` redirects typos.
- `redispatch <bead-id>` (`deacon.go:278-305`,
  `runDeaconRedispatch` at `deacon.go:1465-…`): re-dispatches a
  recovered bead from a dead polecat. Rate-limited; escalates to
  Mayor on repeated failures. Exit codes: `0` success, `1` error,
  `2` cooldown, `3` skipped.
- `redispatch-state` (`deacon.go:307-317`,
  `runDeaconRedispatchState` at `deacon.go:1511-…`): prints
  re-dispatch attempt counts, cooldowns, escalations.
- `feed-stranded` (`deacon.go:319-348`, `runDeaconFeedStranded` at
  `deacon.go:1557-…`): runs `gt convoy stranded --json`,
  dispatches dogs for ready convoys via `gt sling`, auto-closes
  empty convoys via `gt convoy check`, and surfaces tracked-but-
  not-ready convoys for deacon review. Rate-limited (per-cycle
  max, per-convoy cooldown).
- `feed-stranded-state` (`deacon.go:350-360`,
  `runDeaconFeedStrandedState` at `deacon.go:1607-…`): prints
  feed-stranded tracking state.

**Pause controls (`pause` / `resume`):**

- `pause` (`deacon.go:203-221`, `runDeaconPause` at
  `deacon.go:1272-…`): writes a pause file that persists across
  restarts. Paused Deacon will not create patrol molecules, run
  health checks, or take autonomous actions. Accepts `--reason`.
- `resume` (`deacon.go:223-230`, `runDeaconResume` at
  `deacon.go:1309-…`): removes the pause file.

### Flags (summary)

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--json` | bool | `false` | `status` | `deacon.go:417` |
| `--timeout <d>` | duration | `30s` | `health-check` | `deacon.go:420` |
| `--failures <n>` | int | `3` | `health-check` | `deacon.go:422` |
| `--cooldown <d>` | duration | `5m` | `health-check` | `deacon.go:424` |
| `--reason <r>` | string | `""` | `force-kill` | `deacon.go:428` |
| `--skip-notify` | bool | `false` | `force-kill` | `deacon.go:430` |
| `--max-age <d>` | duration | `1h` | `stale-hooks` | `deacon.go:434` |
| `--dry-run` | bool | `false` | `stale-hooks`, `zombie-scan` | `deacon.go:436,444` |
| `--reason <r>` | string | `""` | `pause` | `deacon.go:440` |
| `--rig <r>` | string | `""` | `redispatch` | `deacon.go:448` |
| `--max-attempts <n>` | int | `0` (→3) | `redispatch` | `deacon.go:450` |
| `--cooldown <d>` | duration | `0` (→5m) | `redispatch` | `deacon.go:452` |
| `--max-feeds <n>` | int | `0` (→3) | `feed-stranded` | `deacon.go:456` |
| `--cooldown <d>` | duration | `0` (→10m) | `feed-stranded` | `deacon.go:458` |
| `--json` | bool | `false` | `feed-stranded` | `deacon.go:460` |
| `--agent <alias>` | string | `""` | `start`, `attach`, `restart` | `deacon.go:463-465` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [mayor.md](mayor.md) — sibling town-level agent. Same lifecycle
  shape; the Deacon is the watchdog, the Mayor is the coordinator.
- [boot.md](boot.md) — Boot triages the Deacon on each daemon
  tick; `boot.go:305-323` is where Boot reads the Deacon session
  and calls `deacon.NewManager(townRoot).Start("")` when missing.
- [heartbeat.md](heartbeat.md) — the lower-level heartbeat
  primitive; `gt deacon heartbeat` is the Deacon-specific wrapper.
- [patrol.md](patrol.md) — the patrol concept the Deacon executes.
- [plugin.md](plugin.md) — deacon patrol plugins live here.
- [hook.md](hook.md) — the startup prompt at `deacon.go:521` tells
  the Deacon to `"check gt hook"` for inbound hooked work.
- [convoy.md](convoy.md) — `gt deacon feed-stranded` calls
  `gt convoy stranded --json` and `gt convoy check` internally.
- [sling.md](sling.md) — `feed-stranded` and `redispatch` both
  dispatch via `gt sling`.
- [molecule.md](molecule.md) — `mol-deacon-patrol` molecule is
  named in the startup prompt (`deacon.go:521`).
- [callbacks.md](callbacks.md) — callbacks run "during Deacon
  patrol" (`callbacks.go:73`).
- [agents.md](agents.md) — Deacon is the second-sort agent
  (after Mayor).
- [orphans.md](orphans.md) — adjacent orphan-cleanup primitive.
- [dog.md](dog.md) — dogs are dispatched by the Deacon; see the
  dispatch path in `feed-stranded`.

## Notes / open questions

- **Largest command file in the batch.** 1644 lines, 15 subcommands,
  nine flag variables scoped to specific subcommands. The split
  between "lifecycle primitives" and "patrol helpers" is
  noticeable — the patrol helpers (`stale-hooks`, `redispatch`,
  `feed-stranded`, etc.) are really internal Deacon actions exposed
  as CLI commands so the Deacon agent can call them via tmux. An
  operator would rarely run them by hand.
- **`gt deacon heartbeat` is gated by pause.** A paused Deacon
  cannot update its own heartbeat, which means a paused Deacon
  will be detected as stale by the daemon eventually. Whether the
  daemon then tries to poke the paused Deacon, and what happens,
  is not visible from this file.
- **Immediate vs queued nudging.** `health-check` uses
  `NudgeSession` with immediate delivery rather than the
  mail-queue path. The comment at `deacon.go:882-886` is a good
  design note on why interruption is required for liveness tests.
- **`cleanup-orphans` and `zombie-scan` overlap.** The former uses
  TTY detection (`TTY == "?"`) and the latter uses tmux pane-PID
  verification. Operators with both in rotation should understand
  that `cleanup-orphans` may kill non-Gas-Town Claude processes
  that happen to lack a TTY.
- **`feed-stranded` indirectly depends on `gt convoy stranded`.**
  The exit contract on `gt convoy stranded --json` needs to match
  what `runDeaconFeedStranded` expects. Any change to convoy's
  JSON shape would silently break this command.
- **Role/concept page pending.** The Deacon as a persona
  (watchdog, heartbeat owner, patrol runner) is tracked in the
  Long help here but should live in `gastown/roles/deacon.md` —
  not yet created, pending Batch 6.
