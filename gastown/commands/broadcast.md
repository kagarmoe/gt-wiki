---
title: gt broadcast
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/broadcast.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, nudge, fan-out, workers]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt broadcast

Fan-out a nudge to every running worker (and optionally every
infrastructure agent) in one call.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`broadcast.go:29`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on `broadcastCmd`)
**Beads-exempt:** no (not in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/broadcast.go:27-158`.

### Invocation

```
gt broadcast <message> [flags]
```

`Args: cobra.ExactArgs(1)` — exactly one positional: the message body.
There is no `-m/--message` flag; the message is always positional.

### Behavior

`runBroadcast` at `broadcast.go:47-158`:

1. **Enumerate sessions** via `getAgentSessions(true)`
   (`broadcast.go:54-58`) — the `true` includes polecats. Same helper
   used by [agents](agents.md).
2. **Read sender** from `BD_ACTOR` env var (`broadcast.go:61`) so the
   broadcaster can exclude itself from the target list.
3. **Filter** (`broadcast.go:64-84`):
   - `--rig <name>` — drop agents whose `Rig` doesn't match.
   - Without `--all` — drop everything except `AgentCrew` and
     `AgentPolecat` (i.e. "workers only" is the default; infrastructure
     roles like mayor, deacon, witness, refinery are excluded).
   - Always — drop the caller (self) identified by `BD_ACTOR`
     matching `formatAgentName(agent)`.
4. **Dry-run** (`broadcast.go:95-102`) — if `--dry-run`, print the
   target list + the message and return. No nudges are sent.
5. **DND check per target** (`broadcast.go:115-122`) — calls
   `shouldNudgeTarget(townRoot, agentName, false)`. The `false` means
   DND is respected (no force). Skipped targets are reported as
   `(DND: <level>)`.
6. **Send** via `t.NudgeSession(agent.Name, message)`
   (`broadcast.go:124`). Note this uses the direct `NudgeSession`
   entrypoint, **not** the fancy `deliverNudge` wait-idle/queue
   machinery [nudge](nudge.md) uses — broadcast is always an
   immediate tmux send-keys. There is no `--mode` flag.
7. **Pacing** — `time.Sleep(100 * time.Millisecond)` between nudges
   (`broadcast.go:133-136`) to avoid overwhelming tmux.
8. **Summary** — prints succeeded / failed / skipped counts, with
   failures listed dimly below. Returns a non-nil error if any nudge
   failed (`broadcast.go:149`).

### Target selection details

`broadcast.go:72-75` — "workers" means `AgentCrew` or `AgentPolecat`.
`--all` flips the filter off, widening the target set to include
`AgentMayor`, `AgentDeacon`, `AgentWitness`, `AgentRefinery` as well.
The Long help (`broadcast.go:31-34`) describes this as "worker" vs
"infrastructure agents".

### Subcommands

None.

### Flags

Defined in `init()` at `broadcast.go:20-25`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--rig` | string | `""` | Only broadcast to workers in this rig |
| `--all` | bool | `false` | Include all agents (mayor, witness, etc.), not just workers |
| `--dry-run` | bool | `false` | Show what would be sent without sending |

### Helper: `formatAgentName`

Defined at `broadcast.go:160-177` but used across the Comm group.
Returns the canonical display name per agent type:

- `AgentMayor` → `"mayor"`
- `AgentDeacon` → `"deacon"`
- `AgentWitness` → `"<rig>/witness"`
- `AgentRefinery` → `"<rig>/refinery"`
- `AgentCrew` → `"<rig>/crew/<name>"`
- `AgentPolecat` → `"<rig>/<name>"`

This is the same address scheme consumed by [nudge](nudge.md) and
[mail](mail.md).

### Related commands

- [nudge](nudge.md) — broadcast is "nudge, to everybody at once".
  `broadcast` is not polecat-safe and lacks nudge's `--mode` /
  queue / wait-idle delivery options; broadcast always uses the
  immediate tmux send-keys path.
- [mail](mail.md) — durable alternative. Broadcast leaves no trace
  in beads; every target gets exactly one ephemeral nudge.
- [dnd](dnd.md) — DND state on each target is honored (no
  `--force` equivalent exists on broadcast).
- [agents](agents.md) — shares `getAgentSessions` for enumeration.
- [notify](notify.md) — per-agent notification level gate that
  DND sits on top of.

## Notes / open questions

- **No `--force`** — targets with DND enabled are always skipped. If
  you need to push through, you have to `gt nudge` each one
  individually with `--force`.
- **No queue mode** — broadcast always uses immediate delivery.
  Nudging 20 busy polecats with broadcast will interrupt 20 active
  tool calls. This is presumably deliberate (broadcast is for "hey
  everyone") but worth flagging as a footgun.
- **Not polecat-safe** — even though it's in the Comm group and it
  sends nudges, polecats cannot run it. The policy appears to be:
  polecats can send 1:1 messages ([mail](mail.md), [nudge](nudge.md))
  but cannot fan out to the whole crew.
- **No test coverage file** — no `broadcast_test.go` sibling. The
  sibling files for the Comm group (`mail_*_test.go`,
  `nudge_test.go`, `dnd_test.go`, `escalate_test.go`) do not
  include broadcast.
- **100ms inter-send delay** — hard-coded; not configurable.
