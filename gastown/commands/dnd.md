---
title: gt dnd
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/dnd.go
  - /home/kimberly/repos/gastown/internal/cmd/notify.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, dnd, notifications, focus]
---

# gt dnd

Toggle the current agent's Do Not Disturb mode — a coarse shortcut
around the three-level notification-level state that [notify](notify.md)
manages.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`dnd.go:15`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on `dndCmd`)
**Beads-exempt:** yes — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:67`
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/dnd.go:13-135`.

### Invocation

```
gt dnd [on|off|status]
```

`Args: cobra.MaximumNArgs(1)`. Zero args means "toggle". One arg
selects the action.

### Behavior

`runDnd` at `dnd.go:44-135`:

1. **Resolve agent identity** (`dnd.go:45-72`):
   - `os.Getwd()` → cwd.
   - `workspace.FindFromCwdOrError()` → town root; fails with
     "not in a Gas Town workspace" if missing.
   - `GetRoleWithContext(cwd, townRoot)` → role/rig/polecat.
   - Build a `RoleContext` and pass to `getAgentBeadID(ctx)`. If no
     bead ID can be derived, returns
     `"could not determine agent bead ID for role <role>"`.
2. **Read current level** via
   `bd.GetAgentNotificationLevel(agentBeadID)` (`dnd.go:77-81`). On
   error (agent bead doesn't exist yet), default to
   `beads.NotifyNormal`.
3. **Resolve action** (`dnd.go:84-94`):
   - No args → toggle: `muted` flips to `off` (= normal), anything
     else flips to `on` (= muted).
   - One arg → use that literal (`on` / `off` / `status`).
4. **Dispatch** (`dnd.go:96-131`):
   - `on` → `UpdateAgentNotificationLevel(agentBeadID, NotifyMuted)`.
     Prints "DND enabled - notifications muted" and a hint about
     `gt dnd off`.
   - `off` → `UpdateAgentNotificationLevel(agentBeadID, NotifyNormal)`.
     Prints "DND disabled - notifications resumed".
   - `status` → prints the current level as an icon + bold name +
     dim description. Icons: `🔔` normal, `🔊` verbose, `🔕` muted.
   - Any other value → error `unknown action %q: use on, off, or
     status`.

### Two-state vs three-state

`dnd` is a **two-state binary** (muted / normal) layered over the
three-state model (`verbose`, `normal`, `muted`) that [notify](notify.md)
exposes. Important consequence: running `gt dnd off` from a `verbose`
state **drops you to `normal`**, silently erasing the verbose
preference. There's no "restore previous level" memory. Users who
want verbose must re-run `gt notify verbose` after disabling DND.

### Subcommands

None; `on`/`off`/`status` are positional args on `dndCmd`, not
child cobra commands.

### Flags

None.

### Who honors DND

The level is stored on the agent bead via beads' notification-level
API. Consumers:

- [nudge](nudge.md) — calls `shouldNudgeTarget(townRoot, target,
  force)` before delivering. If the target is muted and
  `--force` is not set, the nudge is skipped with a dim
  `○ Target has DND enabled (<level>) - nudge skipped` message.
- [broadcast](broadcast.md) — same DND check at `broadcast.go:117`,
  but broadcast has **no `--force` flag** to override it.
- Other parts of the notification pipeline (convoy landing, mail
  auto-notification) are gated on this level too, per the notify
  help text.

### Related commands

- [notify](notify.md) — the underlying three-level control
  (`verbose`, `normal`, `muted`). DND is a convenience layer.
- [nudge](nudge.md) — honors DND; has `--force` to override.
- [broadcast](broadcast.md) — honors DND; no override.
- [status](status.md) — agent status view; worth cross-checking
  whether it surfaces the current DND level.
- [config](config.md) — root for agent-level config toggles.

## Notes / open questions

- **Beads-exempt but requires beads to work.** `dnd` is in the
  beads-exempt allowlist (`root.go:67`) — meaning it bypasses the
  "bd is installed and current" preflight — but `runDnd` calls
  `beads.New(townRoot).GetAgentNotificationLevel(...)`. The
  exemption seems to exist so that `dnd on` works during degraded
  bd states (e.g. dolt down), not because the command is bd-free.
  Worth a test.
- **Verbose loss on toggle** — `gt dnd off` unconditionally writes
  `NotifyNormal`, erasing a prior `NotifyVerbose`. Is this
  intentional or a papercut? [notify](notify.md)'s description of
  `verbose` ("mail, convoy events, status updates") makes it a
  real preference that could be worth preserving.
- **No global/room DND** — DND is per-agent only. There is no
  "mute all crew in rig <x>" primitive.
- **Polecat access** — `dndCmd` is not polecat-safe, meaning
  polecats can't set their own DND. Given that the whole Comm
  group is about agents talking to each other, a muted polecat
  would be useful; flag for possible follow-up.
