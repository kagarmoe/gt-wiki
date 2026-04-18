---
title: gt up
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/up.go
tags: [command, services, lifecycle, boot, dolt, daemon, mayor, deacon]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
---

# gt up

Idempotent "boot" command: brings up all Gas Town long-lived services
(Dolt server, daemon, deacon, mayor, per-rig witnesses and refineries)
in the right order, with parallel startup and a Dolt-readiness gate
before agents that depend on it. Reversible complement of
[down](./down.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/up.go:120-448`.

### Invocation

```
gt up                  # Start all infrastructure services
gt up --restore        # Also restore crew (from settings) and polecats with pinned work
gt up --quiet / -q     # Only show errors (ignored with --json)
gt up --json           # Emit a structured JSON report (and exit nonzero on failure)
```

What `gt up` does NOT start by default: crew workspaces (opt-in via
`--restore`) and polecats (opt-in, and only those with pinned beads).
Polecats are "transient workers spawned on demand by the Mayor or
Witnesses" per the Long help (`up.go:136-137`).

### Behavior

**Phase 0 — Workspace resolution + lifecycle config**
(`up.go:161-182`)
- `workspace.FindFromCwdOrError` — must be run inside a Gas Town
  workspace.
- `daemon.EnsureLifecycleConfigFile` — on first run, writes
  `mayor/daemon.json` with six-stage Dolt-lifecycle defaults. On
  subsequent runs, fills in newly added patrols without touching
  existing config. Non-fatal.
- Loads `daemon.LoadPatrolConfig` and exports its `Env` map into the
  process environment so services started next (Dolt) see the intended
  config.

**Phase 0.5 — Rig prefetch + DND reset** (`up.go:188-196`)
- `discoverRigs` (`up.go:764-811`) reads `mayor/rigs.json` or, as a
  fallback, scans the town root for directories containing `.beads/`
  or `polecats/`.
- `disableCurrentAgentDND` resets the current agent's notification
  level from `NotifyMuted` to `NotifyNormal` so orchestration nudges
  are not silently eaten after a previous debug session.

**Phase 1 — Parallel startup of Dolt, daemon, deacon, mayor, rig
prefetch** (`up.go:198-284`)
Five goroutines in a single `WaitGroup`:

0. **Dolt server** — if [`doltserver`](../packages/doltserver.md)`.DefaultConfig(townRoot).DataDir`
   exists, start it via `doltserver.Start`. Otherwise mark `doltSkipped`.
1. **Daemon** — `ensureDaemon` (`up.go:504-566`) checks for a
   `ShutdownSentinel` file first (GH#2656: don't restart the daemon
   mid-shutdown). If the sentinel's PID is dead, the sentinel is
   treated as stale and removed (GH#2907). Then checks
   `daemon.IsRunning`. If not running, re-execs `gt daemon run` via
   `os.Executable()` with a detached process group and waits
   `daemonStartupGrace` (300ms on Unix, 2s on Windows — see
   `up.go:113-118`) before verifying via `daemon.IsRunning`.
2. **Deacon** — `deacon.NewManager(townRoot).Start("")`. Treats
   `deacon.ErrAlreadyRunning` as success.
3. **Mayor** — `mayor.NewManager(townRoot).Start("")`. Treats
   `mayor.ErrAlreadyRunning` and `mayor.ErrACPActive` as success.
4. **Rig prefetch** — `prefetchRigs` (`up.go:577-608`) loads every
   rig's config in parallel using per-rig goroutines and a results
   channel, so agent startup in Phase 3 doesn't serialize on config
   I/O.

**Phase 2 — Dolt readiness + env propagation** (`up.go:317-337`)
If Dolt was started (or already running), `waitForDoltReady`
(`up.go:1010-1014`) polls `doltserver.WaitForReady` for up to
`doltReadyTimeout = 10 * time.Second` (`up.go:1004`). On timeout, logs
a warning but continues (graceful degradation). Then exports
`GT_DOLT_PORT`, `BEADS_DOLT_PORT`, and — if non-empty —
`BEADS_DOLT_SERVER_HOST` so every agent spawned next inherits the
connection info. Without this, `bd` auto-starts rogue embedded Dolt
instances in agent sessions (GH#2412).

**Phase 2.5 — Orphaned bead recovery** (`up.go:339-346`,
`recoverOrphanedBeads` at `up.go:1024-1064`)
For each rig, `witness.DetectOrphanedBeads` scans for beads stuck in
`hooked`/`in_progress` status assigned to polecats whose session is
dead *and* whose worktree is gone. Each orphan is reset to `open` and
the deacon is notified via `mail.Router`. Runs before witnesses start
so duplicate recovery is avoided (gas-udp).

**Phase 3 — Witnesses + refineries** (`up.go:348-367`,
`startRigAgentsWithPrefetch` at `up.go:626-709`)
Worker pool pattern: fixed `maxConcurrentAgentStarts = 10`
(`up.go:108`) goroutines drain a task queue containing
`(rig, witness|refinery)` pairs. Per-rig failures from prefetch are
recorded as failures for both the witness and the refinery for that
rig without attempting startup.

Each start call delegates to `upStartWitness` / `upStartRefinery`
(`up.go:713-761`). Both respect rig parked/docked status via
`IsRigParkedOrDocked`, *unless* the rig's config has
`auto_start_on_up` or the deprecated `auto_start_on_boot` set. Skipped
rigs are reported as `ok: true` with detail `skipped (rig parked)` or
`skipped (rig docked)`.

**Phase 4 — `--restore` (crew + polecats)** (`up.go:370-417`)
Opt-in. For each rig:
- `startCrewFromSettings` (`up.go:815-872`) reads
  `<rig>/settings/config.json` and parses its `crew.startup` field via
  `parseCrewStartupPreference` (`up.go:876-935`). That parser accepts
  natural-language values: `none`, `all`, `pick one`, `any`,
  `any one`, comma/`and`-separated lists (`"joe and max"`), and
  exclusions (`"max, but not joe"`). Unknown names are silently
  dropped.
- `startPolecatsWithWork` (`up.go:939-998`) lists `<rig>/polecats/`,
  and for each polecat queries its embedded beads DB for pinned beads
  assigned to `<rig>/polecats/<name>`. Only polecats with pinned work
  are started via `polecat.NewSessionManager(t, r).Start`.

**Phase 5 — Summary + events** (`up.go:419-448`)
On success, `events.LogFeed(events.TypeBoot, "gt", ...)` records
`dolt`, `daemon`, `deacon`, `mayor`, and `<rig>/witness` +
`<rig>/refinery` for each rig.

With `--json`, emits a structured `UpOutput` (`up.go:48-61`) containing
per-service `ServiceStatus` entries and a `UpSummary`; if any service
failed, `emitUpJSON` returns `NewSilentExit(1)` so the JSON still
prints cleanly and the process exits nonzero.

Otherwise prints per-service lines via `printStatus`
(`up.go:450-459`), honoring `--quiet` by suppressing OK lines only.

### Flags

Defined at `up.go:148-158`:
- `--quiet / -q` — errors only; ignored under `--json`.
- `--restore` — also start crew (from rig settings) and polecats with
  pinned beads.
- `--json` — structured output; exit 1 on any failure.

## Failure modes

### Partial completion

- **Parallel startup with no rollback:** `runUp` at `up.go:208-284` starts Dolt, daemon, deacon, mayor, and rig prefetch in 5 parallel goroutines. If daemon succeeds but deacon fails, there's no rollback. **Present** — by design; `gt up` is idempotent so a re-run starts missing services.
- **`EnsureAllMetadata` error discarded:** `up.go:288` uses `_, _ = doltserver.EnsureAllMetadata(townRoot)`. If metadata propagation fails, `bd` may auto-spawn rogue Dolt servers. **Absent** — silently continues.

### Silent suppression

- **Boot event fire-and-forget:** `up.go:426` uses `_ = events.LogFeed(...)`. **Absent** — audit gap.
- **Env var propagation:** `up.go:330-336` sets Dolt port/host into process env for child agent inheritance. **Present** — intentional.

## Notes / open questions

- **`ensureDaemon` does not wait as long as `daemon start`.** The
  poll in [daemon](./daemon.md) waits up to 3 seconds; `gt up` sleeps
  a single `daemonStartupGrace` (300ms/2s) and then verifies once. If
  the daemon takes longer than that grace window, `gt up` may report
  a transient failure even though the daemon comes up moments later.
- **Dolt readiness gates only agents, not the daemon.** The daemon
  goroutine starts concurrently with the Dolt goroutine in Phase 1 —
  only witnesses/refineries wait for Dolt. If the daemon reads Dolt
  during its first heartbeat, it races the server.
- **`IsRigParkedOrDocked` is auto_start_on_up / auto_start_on_boot
  gated**, but the comment notes `auto_start_on_boot` is deprecated —
  worth a dedicated rig-config page when one lands.
- **`startPolecatsWithWork` opens a bead DB per polecat directory**
  (`beads.New(polecatPath)`), which assumes each polecat still has its
  own `.beads` directory. In the Dolt-server world most beads live in
  the canonical server — so this path may be dormant / stale.
  Cross-reference with [dolt](./dolt.md).

## Related

- [down](./down.md) — reversible counterpart; stops everything this
  command starts. Phase-aligned (Dolt last to start, Dolt last to stop).
- [shutdown](./shutdown.md) — permanent "done for the day" with
  polecat worktree cleanup
- [start](./start.md) — legacy-flavored start that uses a different
  mutex pattern; overlaps with `gt up` but still shipped
- [daemon](./daemon.md) — `ensureDaemon` re-execs `gt daemon run`
- [internal/daemon package](../packages/daemon.md) — the Go implementation of the daemon process that `ensureDaemon` re-execs; lock discipline, `EnsureLifecycleConfigFile` call site
- [dolt](./dolt.md) — Dolt server lifecycle
- [doctor](./doctor.md) — diagnose what `gt up` may not have been able
  to bring online
- [maintain](./maintain.md) — Dolt-lifecycle defaults written by
  `EnsureLifecycleConfigFile` are what the maintain cycle reads
- [mayor](./mayor.md), [deacon](./deacon.md) — Phase 1 core agents
- [witness](./witness.md), [refinery](./refinery.md) — Phase 3
  per-rig agents
- [polecat](./polecat.md) — Phase 4 optional restoration target
  (polecats with pinned beads). Crew workspaces (also restorable) do
  not yet have a wiki page.
