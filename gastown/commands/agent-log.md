---
title: gt agent-log
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/agent_log.go
tags: [command, ungrouped, hidden, telemetry, otlp, agent, session-lifecycle]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt agent-log

Hidden internal daemon that streams an agent's conversation events to the
OTLP log endpoint for the lifetime of a Gas Town tmux session.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** yes (`Hidden: true` on `agent_log.go:25`)
**Polecat-safe:** no
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/agent_log.go:22-93`.

### Invocation

```
gt agent-log --session <tmux-session> --work-dir <path> \
             [--agent <type>] [--since <RFC3339>] [--run-id <GT_RUN>]
```

`Use: "agent-log"` (`agent_log.go:23`). `Short` says "Stream agent
conversation events to OTLP log endpoint (invoked by session lifecycle)"
(`agent_log.go:24`). No `Long` is set — this is an internal helper, not a
user-facing command.

### Flags

Declared in `init()` on `agent_log.go:29-37`:

| flag | type | default | required | description |
|------|------|---------|----------|-------------|
| `--session` | string | `""` | yes | Gas Town tmux session name (used as log tag) |
| `--work-dir` | string | `""` | yes | Agent working directory (used to locate conversation log files) |
| `--agent` | string | `"claudecode"` | no | Agent type (`claudecode`, `opencode`) |
| `--since` | string | `""` | no | RFC3339 timestamp — only watch JSONL files modified at or after this time |
| `--run-id` | string | `""` | no | GASTA run identifier (GT_RUN); falls back to `$GT_RUN` |

`MarkFlagRequired` is called for `--session` and `--work-dir`
(`agent_log.go:35-36`).

### Behavior

`runAgentLog` on `agent_log.go:40-93`:

1. **Seed run ID into context** (`agent_log.go:44-48`): if `--run-id` is
   non-empty, call `telemetry.WithRunID(ctx, agentLogRunID)`; else if
   `$GT_RUN` is set, use that. This ensures every emitted `agent.event`
   carries `run.id` for waterfall correlation in the OTLP backend.
2. **Initialize telemetry** via `telemetry.Init(ctx, "gastown", "")`
   (`agent_log.go:50-60`). On failure, prints a warning to stderr but
   continues. Shuts the provider down with a 5s timeout on exit.
3. **Parse `--since`** as RFC3339 (`agent_log.go:66-72`). When set by the
   session-lifecycle caller, this is approximately the GT session start
   time, ensuring the watcher only observes Claude instances spawned by
   this Gas Town session (not pre-existing user sessions or other Gas Town
   rigs sharing the same work directory).
4. **Create adapter** for the agent type via
   `agentlog.NewAdapter(agentLogAgentType)` (`agent_log.go:74-77`).
   Returns an error like `"unknown agent type %q; supported: claudecode,
   opencode"` when the type is unrecognized.
5. **Start the watcher** — `adapter.Watch(ctx, session, workDir, since)`
   returns a channel of events (`agent_log.go:79-82`).
6. **Event loop** (`agent_log.go:84-91`): for each event received:
   - If `EventType == "usage"`, call
     `telemetry.RecordAgentTokenUsage(ctx, sessionID, nativeSessionID,
     input, output, cacheRead, cacheCreation)`.
   - Otherwise, call `telemetry.RecordAgentEvent(ctx, sessionID, agentType,
     eventType, role, content, nativeSessionID, timestamp)`.
7. Returns nil when the watcher channel closes.

### Who invokes it

The `Short` text says "invoked by session lifecycle." This is a long-lived
background process started alongside a crew/polecat tmux session and torn
down when the session dies. Users shouldn't run it directly; it's
`Hidden: true` to keep it out of `gt --help`.

### Related commands

- [feed](feed.md), [trail](trail.md), [audit](audit.md), [log](log.md) —
  user-facing commands that view/query the events this daemon writes.
- [crew](crew.md), [polecat](polecat.md), [rig](rig.md) — session
  lifecycle commands that would be the likely callers (the files that
  launch tmux sessions) — the source file says "invoked by session
  lifecycle."
- [metrics](metrics.md) — runs on the consumer side of the OTLP pipeline.
- [costs](costs.md) — aggregates the token-usage events this command
  emits.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Session lifecycle caller.** Where exactly is this command launched
  from? Likely inside `crew start` / `polecat start` handlers via an
  `activateAgentLogging` helper mentioned in the `--since` comment
  (`agent_log.go:62-65`). Worth grepping for that symbol when mapping the
  crew/polecat lifecycle.
- **Fan-out vs. per-session.** One `gt agent-log` per tmux session means
  every running agent has a long-lived `gt` process attached to it. The
  telemetry provider is initialized per-process, which multiplies
  OTLP-export overhead. Worth measuring if session counts grow.
- **Two record functions.** `RecordAgentTokenUsage` vs. `RecordAgentEvent`
  diverge on schema — usage events carry token counts, normal events
  carry role/content. Both funnel into the `agent.event` OTLP log family.
