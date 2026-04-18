---
title: gt shutdown
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/start.go
tags: [command, services, lifecycle, shutdown, polecat-cleanup]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt shutdown

Permanent "done for the day" shutdown: stops all Gas Town sessions
*and* removes polecat worktrees and branches. Distinct from
[down](./down.md), which is a reversible pause. Shares a source file
with [start](./start.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/start.go:79-112`,
`runShutdown` at `start.go:501-558`.

### Invocation

```
gt shutdown                          # Default: stop infrastructure + polecats + cleanup
gt shutdown --all / -a               # Also stop crew sessions
gt shutdown --polecats-only          # Only stop polecats (leave infra running)
gt shutdown --force / -f             # Skip confirmation (alias for --yes)
gt shutdown --yes / -y               # Skip confirmation
gt shutdown --graceful / -g          # Send ESC + shutdown message; wait for handoff
gt shutdown --wait <seconds> / -w    # Wait budget for graceful shutdown (default 30)
gt shutdown --nuclear                # Force polecat cleanup despite uncommitted work
gt shutdown --cleanup-orphans        # Use longer grace period for orphan-process cleanup
gt shutdown --cleanup-orphans-grace-secs <n>  # Grace period in seconds (default 60)
```

The Long help block (`start.go:83-110`) emphasizes the comparison:
"gt down - Pause (stop processes, keep worktrees) - reversible / gt
shutdown - Done (stop + cleanup worktrees) - permanent cleanup".
Polecats with uncommitted work are **skipped (protected)** by default;
`--nuclear` bypasses the check.

### Behavior

**Phase 0 — Listing + categorization** (`start.go:501-558`)
- `tmux.ListSessions` enumerates every session.
- `categorizeSessions` (`start.go:561-601`) splits Gas Town sessions
  (matched by `session.IsKnownSession`) into `toStop` / `preserved`.
  Role classification uses `session.ParseSessionName`:
  - `--polecats-only`: only `RolePolecat` stop, everything else is
    preserved.
  - `--all`: everything goes in `toStop`.
  - Default: `RoleCrew` is preserved; everything else stops.
- If nothing matches, still calls `stopDaemonIfRunning` as a safety
  sweep for orphaned daemons.
- Confirmation prompt (stdin `y`/`yes`) unless `--yes` or `--force`.
- Dispatches to `runGracefulShutdown` or `runImmediateShutdown`.

### Graceful path (`start.go:603-674`)

Phased with 8 explicit phases:

1. **Send ESC** to every agent session via `tmux.SendKeysRaw(sess,
   "Escape")` to interrupt whatever they're doing.
2. **Send the shutdown message** (with a
   `constants.ShutdownNotifyDelay` pause between) asking each agent
   to save state, update their handoff bead, and `/exit`.
3. **Wait** `shutdownWait` seconds with a countdown every 5s.
4. **Kill sessions in order** — `killSessionsInOrder`.
5. **Cleanup orphaned Claude processes** via `cleanupOrphanedClaude`
   with a grace period of `defaultOrphanGraceSecs = 5` unless
   `--cleanup-orphans` bumps it to `shutdownCleanupOrphansGrace`
   (default 60).
6. **Cleanup polecats** — `cleanupPolecats` (see below).
7. **Stop the daemon** — `stopDaemonIfRunning`.
8. **Verify no orphans survived** — `verifyNoOrphans`.

### Immediate path (`start.go:676-718`)

Same steps 4-8 as the graceful path but without the ESC/message/wait
preamble. Both paths converge on the same kill-order and cleanup
routines.

### Kill order — `killSessionsInOrder`

`start.go:720-827`. Comment (`start.go:720-726`) lays out the
invariant: "matching gt down":

1. **Polecats and crew** (workers — stop before monitors restart them)
2. **Refineries** (work processors)
3. **Witnesses** (monitors — stop before the deacon so they can't
   restart workers)
4. **Town sessions: Mayor → Boot → Deacon** — Boot monitors Deacon,
   so it must die before Deacon does.

`killAndVerify` checks session existence, calls
`tmux.KillSessionWithProcesses`, then re-checks; success is
"session no longer exists", not "KillSessionWithProcesses returned
nil".

### Polecat cleanup — `cleanupPolecats`

`start.go:831-923`. Loads rigs, iterates each, and for each polecat
checks `git.CheckUncommittedWork` on the polecat's clone:

- **Clean polecat** — `polecat.Manager.RemoveWithOptions(name, true,
  shutdownNuclear, false)` (the trailing `false` is `selfNuke=false`
  because this is a gt-shutdown cleanup, not a self-delete) and
  `git.DeleteBranch("polecat/<name>")` on the mayor clone.
- **Uncommitted work, default mode** — added to
  `uncommittedPolecats`, skipped, counted.
- **Unable to check status, default mode** — also skipped.
- **Any of the above under `--nuclear`** — deleted anyway with a
  `⚠ NUCLEAR` warning.

Summary prints the `uncommittedPolecats` list with a hint to re-run
with `--nuclear` if the user is sure.

### Daemon + orphan sweep — `stopDaemonIfRunning`

`start.go:928-970`. Primary check is `daemon.IsRunning` by PID file,
followed by a *fallback* sweep via `daemon.FindOrphanedDaemons` +
`daemon.KillOrphanedDaemons` — this runs whether or not the PID file
pointed to anything, so a missing PID file cannot leave a zombie
daemon behind.

### Flags

Defined at `start.go:144-161`:
- `--graceful / -g` (bool, default false)
- `--wait / -w` (int, default 30)
- `--all / -a` (bool, default false)
- `--force / -f`, `--yes / -y` (both skip confirmation)
- `--polecats-only` (bool, default false)
- `--nuclear` (bool, default false)
- `--cleanup-orphans` (bool, default false)
- `--cleanup-orphans-grace-secs` (int, default 60)

## Failure modes

No failure modes discovered. `shutdown` delegates to `gt down` with additional worktree cleanup. Wraps the `down` command with `--all` semantics. Error paths propagated from `down`.

## Notes / open questions

- **`gt shutdown` is older than `gt down`.** `gt down` has phases
  0 through 6 and a shutdown lock; `gt shutdown` does not take a
  lock at all. Two concurrent `gt shutdown` invocations will race
  over session kills.
- **Polecat-cleanup semantics differ from `gt down --all`.** `gt
  down` does not delete polecat worktrees or branches ever; this
  command does, because that's the whole point. See [down](./down.md)
  for the "pause" alternative.
- **`constants.ShutdownNotifyDelay`** — worth a dedicated
  `constants` page when one lands; controls how much time passes
  between the ESC and the notification message.
- **`cleanupOrphanedClaude` signatures** live elsewhere in the
  package (shared with `gt down`); worth confirming they're the same
  function when the cleanup module gets indexed.
- **Nuclear mode is logged but not beaded.** `⚠ NUCLEAR` lines go to
  stdout only — no audit trail if someone nukes a polecat with dirty
  work.

## Related

- [start](./start.md) — opposite direction; same source file
- [down](./down.md) — reversible pause (different shape; keeps
  worktrees)
- [up](./up.md) — boot command; complement of `down`
- [daemon](./daemon.md) — stopped explicitly in Phase 7
- [internal/daemon package](../packages/daemon.md) — `FindOrphanedDaemons` / `KillOrphanedDaemons` fallback sweep lives in this package alongside the main heartbeat loop
- [estop](./estop.md) — emergency freeze (SIGTSTP) rather than kill
- [polecat](./polecat.md) — target of the destructive cleanup phase
- [mayor](./mayor.md), [deacon](./deacon.md), [boot](./boot.md) —
  town-level sessions in the kill-order tail
