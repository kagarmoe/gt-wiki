---
title: gt status
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/status.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, rigs, agents, dolt, tmux, polecat-safe, watch-mode]
---

# gt status

Overall workspace status report: town name, overseer, daemon, Dolt
server, tmux server, ACP session, global agents (Mayor, Deacon), and
a per-rig breakdown with agents, hooks, and merge-queue summary.
Supports JSON, fast mode, and continuous watch mode.

**Also known as:** `gt stat` (alias declared on `status.go:42`).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** **yes** — annotated
`AnnotationPolecatSafe: "true"` at
`/home/kimberly/repos/gastown/internal/cmd/status.go:44`.
**Beads-exempt:** **yes** — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:60`; status must
work even with `bd` broken because it's the operator's primary
diagnostic entry point.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/status.go:40-948+`
(file is 1904 lines; the rest is table rendering, not behavior).

### Invocation

```
gt status [--json] [--fast] [-w|--watch] [-n INT] [-v|--verbose]
gt stat ...
```

### Behavior

`runStatus` (`status.go:457-462`) dispatches:

- `--watch` → `runStatusWatch` (continuous refresh)
- otherwise → `runStatusOnce`

### Once mode — `runStatusOnce`

Source: `status.go:596-605`.

Calls `gatherStatus()`, then either `outputStatusJSON` or
`outputStatusText(os.Stdout, ...)`.

### Gather — `gatherStatus`

Source: `status.go:607-948`. This is the work unit that the watch
loop re-runs on each tick.

1. **Resolve town root** via `workspace.FindFromCwdOrError`
   (`status.go:609-612`).
2. **Load town config**: `config.LoadTownConfig(MayorTownPath)` with
   a fallback to `filepath.Base(townRoot)` on error
   (`status.go:615-620`).
3. **Load rigs config**: `config.LoadRigsConfig(MayorRigsPath)`
   with an empty fallback (`status.go:623-628`).
4. **Load town settings**: `config.LoadOrCreateTownSettings` for
   agent alias/runtime display info (`status.go:631`).
5. **Pre-fetch tmux sessions** (`status.go:645-664`): calls
   `t.ListSessions()` and fans out an `IsAgentAlive` check per
   session in parallel via a `sync.WaitGroup`. The map
   `allSessions[name] = alive` is used throughout so "running"
   means *the agent process is alive inside the tmux session*, not
   just that tmux has the session. Comment at `status.go:643-644`
   references `gt-bd6i3` which fixed zombie sessions showing as
   running.
6. **Discover rigs** via `rig.Manager.DiscoverRigs`
   (`status.go:667-670`).
7. **Pre-fetch agent beads** in parallel across all rigs plus town
   (`status.go:672-761`):
   - Town beads client fetches `ListAgentBeads` from
     `beads.GetTownBeadsPath(townRoot)` (Mayor, Deacon).
   - Per-rig goroutines fetch from `<rig>/mayor/rig`.
   - For each agent bead, looks up the `HookBead` field (or falls
     back to `beads.ParseAgentFields` on the description), then
     `ShowMultiple`s all referenced hook beads into the shared
     `allHookBeads` map, protected by `beadsMu`.
8. **Load overseer** via `config.LoadOrDetectOverseer(townRoot)`
   (`status.go:767-782`). Unless `--fast`, also fetches the
   overseer mailbox unread count via `mail.Router.GetMailbox`.
9. **Daemon status** via `daemon.IsRunning(townRoot)`
   (`status.go:794-796`).
10. **Dolt status** (`status.go:799-827`): reads `DefaultConfig`
    and either reports `Remote: true` (no local server) or probes
    `doltserver.IsRunning` and `LoadState` for the actual port
    (which may differ from the config port). On not-running it
    also checks `doltserver.CheckPortConflict` to see if another
    town is holding the port.
11. **Tmux status** (`status.go:830-846`): resolves the default
    socket via `tmux.GetDefaultSocket`, computes the full socket
    path at `/tmp/tmux-<UID>/<socket>`, and stats it for running
    state. `SessionCount = len(allSessions)`.
12. **ACP status** via `mayor.IsACPActive` +
    `mayor.GetACPPid` (`status.go:849-852`).
13. **Parallel agent discovery** (`status.go:854-913`):
    - Global agents via `discoverGlobalAgents` (mayor, deacon, etc.).
    - Per-rig via `discoverRigHooks` (skipped in `--fast`) and
      `discoverRigAgents`. Crew workers counted via
      `crew.NewManager.List`.
    - Merge-queue summary via `getMQSummary` (skipped in `--fast`).
14. **Runtime enrichment** (`status.go:917-931`): each agent's
    `AgentAlias` and `AgentInfo` are filled by `resolveAgentDisplay`,
    which inspects the actual process tree via `detectRuntimeFromSession`
    / `findAgentCmdline` / `parseRuntimeInfo` to read
    `/proc/<pid>/cmdline` and extract the real runtime and model
    (`status.go:232-402`). Walks pane → shell children →
    grand-children to handle wrapper processes (`node /path/to/pi`,
    `cgroup-wrap → pi`). Falls back to config-based display when
    not running. Special-case for `pi`: reads
    `~/.pi/agent/settings.json` for the default provider/model.
