---
title: gt cleanup
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/cleanup.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, process-management, zombie]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt cleanup

Find and kill orphaned Claude processes that outlived their tmux
session — a last-resort janitor, **not** a data-cleanup command.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`cleanup.go:18`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/cleanup.go:16-127`.
Despite the name, this is narrowly scoped: it kills orphaned
`claude` processes, nothing else. It does not clean up beads, data,
or configuration.

### Invocation

```
gt cleanup [--dry-run] [--force|-f]
```

### Behavior

`runCleanup` (`cleanup.go:45-114`):

1. **Find orphans** via `util.FindZombieClaudeProcesses()`
   (`cleanup.go:47-50`). The help text claims this uses "aggressive
   tmux session verification to detect ALL orphaned processes, not
   just those with PPID=1" (`cleanup.go:28-29`). The real
   implementation lives in [internal/util](../packages/util.md) and
   is out of scope for this page.
2. **Return early** with `✓ No orphaned Claude processes found` if
   none (`cleanup.go:52-55`).
3. **Print the zombie table** — PID, command, age (formatted via
   `formatProcessAgeCleanup`, `cleanup.go:117-127`), tty
   (`cleanup.go:57-67`).
4. **Dry-run short-circuit** — `--dry-run` exits after printing the
   table (`cleanup.go:69-72`).
5. **Confirm unless `--force`** — interactive `[y/N]` prompt via
   `fmt.Scanln` (`cleanup.go:74-83`). Accepts `y`, `Y`, `yes`, `Yes`.
6. **Kill** via `util.CleanupZombieClaudeProcesses()`
   (`cleanup.go:86-89`). This is the actual kill path — the earlier
   "find" call is just for display.
7. **Report per-process results** (`cleanup.go:92-105`), distinguishing
   three signals:
   - `SIGTERM` — success case, counted as `killed`.
   - `SIGKILL` — survived SIGTERM, needed escalation; still counted
     as `killed`, with a warning icon.
   - `UNKILLABLE` — survived even SIGKILL, counted as `escalated`.
8. **Summary line** — `✓ Cleaned up N process(es)` plus `, M
   unkillable` if any (`cleanup.go:107-111`).

### Subcommands

None.

### Flags

Defined in `init()` at `cleanup.go:38-43`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--dry-run` | — | bool | `false` | Show what would be killed without killing |
| `--force` | `-f` | bool | `false` | Kill without confirmation |

### `formatProcessAgeCleanup` helper

`cleanup.go:117-127` — formats integer seconds as `Xs`, `XmYs`, or
`XhYm`. The function name is suffixed `Cleanup` presumably because
another file in the package defines a similarly-named helper; check
for symbol clash before refactoring.

### Related commands

- [doctor](doctor.md) — the general diagnostic; `gt cleanup`'s scope
  (zombie Claude processes) could overlap with doctor's health checks
  but they're distinct entry points.
- [done](done.md), [handoff](handoff.md) — normal session lifecycle
  endings. When these fail (polecat killed unclean, Claude spawned a
  subprocess that outlived its parent), `gt cleanup` is the janitor.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Help text drift risk.** The `Long` string at `cleanup.go:20-34`
  claims the command handles cases where "Polecat sessions are killed
  without proper cleanup", "Claude spawns subagent processes that
  outlive their parent", and "Network or system issues interrupt
  normal shutdown". The implementation is pure orphan-PID detection —
  nothing in the cleanup code branches on these cases. The doc is
  describing *causes*, not *behaviors*.
- **The name "cleanup" is overbroad.** A reader scanning the command
  index might reasonably expect data or workspace cleanup. Consider
  whether [compact](compact.md) (wisp TTL compaction), [cleanup]
  (this), and `gt dolt cleanup` (from [CLAUDE.md](../../CLAUDE.md))
  create a confusing three-axis "cleanup" space.
- **No JSON output.** Unlike [changelog](changelog.md) and
  [compact](compact.md), there's no machine-readable mode. Agents
  invoking this programmatically would have to scrape the human
  output.
