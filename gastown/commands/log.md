---
title: gt log
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/log.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, townlog, agent-lifecycle, tmux-hook]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [precondition]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt log

Views the centralized town lifecycle log (`<townRoot>/logs/town.log`)
with filtering, tailing, and an optional `crash` subcommand called
from a tmux pane-died hook. Also provides package-level helpers that
the rest of `internal/cmd/` uses to log events from anywhere in the
codebase.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/log.go:31-54`)
**Beads-exempt:** no (not in `beadsExemptCommands`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/log.go:31-474`.

### Invocation

```
gt log [-n N] [-t TYPE] [-a AGENT] [--since DUR] [-f] [--acp]
gt log crash --agent <addr> [--session NAME] [--exit-code N]
```

### Behavior

#### Top-level `log` command

`runLog` (`log.go:91-164`):

1. Requires a Gas Town workspace — `workspace.FindFromCwdOrError`
   (`log.go:92-95`).
2. If `--acp` is set, delegates to `viewACPLogs` (`log.go:97-100`)
   which reads `<townRoot>/logs/acp.log`. Requires `GT_ACP_DEBUG=1`
   to have produced any content.
3. Otherwise points at `<townRoot>/logs/town.log` (`log.go:102`).
4. If `--follow`/`-f`, delegates to `followLog` (`log.go:105-107`),
   which shells out to `tail -f <logPath>` (`log.go:167-186`) after
   ensuring the log directory and file exist.
5. Otherwise reads all events via `townlog.ReadEvents(townRoot)`
   (`log.go:116-119`) — the same helper [audit](audit.md) uses in
   its `collectTownlogEvents` collector.
6. Builds a `townlog.Filter` from `--type`, `--agent`, and
   `--since` (`log.go:126-143`). `--since` uses `time.ParseDuration`;
   no `d` (days) suffix support (unlike [audit](audit.md)).
7. Applies the filter via `townlog.FilterEvents`
   (`log.go:146`), then tails to the last `-n` entries
   (`log.go:149-151`; default 20).
8. Prints each event via `printEvent` (`log.go:159-161`, defined at
   `log.go:227-267`), which color-codes by event type and formats a
   detail string via `formatEventDetail` (`log.go:270-348`).

Recognized event types (`log.go:232-260`, from the `internal/townlog`
package constants):

- Lifecycle: `spawn`, `wake`, `nudge`, `handoff`,
  `handoff-NOPERSIST`, `done`, `crash`, `kill`, `callback`
- Patrol: `patrol_started`, `polecat_checked`, `polecat_nudged`,
  `escalation_sent`, `patrol_complete`

#### `log crash` — tmux pane-died hook

`logCrashCmd` (`log.go:56-71`) runs `runLogCrash` (`log.go:358-404`):

1. Resolves town root via `workspace.FindFromCwd`, falling back to
   `$HOME/gt` if the cwd path doesn't match — the comment at
   `log.go:361-371` notes that tmux hooks may not have proper cwd.
2. Maps `--exit-code` to event type (`log.go:373-395`):
   - `0` → `townlog.EventDone`, context `"exited normally"`
   - `130` → `townlog.EventKill`, context `"interrupted (exit 130)"`
     (SIGINT / Ctrl+C)
   - any other non-zero → `townlog.EventCrash`, context `"exit code N"`
     plus session name when provided
3. Calls `townlog.NewLogger(townRoot).Log(eventType, crashAgent, context)`
   (`log.go:398-401`).

`--agent` is required (`log.go:85`); `--session` and `--exit-code`
are optional flags.

### Package-level logging helpers

`log.go:406-473` exposes helpers used by other commands in the
package, not by end users. They all wrap
`townlog.NewLogger(townRoot).Log(...)`:

- `LogEvent(eventType, agent, context)` — finds town root from cwd
- `LogEventWithRoot(townRoot, eventType, agent, context)`
- `LogSpawn`, `LogWake`, `LogNudge`, `LogHandoff`, `LogHandoffNoPersist`,
  `LogDone`, `LogCrash`, `LogKill` — typed convenience wrappers

