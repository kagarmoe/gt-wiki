---
title: gt boot
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/boot.go
  - /home/kimberly/repos/gastown/internal/dog/manager.go
tags: [command, agents, boot, deacon, watchdog, triage, daemon]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# gt boot

Manage Boot — the daemon's watchdog for Deacon triage. Boot is a
short-lived "special dog" that runs on each daemon tick to observe
Gas Town state and decide whether to start/wake/nudge/interrupt the
Deacon.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`boot.go:29`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/boot.go` (397
lines). Registration: `boot.go:90-100` attaches three subcommands and
registers the parent on `rootCmd`. The parent `bootCmd`
(`boot.go:27-46`) has no `RunE` — running `gt boot` with no arguments
simply prints help (cobra default).

### Invocation

```
gt boot status  [--json]
gt boot spawn   [--agent <alias>]
gt boot triage  [--degraded]
```

### Boot concept (from the parent Long help, `boot.go:31-45`)

Boot is "a special dog that runs fresh on each daemon tick. It
observes the system state and decides whether to start/wake/nudge/
interrupt the Deacon, or do nothing. This centralizes the 'when to
wake' decision in an agent that can reason about it."

Boot lifecycle per the Long help:

1. Daemon tick spawns Boot (fresh each time).
2. Boot runs triage: observe, decide, act.
3. Boot cleans inbox (discards stale handoffs).
4. Boot exits (or handoffs in non-degraded mode).

Filesystem location: `~/gt/deacon/dogs/boot/`. Session name:
`gt-boot`. Boot sessions are explicitly excluded from
[agents.md](agents.md) listings (`agents.go:321` filters out the
session with name `session.BootSessionName()`).

### Subcommands

**`boot status`** — `bootStatusCmd` at `boot.go:48-59`, implemented
by `runBootStatus` (`boot.go:111-193`).

Calls `boot.New(townRoot)` via `getBootManager` (`boot.go:102-109`)
and inspects four pieces of state: `LoadStatus()` for the last
execution record, `IsRunning()` for active-run lock, `IsSessionAlive()`
for the `gt-boot` tmux session, and `IsDegraded()` for no-tmux mode.

With `--json`, emits a flat object with keys `running`,
`session_alive`, `degraded`, `boot_dir`, and `last_status`. Without
`--json`, prints a styled block showing state, session, mode, last
completed/started time, last action, target, error, and the Boot
directory. Durations are formatted by `formatDurationAgo`
(`boot.go:374-397`): "just now", "N min", "N hours", or "N days".

**`boot spawn`** — `bootSpawnCmd` at `boot.go:61-73`, implemented by
`runBootSpawn` (`boot.go:195-229`).

> "This is normally called by the daemon." If Boot is already
> running, the spawn is a no-op with message "Boot is already running
> - skipping spawn" (`boot.go:201-204`). Otherwise it writes a new
> `Status{StartedAt: now}`, then calls `b.Spawn(agentOverride)` —
> which creates a fresh tmux session, or in degraded mode runs as a
> subprocess. On spawn error, the status is updated with the error
> and a completion timestamp before the error is returned.

**`boot triage`** — `bootTriageCmd` at `boot.go:75-88`, implemented
by `runBootTriage` (`boot.go:231-275`).

Runs triage logic directly without Claude, for degraded-mode
operation when tmux is unavailable (`boot.go:77-86`). Acquires a
Boot lock, runs `runDegradedTriage`, records the action/target/error
on the status file, and prints `Triage complete: <action> → <target>`.

### Degraded triage (`runDegradedTriage`, `boot.go:284-329`)

The in-file comment (`boot.go:277-283`) explains the design:

> ZFC principle: "Agent decides. Go transports." Complex triage
> decisions (heartbeat staleness, idle detection, backoff awareness,
> molecule progress) belong in the Boot agent's mol-boot-triage
> formula, not in Go code. This function handles only mechanical
> safety checks that must work even when no AI agent is available.

Mechanical steps:

1. If `daemon.IsShutdownInProgress(townRoot)` returns true, return
   `("nothing", "shutdown-in-progress", nil)` immediately. Without
   this, Boot could detect Deacon as down during graceful shutdown
   and restart it, creating a zombie that survives `gt down`
   (`boot.go:286-291`).
2. Call `executeWarrants(townRoot/warrants, tm)` as a side effect —
   scans `.warrant.json` files and executes any pending ones (see
   `boot.go:335-371`). Errors are non-fatal.
3. Look up the Deacon session name via `getDeaconSessionName()`
   (defined in [deacon.md](deacon.md)) and call `tm.HasSession`.
4. If Deacon is missing: call `deacon.NewManager(townRoot).Start("")`,
   return `("start", "deacon-restarted", nil)`. `ErrAlreadyRunning`
   is treated as success.
5. If Deacon is present: return `("nothing", "", nil)`. Deeper
   health assessment is left to the Boot agent's molecule.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--json` | bool | `false` | `status` | `boot.go:91` |