15. **Aggregate summary counts** (`status.go:933-947`): rig count,
    polecat/crew counts, witness/refinery totals, active hooks.

### Watch mode — `runStatusWatch`

Source: `status.go:464-594`.

- Rejects `--json --watch` combination (`status.go:465-467`).
- Installs a SIGINT/SIGTERM handler (`status.go:472-474`).
- Tickers at `statusInterval` seconds (default 2).
- Renders to a `bytes.Buffer` before flushing to stdout atomically,
  so the screen never shows a half-drawn frame between
  `\033[H\033[2J` clear and content (`status.go:562-564`).
- **Degradation recovery** (`status.go:485-534`): keeps a cached
  previous `TownStatus` for up to `5 * interval` seconds. If
  `gatherStatus` returns zero running agents when the previous
  iteration had some, retries once; if still zero, uses the cached
  snapshot and annotates the frame with "(using cached data from
  HH:MM:SS)". This prevents transient tmux failures from flashing
  empty bubbles across every agent.
- Exits cleanly on signal with a "Stopped." line when TTY.

### Data shapes

Defined at `status.go:64-180`. Top-level:

```go
type TownStatus struct {
    Name     string
    Location string
    Overseer *OverseerInfo
    DND      *DNDInfo
    Daemon   *ServiceInfo
    Dolt     *DoltInfo
    Tmux     *TmuxInfo
    ACP      *ServiceInfo
    Agents   []AgentRuntime   // global: Mayor, Deacon
    Rigs     []RigStatus
    Summary  StatusSum
}
```

Per-agent runtime includes `Session`, `Running`, `ACP`,
`HasWork`, `WorkTitle`, `HookBead`, `State`, `NotificationLevel`,
`UnreadMail`, `FirstSubject`, `AgentAlias`, and `AgentInfo` (the
detected runtime/model string).

Per-rig `MQSummary` has `Pending`, `InFlight`, `Blocked`, `State`
(`idle`/`processing`/`blocked`), `Health` (`healthy`/`stale`/
`empty`).

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`status.go:56-61`):

| Flag                  | Type | Default | Purpose                                             |
|-----------------------|------|---------|-----------------------------------------------------|
| `--json`              | bool | `false` | Output as JSON                                      |
| `--fast`              | bool | `false` | Skip mail + hook + MQ lookups for speed             |
| `--watch` / `-w`      | bool | `false` | Continuous refresh                                  |
| `--interval` / `-n`   | int  | `2`     | Watch refresh interval in seconds                   |
| `--verbose` / `-v`    | bool | `false` | Multi-line per agent                                |

### `--fast` mode

Skips in the interest of latency:

- Overseer mail count (`status.go:776-781`)
- Agent mail counts (inside `discoverGlobalAgents`/`discoverRigAgents`)
- Per-rig hook discovery (`status.go:891-893`)
- Per-rig merge-queue summary (`status.go:907-909`)

Hook info in `--fast` is populated from pre-fetched agent beads
instead of the expensive handoff bead lookup path.

## Related

- [doctor](doctor.md) — doctor runs deep health checks; status is
  the fast live snapshot.
- [vitals](vitals.md) — vitals is the Dolt-focused health dashboard;
  status reports one-line Dolt state as part of the full picture.
- [heartbeat](heartbeat.md) — heartbeat is a single-agent state
  reporter that status aggregates across all agents.
- [feed](feed.md) — feed is event-stream-based, status is
  snapshot-based; complementary.
- [dashboard](dashboard.md) — dashboard is the web-UI equivalent of
  status / feed combined.
- [activity](activity.md) — activity and status consume some of the
  same process/session data from different angles.
- [info](info.md) — info reports static environment; status reports
  live runtime.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  status's polecat-safe annotation and beads exemption.
- [README.md](README.md) — command tree index.

## Notes / open questions

- `discoverGlobalAgents`, `discoverRigHooks`, `discoverRigAgents`,
  `getMQSummary`, `detectCurrentDNDStatus`, and
  `addEstopToStatus` all live elsewhere in the cmd package.
  Documenting them would round out the status data-flow picture.
- `parseRuntimeInfo` treats `node` / `bun` / `npx` / `bunx` as
  runtime wrappers (`status.go:340-345`). Any other wrapper
  (`deno`, `cgroup-wrap` without the child inspected separately,
  custom launchers) would show the wrapper command instead of the
  real agent.
- Watch mode's 5×interval stale-cache window is a heuristic; the
  comment at `status.go:485-488` explains the rationale but there's
  no documentation of what tmux failures it was designed to mask.
- `SessionCount` reported for tmux is `len(allSessions)`, which
  includes both known Gas Town sessions and anything else tmux is
  running on the same socket — not a pure Gas Town session count.
- The `--fast` mode docs on the cobra `Long` (`status.go:46-51`)
  only mention skipping mail lookups, but the code additionally
  skips MQ and hook lookups. The help text understates the tradeoff.
