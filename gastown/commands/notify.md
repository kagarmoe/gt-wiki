---
title: gt notify
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/notify.go
  - /home/kimberly/repos/gastown/internal/cmd/dnd.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, notifications, notification-level]
---

# gt notify

Set or display the current agent's notification level — the
three-state control that sits under [dnd](dnd.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`notify.go:15`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on `notifyCmd`)
**Beads-exempt:** no (not in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/notify.go:13-131`.

### Invocation

```
gt notify [verbose|normal|muted]
```

`Args: cobra.MaximumNArgs(1)`. Zero args prints the current level;
one arg sets it. No flags.

### Levels

From the Long help (`notify.go:19-23`) and runtime descriptions
in `showNotificationLevelDescription` (`notify.go:122-131`):

| level     | icon | description (displayed) |
|-----------|------|-------------------------|
| `verbose` | `🔊` | All notifications: mail, convoy events, status updates |
| `normal`  | `🔔` | Important notifications: convoy landed, escalations |
| `muted`   | `🔕` | Silent mode: notifications batched for later review |

Default is `normal` if no level has ever been set.

### Behavior

`runNotify` at `notify.go:41-103`:

1. **Resolve agent identity** (`notify.go:42-69`) — identical
   pattern to [dnd](dnd.md): `os.Getwd` → `FindFromCwdOrError` →
   `GetRoleWithContext` → `RoleContext` → `getAgentBeadID`. Fails
   on "not in a Gas Town workspace" or inability to determine the
   bead ID.
2. **Read current level** via
   `bd.GetAgentNotificationLevel(agentBeadID)`
   (`notify.go:73-78`); on error, fall back to
   `beads.NotifyNormal`.
3. **If no args** (`notify.go:81-84`) — call
   `showNotificationLevel(currentLevel)` (prints icon + bold
   name + description) and return.
4. **Validate new level** (`notify.go:88-93`) against the constants
   `beads.NotifyVerbose`, `beads.NotifyNormal`, `beads.NotifyMuted`.
   Invalid input → `invalid level %q: use verbose, normal, or muted`.
5. **Persist** via
   `bd.UpdateAgentNotificationLevel(agentBeadID, newLevel)`
   (`notify.go:95-97`).
6. **Echo** the new state with a description line
   (`notify.go:99-100`).

### Display helpers

`showNotificationLevel` (`notify.go:105-120`) is a pure
formatter — icon + bold level name + description. It defaults an
empty level to `normal` (`notify.go:106-108`) so the "not yet set"
case renders cleanly.

`showNotificationLevelDescription` (`notify.go:122-131`) is the
switch that prints the descriptive line under the level. Reused
when setting via step 6 above. Note the `dnd status` path prints
a near-identical message from `dnd.go:110-128` — the two
commands' output is deliberately the same shape.

### Subcommands

None.

### Flags

None.

### Relationship to `gt dnd`

`gt dnd on` is equivalent to `gt notify muted`. `gt dnd off` is
equivalent to `gt notify normal`. `gt dnd status` prints the same
icon/level/description block as `gt notify` (no args). There is
**no `gt dnd verbose`** — verbose is `notify`-only. Users who
want verbose must use `gt notify` directly; flipping DND off from
a prior verbose state resets to normal, not back to verbose. See
the "two-state vs three-state" note on [dnd](dnd.md).

### Related commands

- [dnd](dnd.md) — binary (muted/normal) convenience wrapper.
- [nudge](nudge.md) — honors the notification level via
  `shouldNudgeTarget`; muted targets are skipped unless
  `--force`.
- [broadcast](broadcast.md) — same DND gate, no force override.
- [mail](mail.md) — the mail auto-notify flag
  (`--notify`/`--no-notify` at `mail.go:470-472`) produces a
  nudge to the recipient; that nudge is subject to the
  recipient's notification level.
- [status](status.md) — displays agent state; worth checking
  whether the notification level is shown there.
- [agents](agents.md) — enumerates agents; could eventually grow
  a `gt agents --show-dnd` view.

## Notes / open questions

- **No per-sender / per-channel filtering.** The level is a
  single global state per agent. There's no "mute this sender"
  or "mute convoy events but not mail" primitive.
- **Verbose fires on more event types.** The description
  mentions "convoy events" and "status updates" for verbose only.
  The actual event-producer code is elsewhere
  (`internal/events/`, `internal/convoy/`?) — worth mapping the
  producers that consult this level.
- **No `--json` output.** The "show current level" path prints
  for humans only. Scripted callers that want the level must
  parse stdout or call the beads API directly.
- **"Batched for later review"** language for muted implies
  some form of delayed delivery / digest queue. Is there a real
  batch consumer, or is muted simply "drop these events on the
  floor"? Follow up by reading `shouldNudgeTarget` and whatever
  produces mail auto-notifications.
