---
title: gt daemon
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/daemon.go
  - /home/kimberly/repos/gastown/internal/cmd/daemon_reload_unix.go
  - /home/kimberly/repos/gastown/internal/cmd/daemon_reload_windows.go
tags: [command, services, daemon, lifecycle, supervisor, logs]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt daemon

Manages the Gas Town background daemon — a "dumb scheduler" Go process
that pokes agents on a heartbeat, processes lifecycle requests, and
restarts sessions when agents request cycling.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/daemon.go:21-167`.

### Invocation

```
gt daemon                          # parent — RunE: requireSubcommand
gt daemon start
gt daemon stop
gt daemon status
gt daemon logs [-n N] [-f]
gt daemon run                      # hidden — internal/supervisor entry
gt daemon enable-supervisor
gt daemon clear-backoff <agent>
gt daemon rotate-logs [--force]
```

The bare `gt daemon` is a parent command with no Run; it dispatches to
`requireSubcommand` which prints the help screen and errors out
(`daemon.go:25`).

### Subcommands

**`start`** (`daemon.go:36-43`, run: `daemon.go:169-235`)
Locates the town root, checks `daemon.IsRunning`, then re-execs the
current binary as `gt daemon run` with a detached process group via
`util.SetDetachedProcessGroup`. Polls `daemon.IsRunning` every 100ms for
up to 3 seconds (30 iterations) waiting for the new process to acquire
the daemon lock. If polling times out, reads
`<townRoot>/daemon/daemon.log` for a `Daemon startup failed (PID N): ...`
line via `readDaemonStartupFailure` (`daemon.go:320-336`) and surfaces
that as the error. Handles a race where another concurrent start won the
lock — reports the winning PID instead of erroring.

**`stop`** (`daemon.go:45-56`, run: `daemon.go:237-257`)
Calls `daemon.IsRunning`, then `daemon.StopDaemon(townRoot)`. Errors if
the daemon is not running.

**`status`** (`daemon.go:58-69`, run: `daemon.go:259-305`)
Reports running/PID/town. If running and `daemon.LoadState` succeeds,
also prints start time, last heartbeat (with heartbeat counter), and the
binary's mtime via `getBinaryModTime` (`daemon.go:307-318`). Warns
`Binary is newer than process — consider 'gt daemon stop && gt daemon
start'` when the on-disk binary is newer than the running daemon's start
time.

**`logs`** (`daemon.go:71-84`, run: `daemon.go:338-363`)
Tails `<townRoot>/daemon/daemon.log` via `tail -n N` (default 50) or
`tail -f` if `--follow`. Shells out — does not implement its own
tailing.

**`run`** (`daemon.go:86-96`, run: `daemon.go:365-388`)
Hidden subcommand. **This is the actual daemon process.** Clears all
identity env vars from `agentconfig.IdentityEnvVars` (so the daemon
doesn't impersonate the agent that launched it via mail/bd — GH#3006),
sets `BD_ACTOR=daemon`, then calls `daemon.New(daemon.DefaultConfig)`
and `d.Run()`. Invoked by `gt daemon start`'s child re-exec and by
launchd/systemd supervisor units.

**`enable-supervisor`** (`daemon.go:98-110`, run: `daemon.go:390-413`)
Calls `templates.ProvisionSupervisor(townRoot)` which writes a launchd
plist on macOS or a user systemd unit on Linux. Prints OS-specific
unload/disable instructions. Branch on `runtime.GOOS == "darwin"`.

**`clear-backoff <agent>`** (`daemon.go:130-145`, run:
`daemon.go:415-448`)
Clears the on-disk crash-loop backoff state for `<agent>` via
`daemon.ClearAgentBackoff`, then if the daemon is running sends it a
`signalDaemonReload` (defined elsewhere — likely SIGHUP) so the
in-memory restart tracker also resets. If not running, the cleared state
takes effect on next start.

**`rotate-logs`** (`daemon.go:112-126`, run: `daemon.go:450-478`)
Rotates daemon-managed log files via `daemon.RotateLogs` (or
`ForceRotateLogs` with `--force`). Uses copytruncate for Dolt server
logs. Skips `daemon.log` itself — it uses lumberjack auto-rotation.
Default threshold is 100MB; `--force` ignores threshold. Reports rotated
/ skipped / errors per file.

### Flags

Subcommand-scoped (`daemon.go:152-165`):
- `daemon logs --lines / -n` (int, default 50)
- `daemon logs --follow / -f` (bool, default false)
- `daemon rotate-logs --force` (bool, default false)

### Notable invariants

- **Lock-based singleton**: only one `daemon run` can hold the lock for
  a given town root. The polling loop in `start` is the discovery
  mechanism.
- **Identity scrub** (`daemon.go:371-379`): the daemon explicitly
  unsets `agentconfig.IdentityEnvVars` so subprocess `gt mail send` and
  `bd` calls aren't misattributed to the launching agent.
- **Supervisor delegation**: `enable-supervisor` does not start the
  daemon directly — it provisions launchd/systemd, which then invokes
  `gt daemon run` according to its own schedule (login/boot).

## Related

- [internal/daemon package](../packages/daemon.md) — the Go implementation this command front-ends; main heartbeat loop, state-file layout, lock discipline, restart tracker
- [boot](./boot.md) — deacon's watchdog, separate from this Go daemon
- [deacon](./deacon.md) — health orchestrator agent
- [doctor](./doctor.md) — diagnostic command that inspects daemon state
- [dashboard](./dashboard.md) — surfaces daemon heartbeat counter

## Failure modes

No failure modes discovered. All subcommands are thin wrappers: `start` spawns a detached process with proper race detection (`daemon.go:224-231`), `stop` delegates to `daemon.StopDaemon`, `status` is read-only. The `run` subcommand (foreground mode) clears identity env vars to prevent misattribution. Error paths are well-checked.

## Notes / open questions

- **`signalDaemonReload`** is referenced from `clear-backoff`
  (`daemon.go:438`) and defined in `daemon_reload_unix.go:10-11` —
  sends **SIGUSR2** (not SIGHUP as originally speculated) to trigger
  a reload. Windows variant in `daemon_reload_windows.go`.
- **`templates.ProvisionSupervisor` mechanics**: where do the plist /
  systemd unit templates live? Worth grounding under `gastown/files/`.
- **`daemon.IdentityEnvVars`**: what's the full list? The comment cites
  `GT_ROLE`/`GT_CREW` but the list is package-defined.
- **Crash-loop backoff thresholds**: how many crashes within what window
  push an agent into backoff? Lives in `internal/daemon`.
- **Lumberjack config for `daemon.log`**: max size, max age, max
  backups — encoded in `internal/daemon` config defaults.
- **Hidden `run` is callable directly**: a user with `gt daemon run`
  bypasses the `start` lock-check guards. Documented as
  internal/supervisor-only but nothing prevents direct invocation.
