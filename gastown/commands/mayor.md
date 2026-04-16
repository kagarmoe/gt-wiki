---
title: gt mayor
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/mayor.go
tags: [command, agents, mayor, tmux, acp, lifecycle, session]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt mayor

Lifecycle commands for the [Mayor](../roles/mayor.md) — the
town-level agent that acts as the Overseer's "Chief of Staff" for
cross-rig coordination. This page documents the `gt mayor` CLI
subcommands only; see the [Mayor role page](../roles/mayor.md) for
responsibilities and decision-making.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`mayor.go:25`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no
**Alias:** `gt may`

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/mayor.go`
(487 lines). Registration: `mayor.go:118-137` attaches six
subcommands to `mayorCmd` and registers the parent on `rootCmd`.

The parent `mayorCmd` (`mayor.go:22-40`) uses `RunE:
requireSubcommand` as its fallback, so `gt mayor` with no arguments
prints the subcommand list.

### Invocation

```
gt mayor start    [--agent <alias>]
gt mayor stop
gt mayor attach   [--agent <alias>]
gt mayor status   [--running]
gt mayor restart  [--agent <alias>]
gt mayor acp      [--rig <name>] [--town <dir>] [--agent <alias>]
```

Role-shortcut note from the Long help at `mayor.go:39`: `"mayor"` in
mail/nudge addresses resolves to this agent.

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `start`   | `mayorStartCmd`   | `mayor.go:47-55`   | `runMayorStart`   (`mayor.go:153-172`) |
| `stop`    | `mayorStopCmd`    | `mayor.go:57-64`   | `runMayorStop`    (`mayor.go:174-190`) |
| `attach`  | `mayorAttachCmd`  | `mayor.go:66-75`   | `runMayorAttach`  (`mayor.go:192-293`) |
| `status`  | `mayorStatusCmd`  | `mayor.go:77-82`   | `runMayorStatus`  (`mayor.go:336-387`) |
| `restart` | `mayorRestartCmd` | `mayor.go:84-91`   | `runMayorRestart` (`mayor.go:389-402`) |
| `acp`     | `mayorAcpCmd`     | `mayor.go:93-113`  | `runMayorAcp`     (`mayor.go:461-487`) |

`gt mayor attach` aliases to `gt mayor at` (`mayor.go:68`).

### start / stop / restart

`runMayorStart` (`mayor.go:153-172`): gets a `mayor.Manager` for the
town root, calls `mgr.Start(agentOverride)`, and converts
`mayor.ErrAlreadyRunning` into a human-readable error pointing at
`gt mayor attach`.

`runMayorStop` (`mayor.go:174-190`): calls `mgr.Stop()`, converts
`mayor.ErrNotRunning` into `"Mayor session is not running"`.

`runMayorRestart` (`mayor.go:389-402`): best-effort Stop (ignoring
`ErrNotRunning`), then calls `runMayorStart`. The stop path does not
wait for clean shutdown beyond whatever `mgr.Stop()` itself blocks
on.

### attach — the most intricate subcommand

`runMayorAttach` (`mayor.go:192-293`) is the only subcommand with
significant lifecycle logic:

1. **ACP graceful transition** (`mayor.go:205-210`). If
   `mayor.IsACPActive(townRoot)` returns true, the ACP proxy must
   be shut down before tmux can take over. The in-file comment
   says: "Only 'gt mayor attach' is allowed to transition from ACP
   to tmux mode." Transition is performed by
   `gracefullyShutdownACP` (`mayor.go:297-334`), which removes the
   PID file (a signal to the ACP proxy to exit gracefully), then
   waits up to 3 seconds (30 × 100ms) for the process to die,
   falling back to `process.Kill()` if it hangs.

2. **Infrastructure bring-up** via `ensureMayorInfra`
   (`mayor.go:409-455`). Non-fatal for the daemon, **fatal for
   Dolt**:
   - Loads `daemon.json` env vars (e.g. `GT_DOLT_PORT`) so Dolt
     uses the right port (`mayor.go:411-415`).
   - If the daemon is not running, prints a warning and calls
     `ensureDaemon(townRoot)` — a daemon start failure only logs a
     warning and continues.
   - If Dolt is local (`doltCfg.IsRemote()` is false) and its data
     dir exists, and the server is not running, `doltserver.Start`
     is called. A Dolt start failure is converted into an enriched
     error message that looks up the port holder via
     `doltserver.PortHolder` and suggests a free port via
     `doltserver.FindFreePort`, producing a concrete remediation
     hint: `gt config set dolt.port <N> && gt mayor at`
     (`mayor.go:436-448`).

3. **Session presence check and respawn**
   (`mayor.go:220-289`). If the tmux session exists but Claude is
   no longer alive inside it (`t.IsAgentAlive(sessionID)` uses
   descendant-process checks because the Mayor launches via a bash
   wrapper — the comment at `mayor.go:231-234` flags this), the
   pane is respawned with proper startup context:
   - Builds a startup beacon via `session.FormatStartupBeacon` with
     `Recipient: "mayor"`, `Sender: "human"`, `Topic: "attach"`.
   - Builds the full startup command via
     `config.BuildAgentStartupCommandWithAgentOverride`.
   - Resolves `CLAUDE_CONFIG_DIR` via
     `config.ResolveAccountConfigDir` (against
     `constants.MayorAccountsPath(townRoot)`), falling back to the
     env var. Prepends it as an env prefix on the startup command
     and also calls `t.SetEnvironment` for the session.
   - Sets `remain-on-exit` on the pane (so killing processes does
     not destroy the pane), kills all pane processes with
     `t.KillPaneProcesses` (because `RespawnPane -k` only sends
     SIGHUP, which Claude/Node may ignore), and finally
     `t.RespawnPane(paneID, startupCmd)`. The comment at
     `mayor.go:282` notes that `respawn-pane` automatically resets
     `remain-on-exit` to off.

4. Calls the shared `attachToTmuxSession` helper for the final
   attach (smart: links if inside tmux, attaches if outside).

### status

`runMayorStatus` (`mayor.go:336-387`) uses `mgr.CombinedStatus()` to
fetch both tmux and ACP state. With `--running`, prints just the
boolean `status.Active` (for scripting). Otherwise prints a block
showing:

- Tmux session: attached/detached + creation time.
- ACP session: PID, "running (headless)".
- An appropriate "Attach with:" hint depending on which is running.

### ACP (headless) mode

`mayorAcpCmd` (`mayor.go:93-113`) runs the Mayor in headless mode
with stdin/stdout connected, designed for IDE integration via the
Agent Control Protocol. The Long help calls out three environment
overrides (`GT_RIG`, `GT_TOWN_ROOT`, `GT_ROLE`) and one crucial
behavior: "While an ACP session is active, automatic cleanup of
polecat workspaces is vetoed to allow the Mayor to review worker
diffs before they vanish" (`mayor.go:110-111`).

`runMayorAcp` (`mayor.go:461-487`) resolves the town root from
flags / env / CWD, calls `ensureMayorInfra`, resolves the rig name
from `--rig` / `GT_RIG`, and finally `mgr.StartACP(ctx,
agentOverride, rigName)`.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--running` | bool | `false` | `status` | `mayor.go:126` |
| `--agent <alias>` | string | `""` | `start`, `attach`, `restart`, `acp` | `mayor.go:128-130,134` |
| `--rig <name>` | string | `""` | `acp` only | `mayor.go:132` |
| `--town <dir>` | string | `""` | `acp` only | `mayor.go:133` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [deacon.md](deacon.md) — the Deacon is the town-level watchdog
  pair of the Mayor. Same lifecycle command shape (start/stop/
  attach/status/restart/heartbeat).
