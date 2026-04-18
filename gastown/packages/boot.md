---
title: internal/boot
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/boot/boot.go
tags: [package, boot, watchdog, daemon, deacon, triage]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/boot

Boot watchdog package -- the daemon's entry point for Deacon triage
decisions. Boot is a "dog" (short-lived agent) that runs fresh on each
daemon tick, deciding whether to wake, nudge, or interrupt the Deacon.
This package manages the Boot lifecycle: session spawning, flock-based
mutual exclusion, and status persistence.

**Go package path:** `github.com/steveyegge/gastown/internal/boot`
**File count:** 1 non-test Go file (boot.go, 237 lines).
**Imports (notable):** [`internal/cli`](cli.md) (`cli.Name()` for
binary name), [`internal/config`](config.md) (`AgentEnv`,
`EnvForExecCommand`), [`internal/session`](session.md)
(`BootSessionName()`, `StartSession`), [`internal/tmux`](tmux.md)
(`NewTmux`, session lifecycle), [`internal/util`](util.md)
(`SetDetachedProcessGroup`), `github.com/gofrs/flock` (file-based
mutual exclusion).
**Imported by:** `internal/daemon/daemon.go` (spawns Boot on each daemon
tick), `internal/doctor/boot_check.go` (health check),
`internal/cmd/boot.go` (CLI wiring).

## What it actually does

### Boot struct (boot.go:38-45)

The `Boot` struct holds:
- `townRoot` -- the Gas Town root path.
- `bootDir` -- working directory at `<townRoot>/deacon/dogs/boot/`
  (`boot.go:51`).
- `deaconDir` -- the Deacon's directory at `<townRoot>/deacon/`
  (`boot.go:52`).
- `tmux` -- a `*tmux.Tmux` instance for session management.
- `degraded` -- boolean, true when `GT_DEGRADED=true` env var is set
  (`boot.go:54`).
- `lockHandle` -- a `*flock.Flock` held during triage execution.

### Status tracking (boot.go:28-35, 121-150)

`Status` struct records each Boot execution: `StartedAt`,
`CompletedAt`, `LastAction` (one of `start`/`wake`/`nudge`/`nothing`),
`Target` (e.g. `deacon`, `witness`), and `Error`. Persisted as
`.boot-status.json` in the boot directory. `SaveStatus` writes via
`os.WriteFile` (not atomic -- operational data, not critical state).
`LoadStatus` returns an empty `Status` if the file does not exist.

### Mutual exclusion (boot.go:89-118)

Boot uses `flock` (not tmux session existence) for mutual exclusion
because triage runs _inside_ the Boot session -- checking session
existence would always succeed.

- `AcquireLock()` (`boot.go:89-105`) -- `TryLock()` on the
  `.boot-running` marker file. Returns error if another triage holds
  the lock.
- `ReleaseLock()` (`boot.go:108-118`) -- unlocks and removes the
  marker file.

### Spawning (boot.go:157-217)

- `Spawn(agentOverride string)` (`boot.go:157-167`) -- entry point.
  Boot is ephemeral by design: no `IsRunning()` guard, each spawn kills
  any existing session. Routes to `spawnTmux` or `spawnDegraded` based
  on the `degraded` flag.

- `spawnTmux(agentOverride string)` (`boot.go:170-196`) -- kills any
  stale Boot session, then uses `session.StartSession` with a
  `SessionConfig` specifying role `"boot"`, working directory
  `bootDir`, a beacon config (recipient: `boot`, sender: `daemon`,
  topic: `triage`), and the instruction `"Run gt boot triage now."`
  The `agentOverride` parameter allows specifying an alternative agent
  alias.

- `spawnDegraded()` (`boot.go:200-217`) -- no-tmux fallback. Runs
  `gt boot triage --degraded` as a detached subprocess with centralized
  `config.AgentEnv` environment plus `GT_DEGRADED=true`. Async: starts
  the process and returns immediately.

### Liveness checking (boot.go:74-83)

- `IsRunning()` / `IsSessionAlive()` -- delegates to
  `tmux.HasSession(session.BootSessionName())` for observable-reality
  liveness (the code comments cite the "ZFC principle").

### Accessors (boot.go:219-237)

`IsDegraded()`, `Dir()`, `DeaconDir()`, `Tmux()` -- simple getters.

## Related

- [gt boot](../commands/boot.md) -- the CLI command that wraps this
  package (command page, distinct entity).
- [internal/daemon](daemon.md) -- the daemon heartbeat loop that
  spawns Boot on each tick.
- [internal/session](session.md) -- `StartSession` used by
  `spawnTmux`, `BootSessionName()` for session naming.
- [internal/tmux](tmux.md) -- tmux wrapper used for session lifecycle.
- [internal/config](config.md) -- `AgentEnv` for environment setup.
- [internal/doctor](doctor.md) -- `boot_check.go` health check.
- [internal/deacon](deacon.md) -- the agent role that Boot triages.
- [go-packages inventory](../inventory/go-packages.md)

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `gt` | `boot triage` | `--degraded` | hardcoded | `boot.go:203` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| boot status file | JSON data | Boot status persistence | `boot.go:131` |

## Notes / open questions

- Boot is explicitly ephemeral ("each spawn kills any existing session
  and starts fresh") -- this is a deliberate design choice documented
  in the code at `boot.go:158-159`. Unlike persistent agents (crew,
  witness), Boot does not accumulate context between ticks.
- The `SaveStatus` method uses `os.WriteFile` rather than
  `util.AtomicWriteJSON`. The comment at `boot.go:131` notes this is
  intentional: boot status is "non-sensitive operational data" where
  the atomic-write overhead is not warranted.
- The `GT_DEGRADED` env var triggers a tmux-free execution path. This
  is the fallback for environments where tmux is unavailable (e.g.,
  containers, CI).
- The `agentOverride` parameter on `Spawn` allows the daemon to
  select a different agent alias for the Boot session, enabling
  per-town agent configuration.
