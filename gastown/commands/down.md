---
title: gt down
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/down.go
tags: [command, services, lifecycle, shutdown, tmux]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
---

# gt down

Stops Gas Town services in a structured shutdown sequence — refineries,
witnesses, town sessions, daemon, Dolt server, and (optionally)
polecats and crew. Reversible via `gt up`. Acquires a per-town
shutdown lock so concurrent `gt down` invocations cannot race each
other.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/down.go:45-471`.

### Invocation

```
gt down                            # Stop infrastructure (default)
gt down --polecats                 # Also stop polecat sessions
gt down --all                      # Full shutdown with orphan cleanup
gt down --nuke                     # Also kill the per-town tmux server
gt down --dry-run                  # Preview without acting
gt down --force                    # Skip graceful Ctrl-C and kill directly
gt down --quiet                    # Only print errors
```

Documented as a *pause*: `gt up` brings everything back. For permanent
cleanup (removing worktrees), use `gt shutdown`. The Long help block
(`down.go:49-73`) lists exactly which agents are stopped: crew,
refineries, witnesses, mayor, boot, deacon, daemon, dolt.

### Phase sequence

The shutdown is intentionally ordered. Each phase is gated on
`downDryRun`, and `allOK` collects whether the phase succeeded so the
final summary can fail the command if anything went wrong.

**Phase 0 — Acquire shutdown lock + write sentinel**
(`down.go:107-133`)
- `acquireShutdownLock` (`down.go:678-700`) takes an exclusive
  `flock` on `<townRoot>/daemon/shutdown.lock` with a 5-second timeout.
  The lock file is **never removed** because flock works on file
  descriptors, not paths — removing it while another waiter is queued
  would cause a phantom-lock split.
- Writes a `daemon/shutting-down` sentinel containing the current PID
  (GH#2656). `ensureDaemon` checks this file so agents don't restart
  the daemon mid-shutdown. Removed via `defer` on the way out.
- Calls `t.SetExitEmpty(false)` so the per-town tmux server stays
  alive even after all sessions are gone. This is what makes
  subsequent `gt up` cheap.

**Phase 0.5 — `--polecats` (optional)** (`down.go:144-165`)
Calls `stopAllPolecats` (`down.go:476-554`). Discovers polecats per
rig via `polecat.NewSessionManager(t, r).ListPolecats()` and stops
them in parallel using a goroutine fan-out. `--force` propagates to
`mgr.Stop`.

**Phase 0.6 — Stop crew sessions** (`down.go:167-178`,
`stopAllCrew` at `down.go:559-640`)
Crew is *always* stopped because crew sessions consume tokens.
Discovers via `crew.NewManager(r, g).List()`, then stops each via
`stopSession` in parallel.

**Phase 1 — Stop refineries** (`down.go:181-198`)
Per rig: `session.RefinerySessionName(session.PrefixFor(rigName))`,
stop via `stopSession`.

**Phase 2 — Stop witnesses** (`down.go:201-218`)
Per rig: `session.WitnessSessionName(prefix)`, stop via `stopSession`.

**Phase 3 — Stop town-level sessions** (`down.go:221-237`)
Iterates `session.TownSessions()` (mayor, boot, deacon) and calls
`session.StopTownSession(t, ts, downForce)`.

**Phase 4 — Stop daemon** (`down.go:240-261`)
`daemon.IsRunning` then `daemon.StopDaemon`.

**Phase 4b-i — Stop bd dolt idle-monitors** (`down.go:264-277`)
`findIdleMonitorProcesses(townRoot)` then `stopIdleMonitors`. **Must
run before Dolt shutdown** because these background processes respawn
per-agent Dolt servers, which would race the canonical server's
restart and grab the port first.

**Phase 4b-ii — Stop Dolt server** (`down.go:280-302`)
[`doltserver`](../packages/doltserver.md)`.IsRunning` then `doltserver.Stop`.

**Phase 4b-iii — Stop imposter / orphan Dolt servers**
(`down.go:304-330`)
Calls `doltserver.KillImposters` (port-conflict-based) *and*
`findOrphanDoltServers` (any `dolt sql-server` rooted in this town's
directory tree, even on a different port). Both run only outside
dry-run.

**Phase 4b-iv — Remove `.beads/dolt` directories**
(`down.go:333-346`)
Legacy per-agent data dirs that bd uses to auto-spawn local Dolt
servers. Removing them prevents rogue respawn on the next `gt up`.
Data is assumed to have been migrated via `gt dolt migrate`.

**Phase 4c — Clean up legacy socket sessions**
(`down.go:348-371`, helpers at `down.go:826-899`)
- `cleanupLegacyDefaultSocket`: kills any Gas Town session left on the
  bare "default" tmux socket by old binaries.
- `cleanupLegacyBaseSocket`: kills sessions on the old basename-only
  socket (e.g., `gt`) — pre-dating path-hashed socket names like
  `gt-a1b2c3`.

**Phase 5 — Orphan cleanup + verification (`--all` / `--force`)**
(`down.go:374-405`)
- `session.KillTrackedPIDs(townRoot)` kills processes recorded in PID
  files left behind by sessions.
- `cleanupOrphanedClaude(defaultDownOrphanGraceSecs)` — 5-second grace
  period (`down.go:42`).
- After 500ms, `verifyShutdown` (`down.go:704-743`) re-checks: any
  known tmux sessions still alive, the daemon PID file's process
  still running, orphan Claude/node processes referencing the town
  root (`findOrphanedClaudeProcesses`, `down.go:752-797`), idle-
  monitors respawned, orphan Dolt servers. Anything found is reported
  as `respawned` and flips `allOK` to false.

**Phase 6 — `--nuke`** (`down.go:412-436`)
Kills the per-town tmux server's socket (derived from a hash of the
town's canonical path, see `registry.go townSocketName`). Requires
`GT_NUKE_ACKNOWLEDGED=1` because users may have opened custom tmux
windows on the same socket. Without the env var, the command refuses
and prints the unlock incantation.

**Summary phase** (`down.go:438-468`)
On success, emits `events.LogFeed(events.TypeHalt, "gt", ...)` listing
the services that were stopped (`dolt`, `daemon`, `deacon`, `boot`,
`mayor`, `<rig>/refinery`, `<rig>/witness`, conditionally `crew`,
`polecats`, `bd-processes`, `tmux-server`).

### Flags

Defined at `down.go:86-94`:
- `--quiet / -q`
- `--force / -f`
- `--polecats / -p`
- `--all / -a`
- `--nuke` (no short alias; gated on `GT_NUKE_ACKNOWLEDGED=1`)
- `--dry-run`

### Helpers

- **`stopSession`** (`down.go:655-674`) — try Ctrl-C first (graceful),
  wait `constants.GracefulShutdownTimeout`, then `KillSessionWithProcesses`.
  `--force` skips the Ctrl-C step.
- **`printDownStatus`** (`down.go:642-651`) — central status printer.
  Honors `--quiet` for the success path.

## Related

- [daemon](./daemon.md) — Phase 4 stops the Go daemon process
- [internal/daemon package](../packages/daemon.md) — the `shutdown.lock`/`shutting-down` sentinel + `daemon.IsRunning`/`daemon.StopDaemon` live here; flock-on-inode rationale at `daemon.go:1972-1976`
- [dolt](./dolt.md) — Phases 4b-i through 4b-iv handle Dolt teardown
- [estop](./estop.md) — emergency freeze (different shape; uses
  SIGTSTP and preserves context, does not stop services)
- [mayor](./mayor.md), [deacon](./deacon.md), [boot](./boot.md) —
  Phase 3 town-level sessions
- [refinery](./refinery.md), [witness](./witness.md) — Phases 1-2
  per-rig sessions
- [polecat](./polecat.md), [crew](./crew.md) (if present) — Phase 0.5
  / 0.6 targets
- [doctor](./doctor.md) — diagnostics for what `gt down` cleans up

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/down.go:49-77` — Cobra `Long` text

