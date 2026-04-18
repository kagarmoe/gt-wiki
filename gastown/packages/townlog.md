---
title: internal/townlog
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/townlog/logger.go
tags: [package, townlog, logging, agent-lifecycle, witness, events]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/townlog

Human-readable agent-lifecycle log writer for `<townRoot>/logs/town.log`.
Gas Town uses this to record "agent X spawned / was nudged / handed off
/ crashed / died" as one line per event, timestamped and bracketed by
event type, in plain text rather than JSON. Separate from the structured
machine-readable `.events.jsonl` stream owned by
[`internal/events`](events.md): `events.jsonl` is the source of truth
for the feed and telemetry pipeline; `town.log` is what a human `tail
-f`s to watch agents come and go.

**Go package path:** `github.com/steveyegge/gastown/internal/townlog`
**File count:** 1 go file (`logger.go`), 1 test file.
**Imports (notable):** stdlib only (`fmt`, `os`, `path/filepath`,
`sync`, `time`).
**Imported by (notable):** [`gt`](../binaries/gt.md)'s session-spawn
and nudge paths, the witness patrol, and anywhere an agent lifecycle
transition is announced in human-readable form.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/townlog/logger.go`.

### Event catalog (`logger.go:12-46`)

`EventType` is a string alias. The constants fall into three groups:

- **Agent lifecycle**: `EventSpawn`, `EventWake`, `EventNudge`,
  `EventHandoff`, `EventDone`, `EventCrash`, `EventKill`.
- **Handoff-persistence failure**: `EventHandoffNoPersist =
  "handoff-NOPERSIST"` — distinct from `EventHandoff` so crash recovery
  can identify false handoffs where the Dolt write didn't land.
- **Callbacks / witness patrol**: `EventCallback`, `EventPatrolStarted`,
  `EventPolecatChecked`, `EventPolecatNudged`, `EventEscalationSent`,
  `EventPatrolComplete`.
- **Session death** (for crash investigation): `EventSessionDeath`
  (single session terminated with a reason), `EventMassDeath` (multiple
  sessions died in a short window — the mass-death alarm).

### `Event` and `Logger`

- `Event` struct (`logger.go:49-54`): `{Timestamp, Type, Agent,
  Context}`. `Agent` is the fully-qualified identity (e.g.
  `"gastown/crew/max"`); `Context` is free-form detail.
- `Logger` struct (`logger.go:57-60`): holds the log path and a
  `sync.Mutex`. Goroutine-safe writes.
- `NewLogger(townRoot string) *Logger` (`logger.go:73-77`) constructs
  a logger pointing at `<townRoot>/logs/town.log`. Path helpers
  `logDir` and `logPath` are private (`logger.go:63-70`).

### Write path

- `LogEvent(event Event) error` (`logger.go:80-103`):
  1. Acquires the mutex.
  2. `os.MkdirAll` for the log directory (0755).
  3. `os.OpenFile` with `O_APPEND|O_CREATE|O_WRONLY`, perms 0600.
  4. `formatLogLine(event)` → write + newline.
  5. Deferred `f.Close()`.
- `Log(eventType, agent, context)` (`logger.go:106-113`) — convenience
  that stamps `time.Now()` and forwards to `LogEvent`.

### `formatLogLine` (`logger.go:117-223`)

Produces one line per event in the exact format:

```
2006-01-02 15:04:05 [spawn] gastown/crew/max spawned for gt-xyz
```

A big switch on `EventType` selects the `detail` phrasing and how
`Context` is folded in. Examples:

- `EventSpawn` → `"spawned for <context>"` or just `"spawned"`.
- `EventNudge` → `"nudged with <truncated-context>"` (via
  `truncate(e.Context, 50)`).
- `EventHandoffNoPersist` → `"handoff FAILED (Dolt persistence) (<ctx>)"`
  — the loudest format in the table, matching the severity of a
  handoff that didn't write.
- `EventMassDeath` → `"MASS SESSION DEATH (<ctx>)"` — all caps for
  log-scanning visibility.

Unknown event types fall through to `<type>` (optionally `"(<ctx>)"`).

`truncate(s, maxLen)` (`logger.go:226-231`) is the standard ellipsis
helper used for the nudge context cap.

### Read path

- `ReadEvents(townRoot) ([]Event, error)` (`logger.go:235-247`) —
  `os.ReadFile` the log, then `ParseLogLines`. Non-existent log file
  is a nil-nil return, not an error.
- `ParseLogLines(content) ([]Event, error)` (`logger.go:251-267`) —
  splits on newlines via the private `splitLines`, parses each with
  `parseLogLine`, skips malformed lines.
- `parseLogLine(line) (Event, error)` (`logger.go:271-326`) is the
  inverse of `formatLogLine` — but only partially. It parses:
  - The 19-char timestamp with `time.Parse("2006-01-02 15:04:05",
    ...)`.
  - The `[event-type]` bracketed tag.
  - The first space-delimited token after the bracket as `Agent`.
  - The rest of the line (the detail) is **not** round-tripped back
    into `Context` — the function leaves `Context` empty. The comment
    at `logger.go:322` is explicit: "The rest is context info (not
    worth parsing further)".
- `splitLines(s)` (`logger.go:328-341`) — custom splitter that keeps
  the trailing line if it doesn't end in a newline.
- `TailEvents(townRoot, n)` (`logger.go:344-353`) — read, return the
  last `n`.
- `FilterEvents(events, Filter)` (`logger.go:362-378`) — filters by
  `Type`, by `Agent` prefix (`hasPrefix` is a local byte-wise helper,
  `logger.go:380-385`), and/or by `Since` time.

## Related wiki pages

- [internal/events](events.md) — the structured `.events.jsonl`
  stream. `events.Event` is a richer, payload-carrying record; the
  feed curator and OTEL pipeline read that file. `townlog`'s output
  is human-oriented and strictly line-based.
- [internal/feed](feed.md) — consumes `events.jsonl`, not `town.log`.
- [gt](../binaries/gt.md) — session spawn, nudge, handoff, and done
  paths all call into `townlog.Logger`.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- **One-way round-trip**: `parseLogLine` recovers the timestamp,
  type, and agent, but the `Context` field is never reconstructed.
  Anything reading log events back (`FilterEvents`, `TailEvents`)
  sees empty contexts. A caller who needs context must grep the raw
  file. Flagged as drift between the write path and the read path.
- The logger takes the mutex per call and re-opens the file each
  time. Fine for lifecycle events (low volume), but not a great
  fit if the caller starts logging per-token events.
- `ReadEvents` calls `os.ReadFile` on the entire log — unbounded
  memory for long-running towns. No tailing optimization. The
  existence of `TailEvents(n)` suggests most callers want a suffix,
  but the implementation still reads the whole file before slicing.
- The parser has a `//nolint:gosec G304` on the read because the
  path is built from a trusted `townRoot` string, not user input.
- The file path `<townRoot>/logs/town.log` is a plain text file,
  not a rotating log. Gas Town relies on the surrounding tooling to
  rotate / archive / cleanup. No rotation code lives here.
