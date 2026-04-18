---
title: internal/channelevents
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/channelevents/channelevents.go
tags: [package, data-layer, events, await-event, channels, refinery, rendezvous]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [precondition]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 2}
---

# internal/channelevents

File-based publish/subscribe primitive for named channels. `Emit(...)`
writes a JSON event file under `<townRoot>/events/<channel>/` and
`await-event` subscribers (e.g., the refinery waiting on `MERGE_READY`)
poll or watch that directory to wake up when matching events land. This is
**not** the same as the activity feed in [`internal/events`](events.md);
see the comparison at the bottom of this page.

**Go package path:** `github.com/steveyegge/gastown/internal/channelevents`
**File count:** 1 go file, 1 test file
**Imports (notable):**

- [`internal/workspace`](workspace.md) — `FindFromCwd` to locate the town root.
- stdlib: `encoding/json`, `os`, `path/filepath`, `regexp`, `strings`,
  `sync/atomic`, `time`.

**Imported by (notable):** the `gt emit` / `gt emit-event` command path,
the refinery's merge-queue signaling, any code that needs to hand off a
named-channel rendezvous file to a subscriber (typically paired with a
`gt await-event <channel> <type>` wait).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/channelevents/channelevents.go`.

### Public API

- `ValidChannelName = regexp.MustCompile(`^[a-zA-Z0-9_-]+$`)` (line 23) —
  compiled once; channel names are restricted to alphanumerics, underscores,
  and hyphens to block path-traversal.
- `Emit(channel, eventType string, payloadPairs []string) (string, error)`
  (lines 31-47) — the primary entry point. Resolves the town root via
  `workspace.FindFromCwd()`, falling back to `$HOME/gt` if the caller isn't
  in a workspace (the **only** place in this package that silently "defaults
  a workspace" — callers that need strict town-scoping should use
  `EmitToTown`). Creates `<townRoot>/events/<channel>/` if needed, then
  delegates to `emitToDir`. Returns the full path to the event file just
  written.
- `EmitToTown(townRoot, channel, eventType string, payloadPairs []string) (string, error)`
  (lines 51-61) — explicit-town-root variant for internal callers that
  already know the town root. Same behavior as `Emit` but skips the
  `FindFromCwd` walk.

### Event file shape

Each `Emit` call produces a JSON file at
`<townRoot>/events/<channel>/<unixNano>-<seq>-<pid>.event`
(line 91). The file contents are a JSON object (pretty-printed via
`json.MarshalIndent`, lines 85-88):

```json
{
  "type":      "<eventType>",
  "channel":   "<channel>",
  "timestamp": "<RFC3339>",
  "payload":   { "<key>": "<val>", ... }
}
```

`payloadPairs` is a `[]string` of `"key=value"` strings; `strings.Cut(...,
"=")` splits each into a `map[string]string` (lines 69-75). Pairs without
`=` are silently dropped.

### Internals / Notable implementation

- **Unique filenames under rapid-fire emits.** The filename pattern
  `<unixNano>-<seq>-<pid>.event` combines three sources of entropy
  (line 91):
  - `time.Now().UnixNano()` — monotonic but low-resolution on some OSes.
  - `emitSeq` — a package-level `atomic.Uint64` counter, incremented per
    call (line 27, line 90). Guarantees in-process uniqueness even if
    two calls happen within the same nanosecond.
  - `os.Getpid()` — disambiguates across cooperating processes.
  Together these avoid collisions when several goroutines or processes
  emit to the same channel in a tight loop.
- **Channel-name validation is double-checked.** Both `Emit` /
  `EmitToTown` and `emitToDir` call `ValidChannelName.MatchString` (lines
  32-34, 52-54, 65-67). The second check defends against internal
  callers that skip the public API.
- **No locking.** Each event is a separate file. Writers don't contend,
  and readers can scan the directory at their own cadence. Contrast
  `internal/events`, which appends to a single JSONL and needs flock.
- **0755 for dirs, 0644 for files.** Standard operational permissions.
- **`~/gt` fallback.** In `Emit` only (not `EmitToTown`), if
  `workspace.FindFromCwd` fails the package silently falls back to
  `$HOME/gt` as the town root (lines 37-40). That means an emit done
  outside any workspace still writes *somewhere* rather than returning
  an error — intentional for tooling that might fire events before
  `gt init`-like setup.

### Usage pattern

Producer:

```go
path, err := channelevents.Emit(
    "merge-queue-main",
    "MERGE_READY",
    []string{"mr=gt-abc123", "branch=feature/foo"},
)
```

Consumer: `gt await-event merge-queue-main MERGE_READY` (separate binary /
subcommand, not in this package) watches
`<townRoot>/events/merge-queue-main/*.event`, parses each JSON body, and
returns when the `type` field matches. After the consumer acts, it's
responsible for deleting the event file — this package only writes.

## Relationship to internal/events

These are **two different event systems** sharing a vaguely similar name;
`channelevents` is not a layer built on top of `events`, and vice versa.

| Axis | [`internal/events`](events.md) | `internal/channelevents` (this page) |
|---|---|---|
| Storage | Single JSONL file at `<townRoot>/.events.jsonl` | One file per event under `<townRoot>/events/<channel>/` |
| Ordering | Total order by append position | Per-channel by filename timestamp |
| Write sync | `flock` cross-process lock on the whole log | None — each event is a new file |
| Intended reader | The feed daemon (tails the JSONL and curates) | `gt await-event` subscribers (one-shot waits) |
| Semantics | "This happened" (audit trail) | "Wake up when this arrives" (rendezvous / pub-sub) |
| Granularity | Flat global stream | Named channels |
| Lifetime | Append forever (no rotation in-package) | Consumer deletes the file after handling |

A sling operation emits *both*: a feed-visible `sling` event into
`.events.jsonl` (via `events.LogFeed`) and, when the sling targets a queue
that has a subscriber, a channel event via `channelevents.Emit` to wake
that subscriber.

## Related wiki pages

- [gt](../binaries/gt.md)
- [internal/events](events.md) — the activity-feed audit log; distinct from
  this package.
- [internal/workspace](workspace.md) — town root resolution.
- [internal/telemetry](telemetry.md) — OTel-based event telemetry, orthogonal
  to this file-based rendezvous.
- [go-packages inventory](../inventory/go-packages.md)

## Failure modes

### Precondition violations (what does it assume?)
- **Town root fallback to ~/gt:** `Emit` at `channelevents.go:36-39`
  — when `workspace.FindFromCwd()` fails or returns empty, the code
  silently falls back to `$HOME/gt`. If `$HOME` is unset or the
  directory doesn't exist, `os.MkdirAll` at line 42 will create it.
  **Absent** — no warning that events are being written to a fallback
  location; subscribers watching the real town's `events/` directory
  will never see them.

## Outgoing calls

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| channel event file | JSON event data | Channel event persistence | `channelevents.go:92` |

## Notes / open questions

- This package only writes. The `await-event` consumer lives elsewhere
  (likely `internal/cmd` or a dedicated subcommand package) and owns the
  polling/filesystem-watching logic plus event-file cleanup.
- The filename scheme depends on `time.Now().UnixNano()` increasing across
  subsequent `Emit` calls. With the `emitSeq` atomic counter this is
  robust in practice even on macOS where clocks can jitter.
- Channel names are a flat namespace; there's no directory hierarchy.
  Convention presumably encodes scope (e.g. `merge-queue-main`,
  `merge-queue-feature-x`) in the name itself.
- The `~/gt` fallback in `Emit` (not `EmitToTown`) is a small but real
  discrepancy between the two entry points and worth noting if a caller
  needs strict "fail when not in a town" behavior.
