---
title: gt nudge
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/nudge.go
  - /home/kimberly/repos/gastown/internal/cmd/nudge_poller.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, nudge, tmux, delivery-modes, polecat-safe]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt nudge

Send a synchronous message to any Gas Town worker's Claude Code
session — the write half of the `nudge` / [peek](peek.md) pair,
and the universal agent-to-agent messaging primitive.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`nudge.go:73`)
**Polecat-safe:** yes (`Annotations: map[string]string{
AnnotationPolecatSafe: "true"}` at `nudge.go:74`)
**Beads-exempt:** yes — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:56`
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/nudge.go:71-598`,
plus the sibling `nudge_poller.go` (not read for this page).

### Invocation

```
gt nudge <target> [message] [flags]
```

`Args: cobra.RangeArgs(1, 2)`. The message comes from the second
positional arg, `-m/--message`, or `--stdin` (mutually exclusive
with `-m`).

### Address forms

Three shapes, tried in order in `runNudge`:

1. **Channel** (`nudge.go:426-429`) — `channel:<name>`. Resolves
   to member patterns in
   `~/gt/config/messaging.json → nudge_channels`, then fans out
   via `runNudgeChannel` at `nudge.go:602-648`.
2. **Role shortcut** (`nudge.go:453-471`) — `mayor`, `witness`,
   `refinery`, `deacon`. Mapped through
   `session.MayorSessionName()` / `WitnessSessionName(prefix)` /
   `RefinerySessionName(prefix)` / `DeaconSessionName()`. `witness`
   and `refinery` need the caller's rig context; errors if
   not in a rig.
3. **`rig/polecat` or raw session name** (`nudge.go:503-568`) —
   parsed with `parseAddress`; `crew/<name>` and
   `polecats/<name>` sub-prefixes are recognized; short
   addresses probe for a crew session first then fall back to
   the polecat session manager (mirrors the mail system's
   `addressToSessionIDs`).

### Delivery modes (`--mode`)

Defined at `nudge.go:49-59`:

| mode | description |
|------|-------------|
| `immediate` | Direct tmux `send-keys`. Interrupts in-flight work; guarantees delivery. |
| `queue` | Writes to a file queue; the recipient's `UserPromptSubmit` hook drains it at the next turn boundary. Zero interruption. |
| `wait-idle` | **Default.** Polls for idle state via `t.WaitForIdle` (prompt visible). On idle → direct delivery. On busy-timeout → queue + synchronous `watchAndDeliver` poll over a longer window. On terminal errors (session gone) → error, don't queue. |

`deliverNudge` (`nudge.go:157-279`) contains the mode dispatch.
Key details worth preserving:

- **ACP sessions are force-queued** (`nudge.go:162-165`). ACP
  agents have no tmux pane to `send-keys` to, so even if the
  caller asked for `immediate` or `wait-idle`, the nudge goes
  via the queue. Currently only the Mayor supports ACP
  (`nudge.go:26-37` `hasACPSessionByName`).
- **Non-Claude agents degrade to queue mode**
  (`nudge.go:194-221`): `wait-idle` looks up the target's
  `GT_AGENT` env var and the corresponding `config.AgentPreset`.
  If the preset has no `ReadyPromptPrefix` (Gemini, Codex, etc.),
  `WaitForIdle` can't detect idle and would produce false
  positives, so delivery drops to `queue` with a stderr warning.
  GH#gt-5ey3 is cited in the comment.
- **Gemini escape-cancel avoidance** (`nudge.go:272-276`):
  `immediate` mode consults the agent preset's
  `EscapeCancelsRequest` flag; if set, the tmux nudge opts pass
  `SkipEscape: true` so the leading `Escape` keystroke doesn't
  cancel in-flight generation. GH#gt-wasn.
- **Wait-idle watcher** (`nudge.go:296-332` `watchAndDeliver`) —
  synchronous, runs after queueing in `wait-idle` mode. Polls
  every `idleWatcherPollInterval` (1s) up to `idleWatcherTimeout`
  (60s). Exits on: delivery, queue empty (race), session
  disappears, or timeout. Must be synchronous because `gt nudge`
  is a CLI and goroutines die at process exit.
- **Fall-through immediate fallback** (`nudge.go:241-257`) —
  if queue enqueue fails, `wait-idle` falls back to
  `FormatForInjection` + direct `NudgeSession`. Prints a warning.
  The rationale: "Better to interrupt than lose the message
  entirely."
- **Sender attribution** (`nudge.go:167-170`) — direct
  (immediate) delivery gets an explicit `[from <sender>]`
  prefix. Queue-based delivery does **not** double-prefix
  because `FormatForInjection` adds the prefix when the hook
  drains.

### Message injection format

Queue-drained and idle-delivered messages are wrapped by
`nudge.FormatForInjection` as a `<system-reminder>` block
(`nudge.go:224-233`), so the target agent processes them as
background notifications rather than user interruptions or
corrections.