### Verbatim
> This is a "pause" operation - use 'gt start' to bring everything back up.
> For permanent cleanup (removing worktrees), use 'gt shutdown' instead.

## Drift

### `downCmd.Long` recommends `gt start` as complement; `gt up` is the actual complement
- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/down.go:66`
- **Docs claim:** "use 'gt start' to bring everything back up"
- **Code does:** `gt start` (`start.go:167-296`) does NOT start the daemon — it starts Mayor + Deacon and optionally witnesses/refineries/crew, but has no `ensureDaemon` call. `gt up` (`up.go:120-448`) is the actual complement: it starts Dolt, daemon, deacon, mayor, witnesses, and refineries with a Dolt-readiness gate and orphan recovery. The wiki page for [start](./start.md) explicitly notes: "start does not launch the daemon. Unlike gt up, nothing in runStart calls ensureDaemon." The wiki page for this command (`down.md`) itself says "Reversible via gt up" in its header text — the Long text contradicts the wiki and the code's intended pairing.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code`
- **Release position:** `in-release`

See also: [gastown/drift/README.md](../drift/README.md)

## Failure modes

### Partial completion

- **Shutdown sentinel may persist on panic:** `down.go:124` writes sentinel, defers `os.Remove`. On abnormal termination, sentinel persists and `ensureDaemon` refuses to start. **Absent** — no cleanup for abnormal exit.

### Silent suppression

- **`exit-empty` configuration:** `down.go:131` uses `_ = t.SetExitEmpty(false)`. If this fails, tmux server may exit when all sessions die. **Absent** — no warning.
- **Halt event fire-and-forget:** `down.go:464` uses `_ = events.LogFeed(events.TypeHalt, ...)`. **Absent** — audit gap.

## Notes / open questions

- **Lock file is never removed** — comment at `down.go:114-120`
  explains the flock semantics. Worth a small concept page on
  flock-on-deleted-inode if it bites again.
- **`GT_NUKE_ACKNOWLEDGED` is the only "acknowledged" gate** in the
  command — pattern worth noting. Compare to other destructive
  commands.
- **`findOrphanedClaudeProcesses`** parses `ps -eo pid,comm,args` and
  filters by command name (`claude`, `claude-code`, `codex`, `node`)
  *and* by whether `args` contains the town root path
  (`down.go:752-797`). The earlier `pgrep -l node` implementation was
  too broad.
- **`gt up` complement** is referenced throughout but not yet mapped
  in the wiki. When it lands, cross-link it bidirectionally with
  Phase 0's `SetExitEmpty(false)` rationale.
- **`GH#3006` and `GH#2656`** are explicit issue refs in the
  comments — useful provenance hooks if those issues survive in
  beads/upstream.
