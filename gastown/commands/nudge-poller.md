---
title: gt nudge-poller
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/nudge_poller.go
tags: [command, ungrouped, hidden, nudge, poller, background, non-claude]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: internal
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt nudge-poller

Hidden background daemon that polls the nudge queue for a tmux session
and drains it into the agent when idle — the non-Claude equivalent of
Claude Code's `UserPromptSubmit` hook drain.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** yes (`Hidden: true` on `nudge_poller.go:31`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/nudge_poller.go:28-140`.

### Invocation

```
gt nudge-poller <session> [--interval <dur>] [--idle-timeout <dur>]
```

`Use: "nudge-poller <session>"` (`nudge_poller.go:29`).
`Args: cobra.ExactArgs(1)` (`nudge_poller.go:44`). Exactly one positional:
the target tmux session name.

### Flags

Declared in `init()` on `nudge_poller.go:22-26`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--interval` | string | `nudge.DefaultPollInterval` | Poll interval (e.g., `10s`, `30s`) |
| `--idle-timeout` | string | `nudge.DefaultIdleTimeout` | How long to wait for agent idle before skipping |

Both are parsed via `time.ParseDuration` inside `runNudgePoller`
(`nudge_poller.go:56-64`).

### Why it exists

From the `Long` help on `nudge_poller.go:32-43`:

> Polls the nudge queue for a tmux session and drains it when the agent
> is idle. This is the background equivalent of Claude's
> UserPromptSubmit hook drain — it ensures queued nudges are delivered
> to agents that lack turn-boundary hooks (Gemini, Codex, Cursor, etc.).
>
> This command runs as a long-lived background process. It exits when:
>   - The target tmux session dies
>   - It receives SIGTERM (from StopPoller or session teardown)
>   - The poll loop encounters an unrecoverable error
>
> Normally launched automatically by 'gt crew start' for non-Claude
> agents. Not intended for direct user invocation.

Claude Code's `UserPromptSubmit` hook fires at the start of every user
turn, which is where `gt nudge --drain` normally runs. Agents without
that hook (Gemini CLI, Codex, Cursor, …) need an external poller — this
command.

### Behavior

`runNudgePoller` on `nudge_poller.go:48-136`:

1. **Find town root** via `workspace.FindFromCwdOrError()`
   (`nudge_poller.go:51-54`).
2. **Parse intervals** (`nudge_poller.go:56-64`).
3. **Check session exists** via `tmux.HasSession(sessionName)`
   (`nudge_poller.go:68-71`). Returns `session %q not found` if absent.
4. **Resolve agent config once at startup** (`nudge_poller.go:73-87`):
   reads the session's `GT_AGENT` environment variable, looks up a
   preset via `config.GetAgentPresetByName(name)`, and extracts two
   booleans:
   - `hasPromptDetection = preset.ReadyPromptPrefix != ""` — does this
     runtime let us detect "ready for input" via `WaitForIdle`?
   - `nudgeOpts.SkipEscape = preset.EscapeCancelsRequest` — does Escape
     cancel in-flight generation? (Gemini CLI does; GH issue `gt-wasn`.)
5. **Install SIGTERM/SIGINT handler** for graceful shutdown
   (`nudge_poller.go:90-91`).
6. **Poll loop** (`nudge_poller.go:93-135`): a `time.Ticker` fires every
   `pollInterval`. On each tick:
   - Return nil if the session disappeared (`nudge_poller.go:102-105`).
   - Skip the tick if `nudge.Pending(townRoot, sessionName)` returns 0
     (`nudge_poller.go:107-110`).
   - If the runtime has prompt detection, wait for idle via
     `t.WaitForIdle(sessionName, idleTimeout)`. If that returns an error
     (timeout) and `hasPromptDetection` is true, skip this tick — the
     helper `shouldSkipDrainUntilIdle` on `nudge_poller.go:138-140` is:
     ```go
     return hasPromptDetection && waitErr != nil
     ```
     Runtimes without prompt detection fall back to best-effort
     delivery on every poll interval.
   - Drain the queue via `nudge.Drain(townRoot, sessionName)`
     (`nudge_poller.go:121-128`). If another caller already drained it
     (empty result), skip.
   - Format with `nudge.FormatForInjection` and inject via
     `t.NudgeSessionWithOpts(sessionName, formatted, nudgeOpts)`
     (`nudge_poller.go:130-133`). The `SkipEscape` option prevents the
     tmux helper from sending an Escape keystroke that would cancel
     Gemini's in-flight response.

### Exit conditions

- Target tmux session disappears → clean exit (nil).
- SIGTERM / SIGINT → clean exit (nil).
- Unrecoverable errors in inner helpers are logged to stderr and the
  loop continues — most errors are not fatal.

### Subcommands

None.

## Related commands

- [nudge](nudge.md) — the user-facing command this poller drains. The
  two sides of the same queue.
- [crew](crew.md) — `gt crew start` is the normal launcher per the
  `Long` help.
- [callbacks](callbacks.md) — a different turn-boundary mechanism; worth
  comparing how each hooks into agent runtimes.
- [prime](prime.md) — drains on session start; this command handles the
  steady-state case after prime finishes.
- [signal](signal.md) — related injection primitive.
- [agent-log](agent-log.md) — another hidden session-lifecycle daemon
  launched alongside non-Claude agents.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Background process that drains the nudge queue. Runs in a loop with error handling per-drain cycle. Session-death detection exits cleanly.

## Troubleshooting

For diagnostic workflows involving this command, see
[Investigating: message delivery](../workflows/investigations/message-delivery.md).

## Notes / open questions

- **Who starts it?** The `Long` help says `gt crew start`, but polecats
  also use non-Claude runtimes. Worth grepping for `nudge-poller` call
  sites (probably in `crew.go`, `polecat.go`, `rig.go`, or a shared
  session-bringup helper). One poller per session is implied.
- **Preset cache.** `GetAgentPresetByName` is called once at startup —
  so if the user changes the preset mid-session, the poller will keep
  using the stale `SkipEscape` and `ReadyPromptPrefix` values.
  Restart-to-reload is the workaround.
- **Race with `gt nudge --drain`.** Both paths drain the same file. The
  `len(drained) == 0` check at `nudge_poller.go:127-128` handles the
  race benignly.
- **`shouldSkipDrainUntilIdle` is factored out** (`nudge_poller.go:138-140`)
  but only called once — likely for testing. Matches the pattern from
  elsewhere in the codebase where trivial predicates get named so tests
  can assert against them.
- **No `GH#gt-wasn` in beads today.** The reference on `nudge_poller.go:75`
  may be a dead or closed issue — worth checking against the current
  bd state.
