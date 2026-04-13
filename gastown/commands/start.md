---
title: gt start
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/start.go
tags: [command, services, lifecycle, boot, crew, mayor, deacon]
---

# gt start

Start Gas Town (Mayor + Deacon, optionally witnesses / refineries /
configured crew) or start a single crew workspace via the
`gt start crew` subcommand. Shares a source file with
[shutdown](./shutdown.md), which documents the opposite direction.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/start.go:58-165`,
`runStart` at `start.go:167-296`.

### Invocation

```
gt start                                       # Start Mayor + Deacon; crew from settings
gt start --all                                 # Also start Witnesses and Refineries for all rigs
gt start --agent <alias>                       # Override Mayor/Deacon agent
gt start --cost-tier standard|economy|budget   # Set ephemeral GT_COST_TIER for this session
gt start <rig>/crew/<name>                     # Shortcut: route to `gt start crew <rig>/<name>`
gt start crew <name> [--rig R] [--account A] [--agent A]
```

The Long help block (`start.go:62-74`) frames this as: "Start Gas
Town by launching the Deacon and Mayor. ... To stop Gas Town, use
'gt shutdown'." Note: [up](./up.md) is the newer, JSON-capable boot
command with similar scope but a different ordering contract and a
Dolt-readiness gate.

### Subcommands

**`crew`** (`start.go:114-131`, run: `start.go:974-1051`)
Convenience command that combines [`gt crew add`](crew.md) and
[`gt crew at --detached`](crew.md). Starts a crew workspace, creating
it if it doesn't exist. Resolves `rig/name` format from the positional arg, or
falls back to `--rig`, then to `inferRigFromCwd`, then to
`inferRigFromCrewName`. Resolves the Claude account via
`config.ResolveAccountConfigDir` using `constants.MayorAccountsPath`.
Ultimately delegates to `crew.Manager.Start` — that manager "handles
workspace creation, settings, and session all in one".

### Behavior of bare `gt start`

**Phase 0 — Crew-path shortcut** (`start.go:168-177`)
If the positional arg contains `/crew/`, it's reinterpreted as
`<rig>/<name>` and routed through `runStartCrew`. This is the
`gt start <rig>/crew/<name>` convenience form.

**Phase 0.5 — Workspace + cost tier** (`start.go:179-192`)
- `workspace.FindFromCwdOrError` is required.
- If `--cost-tier` is set, validates via `config.IsValidTier` and
  exports `GT_COST_TIER` into the process environment so agents spawned
  next inherit it.

**Phase 1 — Ensure daemon patrol config + orphan sweep**
(`start.go:194-207`)
- `config.EnsureDaemonPatrolConfig(townRoot)` seeds defaults.
- `tmux.CleanupOrphanedSessions(session.IsKnownSession)` kills
  zombie Gas Town tmux sessions (tmux alive, Claude dead) before
  starting new agents to avoid name-collision resource accumulation.

**Phase 2 — Discover rigs + start Dolt** (`start.go:213-246`)
- `discoverAllRigs` (`start.go:488-499`) loads `mayor/rigs.json` via
  `config.LoadRigsConfig` and iterates rigs.
- **Dolt must come up before agents** (comment gt-t2zf,
  `start.go:220-223`). If `doltserver.DefaultConfig.DataDir` doesn't
  exist, skip. Otherwise check `doltserver.IsRunning` and call
  `doltserver.Start`. On success, `doltserver.EnsureAllMetadata`
  writes beads metadata so `bd` won't auto-spawn orphan servers.

**Phase 3 — Parallel agent startup** (`start.go:249-282`)
Three goroutines coordinated by one `WaitGroup` and one `sync.Mutex`
for stdout:

1. **`startCoreAgents`** (`start.go:300-365`) — starts Mayor and
   Deacon concurrently. Each is its own nested goroutine. Treats
   `mayor.ErrAlreadyRunning`, `mayor.ErrACPActive`, and
   `deacon.ErrAlreadyRunning` as success conditions.
2. **`startRigAgents`** if `--all` (`start.go:369-395`) — for every
   rig, spawns a witness goroutine and a refinery goroutine (via
   `startWitnessForRig` / `startRefineryForRig`).
3. **`startConfiguredCrew`** (`start.go:422-450`) — reads each rig's
   `settings/config.json`, parses its `crew.startup` field via
   `getCrewToStart` (`start.go:1055-1098`), and launches each crew
   member in a goroutine. `startOrRestartCrewMember`
   (`start.go:456-485`) uses `tmux.IsAgentAlive` (a descendant-
   process check, not just a pane-command check; fixed in #1315 /
   #1330) to distinguish live vs zombie sessions. If zombie, it
   builds a predecessor-discovery beacon via
   `session.FormatStartupBeacon` and sends
   `config.BuildCrewStartupCommand` keys into the existing session to
   restart the agent in place.

**Phase 4 — Summary** (`start.go:288-295`)
On `coreErr == nil`, prints the attach-hint banner pointing at
`gt mayor attach`, `gt deacon attach`, and `gt status`.

### `getCrewToStart` grammar

`start.go:1055-1098`. Parses the rig's `crew.startup` string:
- `"" | "none"` — no crew
- `"all"` — every crew member returned by `crew.Manager.List()`
- Everything else is treated as a comma-separated list with `" and "`
  folded to `","` before splitting. Whitespace is trimmed; empty
  parts are dropped.

This is a simpler grammar than `gt up --restore`'s
`parseCrewStartupPreference` (which understands `pick one`, `any`,
`but not`). See [up](./up.md) for the richer parser.

### Flags

Defined at `start.go:134-142`:
- `--all / -a` — also start witnesses and refineries
- `--agent <alias>` — override agent for Mayor/Deacon
- `--cost-tier <tier>` — ephemeral `GT_COST_TIER`
- `gt start crew --rig <rig>` — explicit rig
- `gt start crew --account <handle>` — Claude account override
- `gt start crew --agent <alias>` — agent override for the crew
  worker

## Notes / open questions

- **Why two boot commands?** `gt start` predates `gt up`. Observable
  differences: `gt up` has a Dolt-readiness gate, JSON output,
  orphaned-bead recovery, richer crew-startup grammar, and a worker
  pool for witness/refinery startup. `gt start` has the `/crew/`
  shortcut, an inline cost-tier setter, and calls
  `config.EnsureDaemonPatrolConfig` (vs `up`'s
  `daemon.EnsureLifecycleConfigFile`). These look like divergent
  epochs of the same intent — flag for the drift folder when one
  gets retired.
- **`start` does not launch the daemon.** Unlike `gt up`, nothing in
  `runStart` calls `ensureDaemon` or spawns `gt daemon run`. Agents
  started here rely on the daemon being started separately (e.g., by
  `gt daemon start` or `gt up`). If the daemon is down, the Mayor
  has no scheduler nudging it.
- **`config.EnsureDaemonPatrolConfig` vs
  `daemon.EnsureLifecycleConfigFile`** — two different config-seeding
  entry points. Which file each writes and whether they collide is
  worth checking.
- **`coreErr`** only surfaces *the first* Mayor/Deacon error
  (`start.go:285`). Witness/refinery/crew failures are logged but
  don't fail the command.

## Related

- [shutdown](./shutdown.md) — opposite direction; same source file
- [up](./up.md) — newer boot command with overlapping scope
- [down](./down.md) — reversible pause; paired with `up`
- [daemon](./daemon.md) — not started here; a gap compared to `up`
- [mayor](./mayor.md), [deacon](./deacon.md) — the two core agents
  this command exists to start
- [witness](./witness.md), [refinery](./refinery.md) — started per
  rig when `--all` is passed