| `--degraded` | bool | `false` | `triage` | `boot.go:92` |
| `--agent <alias>` | string | `""` | `spawn` | `boot.go:93` |

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/boot.go:31-45` — Cobra `Long` text on the parent `bootCmd`.

### Verbatim

> Manage Boot - the daemon's watchdog for Deacon triage.
>
> Boot is a special dog that runs fresh on each daemon tick. It observes
> the system state and decides whether to start/wake/nudge/interrupt the
> Deacon, or do nothing. This centralizes the "when to wake" decision in
> an agent that can reason about it.
>
> Boot lifecycle:
>   1. Daemon tick spawns Boot (fresh each time)
>   2. Boot runs triage: observe, decide, act
>   3. Boot cleans inbox (discards stale handoffs)
>   4. Boot exits (or handoffs in non-degraded mode)
>
> Location: ~/gt/deacon/dogs/boot/
> Session: gt-boot

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Boot is "a special dog" per Cobra `Long`, but is explicitly NOT a dog in the kennel

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/boot.go:33`.
- **Docs claim:** the `Long` text opens with `"Boot is a special dog that runs fresh on each daemon tick."` The framing is introduced as the canonical mental model for what Boot is.
- **Code does:** `internal/dog/manager.go:300` is explicit: in `Manager.Get()`, if `loadState(name)` fails (no `.dog.json`) the function returns `ErrDogNotFound` with the inline comment `"No .dog.json means this isn't a valid dog worker (e.g., "boot" is the boot watchdog using .boot-status.json, not a dog)"`. A matching branch in `internal/dog/manager_lifecycle_test.go:326-332` ("Create dog directory but no `.dog.json` (e.g., boot watchdog)") confirms this is deliberate. Boot also has **no `AgentDog` entry** in the `AgentType` enum at `/home/kimberly/repos/gastown/internal/cmd/agents.go:24-33` (there is no `AgentDog` type in the code tree at all), and `bootCmd` lives in `GroupAgents` rather than being registered as a subcommand of `dogCmd`. The only thing Boot shares with dogs is the `~/gt/deacon/dogs/boot/` filesystem path (which the dog manager explicitly excludes from dog enumeration). Boot's persistence state lives in `.boot-status.json`, not `.dog.json`; its lifecycle is daemon-tick-driven (fresh per tick) rather than kennel-managed (reusable idle→working→idle); and its agent bead is created via a different path. The `"special dog"` framing is an intuition pump that directly contradicts the code's dog-manager exclusion.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — edit the `Long` text in `boot.go` to drop or reframe the "special dog" metaphor. Candidate replacements: `"Boot is a daemon-tick watchdog"`, `"Boot is a short-lived triage agent that runs fresh on each daemon tick"`, or `"Boot is a special-purpose agent under ~/gt/deacon/dogs/boot/ but is not a kennel dog"`. Any replacement that preserves the "fresh per daemon tick" semantics without inviting the "it's a dog" mental model is acceptable.
- **Release position:** `in-release` — `boot.go:31-45` is byte-identical at `v1.0.0`; `internal/dog/manager.go:300` "not a dog" exclusion is present at `v1.0.0`.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [deacon.md](deacon.md) — Boot exists to triage the Deacon and
  calls into the `internal/deacon` package to start it when missing.
  See especially `gt deacon start` / `gt deacon status` /
  `gt deacon heartbeat`.
- [agents.md](agents.md) — filters out Boot's session by name so it
  never appears in the agent list or menu.
- [mayor.md](mayor.md) — Boot does not interact with the Mayor
  directly in degraded mode, but the Long help calls out
  "start/wake/nudge/interrupt the Deacon" as its decision surface.
- [scheduler.md](scheduler.md), [heartbeat.md](heartbeat.md) —
  adjacent daemon-tick primitives.
- [doctor.md](doctor.md) — for diagnosing daemon/Boot interaction
  problems.

## Notes / open questions

- **"Special dog" vs [dog.md](dog.md).** → promoted to `## Drift` as
  `cobra drift`: the `Long` text at `boot.go:33` calls Boot "a special
  dog" but `internal/dog/manager.go:300` explicitly excludes Boot from
  the kennel, and no `AgentDog` type exists anywhere in the code tree.
  See [../drift/README.md](../drift/README.md).
- **`boot spawn` in non-degraded mode** claims to run the agent in a
  fresh tmux session, but the actual behavior depends on
  `boot.Spawn` in the `internal/boot` package — not visible from
  this file. What does `--agent` override when degraded mode is on?
- **`executeWarrants` is a side effect** of degraded triage. The
  `Warrant` type (referenced at `boot.go:356`) and the warrant
  execution logic live outside this file. Worth a dedicated page.
- **No `boot stop` subcommand.** Because Boot is fresh per tick,
  there is no lifecycle command to gracefully shut it down; killing
  the `gt-boot` session is the closest equivalent.
- **Lock coordination.** `runBootTriage` calls
  `b.AcquireLock()` / `b.ReleaseLock()`, but `runBootSpawn` relies on
  `b.IsRunning()` without acquiring a lock itself. Whether concurrent
  `spawn` + `triage` calls can race is not obvious from this file.