`LogHandoffNoPersist` (`log.go:452-458`) is the specific marker for
handoffs that attempted to persist to Dolt and failed — crash recovery
can distinguish it from a successful handoff.

### Subcommands

- `crash` (`log.go:56-71`, registered at `log.go:87`) — called by tmux
  pane-died hooks, not typically run manually.

### Flags (top-level `log`)

Defined in `init()` (`log.go:74-79`):

| Flag                  | Type   | Default | Purpose                                                             |
|-----------------------|--------|---------|---------------------------------------------------------------------|
| `--tail` / `-n`       | int    | `20`    | Number of events to show                                            |
| `--type` / `-t`       | string | `""`    | Filter by event type (spawn, wake, nudge, handoff, done, crash, kill) |
| `--agent` / `-a`      | string | `""`    | Filter by agent prefix                                              |
| `--since`             | string | `""`    | Events since duration (Go `time.ParseDuration` format)              |
| `--follow` / `-f`     | bool   | `false` | Tail the log via `tail -f`                                          |
| `--acp`               | bool   | `false` | View `logs/acp.log` instead (requires `GT_ACP_DEBUG=1`)             |

### Flags (`log crash`)

`log.go:82-85`:

| Flag           | Type   | Default | Purpose                                     |
|----------------|--------|---------|---------------------------------------------|
| `--agent`      | string | `""`    | Agent ID (required)                         |
| `--session`    | string | `""`    | Tmux session name                           |
| `--exit-code`  | int    | `-1`    | Exit code from the tmux pane                |

## Related

- [audit](audit.md) — `collectTownlogEvents` (`audit.go:337-366`)
  reuses `townlog.ReadEvents` to pull the same events into a wider
  cross-source timeline.
- [feed](feed.md) — live TUI view of events; feed's sources
  (`.events.jsonl`, `bd activity`, MQ events) are distinct from
  `town.log`.
- [activity](activity.md) — writes to `.events.jsonl`; `log` reads
  `town.log`. Two separate JSONL streams.
- [heartbeat](heartbeat.md) — agent self-reports state; the witness
  may convert heartbeat transitions into townlog events.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Failure modes

### Precondition violations (what does it assume?)
- **`log crash` falls back to `$HOME/gt`:** `runLogCrash` at
  `log.go:361-370` assumes `~/gt` is the town root when
  `workspace.FindFromCwd` fails. If the town root is at a
  non-conventional path, the crash event is either lost or written to
  the wrong workspace. **Absent** — no validation that the fallback
  directory is actually a Gas Town workspace beyond checking for
  `$HOME/gt/mayor`.
- **`followLog` creates empty files:** `followLog` at
  `log.go:169-176` creates the logs directory and an empty log file
  if neither exists, then tails it. This silently creates filesystem
  artifacts in the workspace. **Absent** — no warning that the file
  was created.
- **`tail -f` dependency:** `followLog` at `log.go:181` shells out to
  the external `tail` binary. On minimal containers without coreutils,
  this fails. **Absent** — no fallback or check for `tail`.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `tail` | `-f` | `<logPath>` | runtime (resolved log file path) | `log.go:181` |

## Notes / open questions

- `--since` uses Go's `time.ParseDuration` only — no `d` suffix like
  [audit](audit.md)'s `parseDuration`. `"1d"` will fail here but work
  in `audit`.
- The fallback to `$HOME/gt` in `runLogCrash` (`log.go:362-370`) is
  brittle if the town root is at a non-conventional location. Tmux
  hooks that forget to `cd` before running `gt log crash` will miss
  the correct town.
- Exit code 130 (Ctrl+C / SIGINT) is mapped to `kill`, not `crash`
  (`log.go:383-386`). Other signal-style exits (137 for SIGKILL, 143
  for SIGTERM) hit the generic "crash" branch.
- `viewACPLogs` (`log.go:189-224`) does its own tail-by-line on the
  ACP log because ACP log lines aren't townlog events.
- `followLog` shells out to the external `tail` binary rather than
  using a Go file watcher — a simple, portable choice.