- [agents.md](agents.md) — Mayor is listed first in `gt agents
  list` / `gt agents menu` (Mayor ranks above all other types via
  the sort in `agents.go:332`).
- [callbacks.md](callbacks.md) — the Mayor's inbox is the target of
  `gt callbacks process`; this command exists to drive
  Mayor-routed messages.
- [boot.md](boot.md) — Boot may nudge/start/wake the Deacon but
  does not directly manage the Mayor.
- [config.md](config.md) — the error path at `mayor.go:446-447`
  references `gt config set dolt.port <N>` as the port-conflict
  remediation.
- [doctor.md](doctor.md) — general diagnostics.
- `gt mail` / `gt nudge` — mailing addresses resolve `"mayor"` to
  this agent per `mayor.go:39` (Sub B scope; cross-link pending).

## Notes / open questions

- **`gt mayor attach` does much more than attach.** It silently
  starts the daemon, starts Dolt, upgrades ACP → tmux, and may
  respawn the pane with rebuilt startup context. For operators
  debugging a stuck Mayor, this is both convenient (one command)
  and surprising (attach can *restart* the process).
- **No `gt mayor nudge` / `mail` / `heartbeat` subcommands** —
  unlike [deacon.md](deacon.md), the Mayor does not expose
  heartbeat or health-check commands at the `gt mayor` prefix. All
  those go via `gt mail` / `gt nudge` / the router. The Mayor has
  no local health-check equivalent to `gt deacon health-check`.
- **Dolt is fatal, daemon is not.** A failing daemon prints a
  warning and continues; a failing Dolt blocks attach entirely
  because "Mayor requires database access" (`mayor.go:407-408`).
  Worth a note on the Dolt operational page.
- **`--agent` appears on four subcommands** via shared variable
  `mayorAgentOverride` (`mayor.go:43`). Because all four set the
  same global, concurrent invocations that race the flag would
  conflict — but since the CLI runs once per process, this is
  fine in practice.
- **Role/concept page pending.** The Mayor as a persona (Chief of
  Staff, escalation target, inter-rig coordinator) is tracked in
  the Long help here but should live in
  `gastown/roles/mayor.md` — not yet created, pending Batch 6.