### `--if-fresh` (compaction suppression)

`nudge.go:365-378`: when set, `runNudge` fetches the caller's
tmux session creation time (`GetSessionCreatedUnix`). If the
session is older than `ifFreshMaxAge` (60s), the nudge is
silently dropped. Purpose per the comment at `nudge.go:133-135`:
"Sessions older than this are considered compaction/clear
restarts, not new sessions" — so compaction-triggered
`SessionStart` hooks don't spam the deacon.

### DND / force

`nudge.go:440-447`: unless `--force`, nudge calls
`shouldNudgeTarget(townRoot, target, nudgeForceFlag)`. If the
target is muted, prints a dim skip message and returns cleanly
(exit 0, not an error). `--force` overrides. See
[dnd](dnd.md) / [notify](notify.md) for the level state that's
consulted.

### Subcommands

None.

### Flags

Defined at `nudge.go:62-69`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--message` | `-m` | string | `""` | Message to send |
| `--force` | `-f` | bool | `false` | Send even if target has DND enabled |
| `--stdin` | — | bool | `false` | Read message from stdin (avoids shell quoting issues) |
| `--if-fresh` | — | bool | `false` | Only send if caller's tmux session is <60s old (compaction-nudge suppression) |
| `--mode` | — | string | `wait-idle` | Delivery mode: `wait-idle`, `queue`, `immediate` |
| `--priority` | — | string | `normal` | Queue priority: `normal` or `urgent` |

Mode and priority are validated eagerly at `nudge.go:356-361`.

### Telemetry & event logging

- `telemetry.RecordNudge(ctx, target, err)` deferred at
  `nudge.go:347-354` — every nudge records a telemetry entry
  including failures.
- `LogNudge(townRoot, target, message)` on successful delivery
  (e.g. `nudge.go:497`, `565`, `591`).
- `events.LogFeed(events.TypeNudge, sender, …)` emits a feed
  event consumable by [feed](feed.md) (e.g. `nudge.go:499`, `568`,
  `594`).

### Related commands

- [peek](peek.md) — read-side partner. "`nudge` writes, `peek`
  reads."
- [mail](mail.md) — durable alternative. Nudges are ephemeral
  (no bead, no Dolt commit); mail is the permanent audit trail.
  See the project CLAUDE.md mail-protocol note: "Default to
  nudge for routine agent-to-agent communication. Only use mail
  when the message MUST survive the recipient's session death."
- [broadcast](broadcast.md) — fan-out shortcut that uses the
  same DND check but always the immediate-delivery path (no
  `--mode` flag). Lacks `--force`.
- [dnd](dnd.md) / [notify](notify.md) — target state consulted
  before delivery.
- [session](session.md) — underlying session management; `nudge`
  uses `session.MayorSessionName` / `WitnessSessionName` / etc.
- [feed](feed.md) — consumes the feed events that every nudge
  emits.
- [handoff](handoff.md) — a related communication primitive;
  handoff composes mail (not nudge), but the handoff cooldown
  machinery was introduced to prevent crash loops from nudges-
  into-handoffs-into-respawn cycles.
- [callbacks](callbacks.md) — overlapping messaging primitive.

## Notes / open questions

- **The 1.5K-line file.** `runNudge` is the tip of a long tail
  including `runNudgeChannel`, pattern resolution, ACP handling,
  and `watchAndDeliver`. This page covers the cobra surface and
  the core delivery flow; the `nudge_poller.go` sibling is not
  mapped, but the [`internal/nudge`](../packages/nudge.md) package is.
- **Queue durability.** `nudge.Enqueue` writes a file; `Drain`
  uses rename-based atomic claiming (`nudge.go:317-319`
  comment). If two watchers race, only one wins — the losers
  get an empty slice and skip delivery to avoid duplicates.
  The file queue is on disk in the town root.
- **Synchronous watcher is a process.** `gt nudge` in
  `wait-idle` mode can block up to 15s (`waitIdleTimeout`) +
  60s (`idleWatcherTimeout`) = up to ~75s worst case before
  returning. Callers that want fire-and-forget should use
  `--mode=queue` or `--mode=immediate`.
- **`--if-fresh`'s cutoff is a magic 60s.** Hard-coded as
  `ifFreshMaxAge` at `nudge.go:135`. If session startup is
  slow (e.g. under load), a nudge that's actually from a new
  session might be dropped. Trade-off noted.
- **Channel members are patterns**
  (`nudge_channels` in `messaging.json`) expanded via
  `resolveNudgePattern`. The expansion happens at nudge time
  against the current live session list, so a channel like
  `gastown/polecats/*` always hits the currently-running
  polecats — worth remembering when reasoning about
  reproducibility.
- **No rate limiting per sender.** Unlike
  [broadcast](broadcast.md)'s 100ms inter-send pacing, `nudge`
  to a single target has no delay. A script that loops over
  agents calling `gt nudge` could hammer tmux; prefer
  [broadcast](broadcast.md) for that shape.
