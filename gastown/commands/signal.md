---
title: gt signal
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/signal.go
  - /home/kimberly/repos/gastown/internal/cmd/signal_stop.go
tags: [command, agents, signal, claude-code, hooks, turn-boundary]
---

# gt signal

Signal handlers for Claude Code's hook system. Not intended for direct
use by humans — these subcommands emit JSON that Claude Code interprets
at well-defined hook points.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`signal.go:14`)
**Polecat-safe:** no
**Beads-exempt:** yes (listed in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Sources:
- `/home/kimberly/repos/gastown/internal/cmd/signal.go` (37 lines) —
  the parent `signalCmd` with group, short/long descriptions, and
  registration of the single subcommand.
- `/home/kimberly/repos/gastown/internal/cmd/signal_stop.go` (272
  lines) — the `stop` subcommand, `stopHookResponse` shape, and all
  the check/state logic.

The parent `signalCmd` (`signal.go:12-37`) has no `RunE` — running
`gt signal` alone prints the cobra help text. Only one subcommand is
wired today: `stop`.

### Invocation

```
gt signal stop
```

No flags, no positional arguments (`signal_stop.go:40`
`Args: cobra.NoArgs`). It is designed to be configured in Claude
Code's `.claude/settings.json` as the command run on the `Stop` hook
— the Long help at `signal.go:16-36` shows the exact JSON block.

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| `stop` | `signalStopCmd` | `signal_stop.go:24-45` | `runSignalStop` (`signal_stop.go:47-108`) |

`signalStopCmd` sets both `SilenceUsage: true` and
`SilenceErrors: true` (`signal_stop.go:43-44`) because the command
is machine-consumed — Claude Code parses stdout as JSON and any
stray usage/error text would poison the response.

### `stop` — Stop-hook handler

Called by Claude Code at every turn boundary. Two stdout outputs are
possible (JSON-encoded on a single line):

```json
{"decision":"approve"}
{"decision":"block","reason":"<message>"}
```

The shape is defined at `signal_stop.go:19-22` as `stopHookResponse`:

```go
type stopHookResponse struct {
    Decision string `json:"decision"`         // "block" or "approve"
    Reason   string `json:"reason,omitempty"` // Message to inject when blocking
}
```

**Budget:** the Long help at `signal_stop.go:38` states
"This command must complete in <500ms as it runs on every turn
boundary."

**Logic** (`runSignalStop`, `signal_stop.go:47-108`):

1. Resolve the current agent address via `detectSender()`.
   - If empty or equal to `"overseer"`, emit approve and exit
     (`signal_stop.go:49-53`). Non-agent sessions never block.
2. Find the town root via `workspace.FindFromCwd`. If not in a
   Gas Town workspace, approve and exit (`signal_stop.go:56-60`).
3. Run two checks **in parallel** via a `sync.WaitGroup`
   (`signal_stop.go:64-78`) to stay under the 500ms budget:
   - `checkUnreadMail` — uses `mail.NewRouterWithTownRoot` and
     `mailbox.ListUnread` to list unread mail, filters out self-
     handoff mail (`isSelfHandoff` at `signal_stop.go:150-155`),
     and builds a human-readable block reason naming the count and
     the most recent sender/subject (`signal_stop.go:111-147`).
   - `checkStopSlungWork` — uses `GetRole()` and
     `buildAgentBeadID` to compute the agent bead ID, fast-paths
     via the agent bead's `HookBead` field, and falls back to a
     `beads.List(Status=hooked, Assignee=identity)` query if the
     agent bead doesn't exist yet (`signal_stop.go:158-206`).
     Only blocks if the hooked bead's `Status` is
     `beads.StatusHooked` (not yet claimed).
4. **Mail wins over work** (`signal_stop.go:82-86`). If both
   checks returned a reason, the mail reason is used.
5. If no reason is produced, call `clearStopState(statePath)` and
   approve (`signal_stop.go:90-94`).
6. Otherwise, load the per-agent state file from OS temp
   (`stopStateFilePath` at `signal_stop.go:219-222`):
   `os.TempDir() + "/gt-signal-stop-<addr>.json"` (with `/`
   replaced by `_`). If the last saved reason equals the current
   reason, **approve** instead of blocking — this is the explicit
   infinite-loop guard at `signal_stop.go:96-103`: "prevents
   infinite loops where the same block reason fires at every turn
   boundary, consuming the entire agent context."
7. New or changed condition: save the reason to the state file
   and block (`signal_stop.go:105-107`).

### Failure mode: approve, don't crash

`outputStopResponse` (`signal_stop.go:262-270`) has a fallback: if
`json.Marshal` fails for any reason, it writes a literal
`{"decision":"approve"}` to stdout and returns nil. The command
never fails loudly — failing loudly here would crash the agent's
turn boundary, and that is worse than missing a notification.

### State file shape

`stopState` (`signal_stop.go:213-215`):

```go
type stopState struct {
    LastReason string `json:"last_reason"`
}
```

Stored at `os.TempDir()/gt-signal-stop-<safe-address>.json`. Mode
`0600`. Cleared (via `os.Remove`) whenever no block reason fires
— see `clearStopState` at `signal_stop.go:247-249`.

### Flags

None.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [hook.md](hook.md) — `gt hook` is what an agent runs to see the
  slung work that `signal stop` blocks on. The block message at
  `signal_stop.go:181-183` explicitly says `"Run gt hook to see
  details, then execute the work."`
- [mayor.md](mayor.md) — Mayor is an agent that receives mail; its
  unread mail would trigger a stop block on its next turn boundary.
- [deacon.md](deacon.md) — same.
- [dog.md](dog.md) — dogs also have agent addresses and go through
  the same stop check.
- [callbacks.md](callbacks.md) — unrelated hook runner but shares
  the "called by external scheduler" shape.
- [hooks.md](hooks.md) — the install/scaffold command for Claude
  Code hook wiring, the other side of the integration.
- [nudge.md](nudge.md) — alternative inbound channel (direct tmux
  send-keys); `signal stop` does not fire for nudges because
  nudges land as Claude input, not as mail or hooked beads.

## Notes / open questions

- **`detectSender` and `buildAgentBeadID` are shared helpers** used
  elsewhere in `internal/cmd/`. Their exact definitions are not on
  this page — resolving them would be a follow-up if drift-hunting
  the agent-address resolution chain.
- **The 500ms budget is a comment, not an enforcement.** No
  `context.WithTimeout` or `time.AfterFunc` wraps the checks. Two
  `bd` shell-outs in parallel can and do exceed 500ms under Dolt
  load — the cost is a slower turn boundary, not a failed hook.
- **Only `self-handoff` mail is filtered.** Any other mail from
  the agent to itself (non-handoff subjects) still triggers a
  block. Unclear if that is intentional.
- **State file survives reboots.** `os.TempDir()` on Linux is
  usually `/tmp`, which is tmpfs on most distros but persistent on
  others. Operators restarting a machine mid-session could see a
  stale block reason from before the reboot.
- **No `start`, `status`, or other sibling subcommands** — the
  parent `signalCmd` is a group placeholder with room to grow. If
  other Claude Code hooks (e.g. `PreToolUse`, `PostToolUse`) ever
  need a `gt`-backed handler, they would land here.
