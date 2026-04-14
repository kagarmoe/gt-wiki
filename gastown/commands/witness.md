---
title: gt witness
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/witness.go
tags: [command, agents, witness, tmux, per-rig, polecat-monitor, lifecycle]
---

# gt witness

Lifecycle commands for the [Witness](../roles/witness.md) — the
per-rig polecat health monitor. One Witness per
[rig](../concepts/rig.md). This page documents the `gt witness`
CLI subcommands only; see the [Witness role page](../roles/witness.md)
for patrol loop, decision logic, and interaction with Mayor/Deacon.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`witness.go:26`)
**Polecat-safe:** no
**Beads-exempt:** yes (listed in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/witness.go`
(359 lines). Registration: `witness.go:122-143` wires flags and
attaches five subcommands to `witnessCmd`, then registers the
parent on `rootCmd`.

The parent `witnessCmd` (`witness.go:24-44`) uses
`RunE: requireSubcommand` — `gt witness` alone prints the
subcommand list.

### Invocation

```
gt witness start <rig>    [--agent <a>] [--env KEY=VAL]... [--foreground]
gt witness stop <rig>
gt witness attach [rig]
gt witness status <rig>   [--json]
gt witness restart <rig>  [--agent <a>] [--env KEY=VAL]...
```

Role-shortcut note from the Long help at `witness.go:43`:
`"witness"` in mail/nudge addresses resolves to this rig's
Witness.

### Long help (role framing)

`witness.go:29-43` is the canonical CLI-level summary of what the
Witness does:

> The Witness patrols a single rig, watching over its polecats:
>   - Detects stalled polecats (crashed or stuck mid-work)
>   - Nudges unresponsive sessions back to life
>   - Cleans up zombie polecats (finished but failed to exit)
>   - Nukes sandboxes when polecats complete via 'gt done'
>
> The Witness does NOT force session cycles or interrupt working
> polecats. Polecats manage their own sessions (via gt handoff).
> The Witness handles failures and edge cases only.
>
> One Witness per rig. The Deacon monitors all Witnesses.

The Long help on `witnessStartCmd` (`witness.go:51-63`) adds the
"Self-Cleaning Model" framing: polecats nuke themselves after
work; the Witness handles crash recovery (restart with hooked
work) and orphan cleanup. "There is no 'idle' state — polecats
either have work or don't exist."

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `start`  (alias `spawn`) | `witnessStartCmd`   | `witness.go:46-66`  | `runWitnessStart`   (`witness.go:156-189`) |
| `stop`   | `witnessStopCmd`    | `witness.go:68-76`  | `runWitnessStop`    (`witness.go:191-224`) |
| `status` | `witnessStatusCmd`  | `witness.go:78-86`  | `runWitnessStatus`  (`witness.go:234-290`) |
| `attach` (alias `at`)    | `witnessAttachCmd`  | `witness.go:88-105` | `runWitnessAttach`  (`witness.go:297-332`) |
| `restart`| `witnessRestartCmd` | `witness.go:107-120`| `runWitnessRestart` (`witness.go:334-359`) |

### `start`

`runWitnessStart` (`witness.go:156-189`). Accepts the rig as a
required positional, calls `checkRigNotParkedOrDocked(rigName)` to
refuse parked/docked rigs, gets a `witness.Manager`, and calls
`mgr.Start(foreground, agentOverride, envOverrides)`.

Special return path: `witness.ErrAlreadyRunning` is converted to
a warning `"⚠ Witness is already running"` plus a hint to use
`gt witness attach`, and the function returns nil (not an error)
— `witness.go:171-175`.

**Foreground mode is effectively a stub.** At `witness.go:179-183`:

```
if witnessForeground {
    fmt.Printf("%s Note: Foreground mode no longer runs patrol loop\n", ...)
    fmt.Printf("  %s\n", ...("Patrol logic is now handled by mol-witness-patrol molecule"))
    return nil
}
```

The `--foreground` flag still exists on the command but the
patrol-loop-in-foreground semantics are gone. Patrol logic has
moved to the `mol-witness-patrol` molecule, invoked by the
Witness agent itself.

### `stop`

`runWitnessStop` (`witness.go:191-224`). Takes a rig, gets a
tmux handle, computes the session name via
`witnessSessionName(rigName)` (`witness.go:293-295`, which wraps
`session.WitnessSessionName(session.PrefixFor(rigName))`), and if
a session is running, calls `t.KillSessionWithProcesses` — the
descendant-killing variant — to ensure Claude and its children
are actually dead.

Then calls `mgr.Stop()` to update the state file, but tolerates
`witness.ErrNotRunning` when the session had already been killed
(`witness.go:211-220`). If the tmux kill succeeded, a subsequent
manager.Stop error is non-fatal ("if we killed the session it's
stopped").

### `status`

`runWitnessStatus` (`witness.go:234-290`). ZFC-style state check:
`mgr.IsRunning()` consults tmux directly; `r.Polecats` is read
from the rig config. The monitored polecats list is sourced from
rig config, NOT from state. Comment at `witness.go:249`:
"Polecats come from rig config, not state file."

JSON output shape is `WitnessStatusOutput` at `witness.go:227-232`:

```go
type WitnessStatusOutput struct {
    Running           bool     `json:"running"`
    RigName           string   `json:"rig_name"`
    Session           string   `json:"session,omitempty"`
    MonitoredPolecats []string `json:"monitored_polecats,omitempty"`
}
```

### `attach`

`runWitnessAttach` (`witness.go:297-332`). Rig is optional — if
omitted, inferred from cwd via `workspace.FindFromCwdOrError` +
`inferRigFromCwd` (`witness.go:303-313`).

Auto-starts the Witness if needed: calls
`mgr.Start(false, "", nil)` and tolerates `ErrAlreadyRunning`. On
a fresh start path, prints `"Started witness session for <rig>"`.
Then calls the shared `attachToTmuxSession` helper, which handles
the link-if-inside-tmux vs. attach-if-outside distinction and
uses the town's tmux socket.

### `restart`

`runWitnessRestart` (`witness.go:334-359`). Rejects parked/docked
rigs, then best-effort `mgr.Stop()` (ignoring error), then
`mgr.Start(false, agentOverride, envOverrides)`. Note: always
starts in background (first arg is hardcoded `false`), ignoring
any previous foreground state. Comparable in shape to
`runMayorRestart` / `runDeaconRestart`.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--foreground`      | bool   | `false` | `start` | `witness.go:124` |
| `--agent <alias>`   | string | `""`    | `start`, `restart`, (also wired on attach but no-op) | `witness.go:125,132` |
| `--env KEY=VALUE`   | []string | `nil` | `start`, `restart` | `witness.go:126,133` (`StringArrayVar`, repeatable) |
| `--json`            | bool   | `false` | `status` | `witness.go:129` |

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [refinery.md](refinery.md) — sibling per-rig agent; both run
  inside the same rig and share the `getRig` / `inferRigFromCwd`
  helpers. Refinery handles merges, Witness handles polecat
  health.
- [deacon.md](deacon.md) — "The Deacon monitors all Witnesses"
  (`witness.go:41`). Deacon health-checks target Witnesses via
  mail addresses.
- [mayor.md](mayor.md) — the Witness escalates NEEDS_RECOVERY
  cases to the Mayor per the
  [polecat.md](polecat.md) `check-recovery` verdict path.
- [polecat.md](polecat.md) — the Witness's subject of observation.
  `gt polecat git-state`, `gt polecat check-recovery`, and
  `gt polecat nuke` are all commands the Witness issues during
  its patrol cycle (via the `mol-witness-patrol` molecule).
- [molecule.md](molecule.md) — `mol-witness-patrol` is the
  molecule referenced at `witness.go:181` as the current home of
  the patrol loop.
- [done.md](done.md) — `gt done` is the polecat-side completion
  command the Long help at `witness.go:35` names as the trigger
  for Witness sandbox nuking.
- [handoff.md](handoff.md) — `gt handoff` is named at
  `witness.go:39` as the mechanism by which polecats manage their
  own sessions (the thing the Witness does NOT do).
- [sling.md](sling.md) — work delivery. Restarted polecats
  recover by picking up their hooked bead via sling.
- [agents.md](agents.md) — Witnesses appear in the
  `gt agents list` enumeration alongside the other rig-level
  agents.
- [mq.md](mq.md) — merge queue; the Witness does not touch it
  directly but the broader work pipeline flows through the MQ.
- [status.md](status.md) — town-level status includes witness
  running state.

## Notes / open questions

- **`--foreground` is vestigial.** The flag still parses but no
  longer drives a foreground patrol loop (`witness.go:179-182`).
  Anyone using it in scripts or docs expecting the old behavior
  will be surprised.
- **`witnessAgentOverride` is a shared global** across start,
  attach, and restart flag definitions (`witness.go:125,132`).
  Attach wires it into its flag set but `runWitnessAttach` never
  reads it — the auto-start path uses `mgr.Start(false, "", nil)`
  with an empty agent override (`witness.go:324`). Dead flag on
  the attach surface.
- **`witness.go:132` reuses `witnessAgentOverride`** from the
  start definition without resetting. Both start and restart
  write the same global; sequential invocations within one
  `gt witness` batch (not supported via CLI but hypothetically
  via a tmuxinator-style wrapper) would clobber each other.
- **Stop is tmux-first, manager-second.** The code kills the
  session before updating the state file, and the state update
  is allowed to fail if the tmux kill succeeded. This matches
  the ZFC principle stated at `witness.go:245-246` ("tmux is
  source of truth for running state") — state files are caches
  of tmux reality, not authoritative.
- **MonitoredPolecats is whatever the rig config says.** Not
  filtered by whether the polecats actually exist on disk or
  have agent beads. `gt witness status` can show polecats that
  the Witness never actually sees.
- **`getWitnessManager` is a thin helper** (`witness.go:146-154`)
  that calls `getRig` and wraps the result with
  `witness.NewManager`. No error wrapping beyond what `getRig`
  provides.
- **Role/persona page pending Batch 6.** The Witness's decision
  logic (when to escalate, when to nuke, how many attempts
  before force-kill) lives inside `internal/witness/` and the
  patrol molecule — not in this command file.
