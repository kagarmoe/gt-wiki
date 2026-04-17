---
title: internal/feed
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/feed/curator.go
tags: [package, feed, curator, events, dedup, aggregation, zfc, flock]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/feed

The feed curator: a daemon goroutine that tails the raw events stream
(`<townRoot>/.events.jsonl`) and writes a filtered, deduped, aggregated
user-facing feed to `<townRoot>/.feed.jsonl`. This is the backing
implementation of `gt feed`. Raw events go in, summarized "what's
happening" lines come out.

**Go package path:** `github.com/steveyegge/gastown/internal/feed`
**File count:** 1 go file (`curator.go`), 1 test file.
**Imports (notable):** stdlib (`bufio`, `context`, `encoding/json`, `io`,
`log`, `os`, `path/filepath`, `sync`, `time`), `github.com/gofrs/flock`
for cross-process file locking, plus [`internal/config`](config.md) (for
`TownSettings.FeedCurator` tuning) and [`internal/events`](events.md)
(for `Event`, `EventsFile`, type/visibility constants).
**Imported by (notable):** the [`gt feed`](../commands/feed.md) command
and (most commonly) the deacon/daemon process that runs the curator as a
long-lived goroutine.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/feed/curator.go`.

### Lifecycle

- `NewCurator(townRoot string) *Curator` (`curator.go:66-93`) constructs
  a curator bound to a town. Loads `TownSettings.FeedCurator` via
  `config.LoadOrCreateTownSettings`, falling back to
  `config.DefaultFeedCuratorConfig()` on error. Three tunables land in
  the struct: `doneDedupeWindow`, `slingAggregateWindow`,
  `minAggregateCount` (`curator.go:58-62`, default 3 via `minAgg`
  guard).
- `Start() error` (`curator.go:97-119`) is a `sync.Once`-guarded
  bootstrap: opens `<townRoot>/.events.jsonl` with `O_RDONLY|O_CREATE`,
  seeks to end (only new events are processed), and spawns the `run`
  goroutine. Safe to call concurrently — extra callers are no-ops and
  all see the same `startErr`.
- `Stop()` (`curator.go:122-125`) cancels the context and
  `wg.Wait()`s for the goroutine to exit; the deferred `file.Close()`
  inside `run` handles the file handle.
- `run(file *os.File)` (`curator.go:129-153`) loops on a 100 ms ticker,
  reading every available line from the events file and calling
  `processLine`. No fsnotify: pure poll-at-100ms. The comment labels
  the design **ZFC** ("zero-footprint / derived-from-file state"): the
  curator keeps no in-memory cache of what it has already processed.

### Per-event pipeline

`processLine(line)` (`curator.go:156-178`):

1. Skip empty lines.
2. Unmarshal into `events.Event`; malformed lines are dropped silently.
3. Filter by visibility — only `events.VisibilityFeed` and
   `events.VisibilityBoth` survive. Audit-only events never reach the
   feed.
4. Run `shouldDedupe(&event)` — if true, drop.
5. Otherwise call `writeFeedEvent(&event)`.

### Deduplication (`curator.go:183-203`)

Currently a single case: `events.TypeDone`. To decide, the curator
reads back the **feed file itself** for events within `doneDedupeWindow`
(`readRecentFeedEvents`, `curator.go:216-280`), and if any of them is a
`done` from the same actor, the new event is dropped as a duplicate.
This is the load-bearing ZFC choice: "what have we already written" is
derived from the output file, not a sync.Map. Mail and sling events
fall through unchanged at this stage; their dedup logic (aggregation)
runs later, in `writeFeedEvent`.

Fail-open: if the feed file read errors, dedup is skipped and the event
is written.

### Aggregation (`curator.go:356-373`)

`writeFeedEvent` is where sling events get aggregated. For
`events.TypeSling`, it counts recent slings from the same actor inside
`slingAggregateWindow` via `countRecentSlings`
(`curator.go:340-352`) → `readRecentEvents` (`curator.go:285-336`). If
`count >= minAggregateCount` (default 3), the event is rewritten with
`Count = N` and summary `"<actor> dispatching work to N agents"`. Fewer
than the threshold means the event is passed through individually.

Again ZFC: aggregation counts come from scanning the events file, not
from an in-memory counter.

### File reads are tail-bounded

Both `readRecentFeedEvents` and `readRecentEvents` seek to
`max(0, size - tailReadSize)` where `tailReadSize = 1 << 20` (1 MB,
`curator.go:211`). The first potential partial line after the seek is
skipped with a throwaway `scanner.Scan()`. This caps per-read memory at
~1 MB regardless of how long the feed has been running.

### Write path and locking (`curator.go:354-415`)

- Each write goes through `feedMu` (in-process mutex) **and** a
  cross-process `flock.New(feedPath + ".lock")`. The in-process mutex
  coordinates goroutines in one binary; the file lock coordinates
  multiple processes writing to the same `.feed.jsonl`
  (`curator.go:384-394`).
- Before appending, if the feed file exceeds `maxFeedFileSize` (10 MB,
  `curator.go:207`), `truncateFeedFile` copies the newest half into a
  sibling `*.truncate.tmp` and atomic-renames it into place
  (`curator.go:419-465`). Read handle is closed before rename so
  Windows doesn't choke. Errors during truncation are best-effort —
  the function just returns.
- `O_APPEND|O_CREATE|O_WRONLY` for the append, permissions 0600.

### `FeedEvent` shape (`curator.go:32-40`)

Written to the feed file as one JSON per line: `ts`, `source`, `type`,
`actor`, `summary`, optional `payload`, optional `count` (for
aggregated events).

### Human summaries (`generateSummary`, `curator.go:468-544`)

Per-event-type template strings — the layer that turns
`events.TypeMerged` into `"Merged work from <worker>"`. Handled types:
`TypeSling`, `TypeDone`, `TypeHandoff`, `TypeMail`, `TypePatrolStarted`,
`TypePatrolComplete`, `TypeMerged`, `TypeMergeFailed`,
`TypeSessionDeath`, `TypeMassDeath`. Unknown types fall through to
`"<actor>: <type>"`. `TypeMassDeath` is called out in its own loud form:
`"MASS DEATH: N sessions died - <cause>"`.

## Related wiki pages

- [gt feed](../commands/feed.md) — the user-facing CLI.
- [internal/events](events.md) — producer of the `.events.jsonl` stream
  this package consumes; defines `Event`, `EventsFile`, visibility and
  type constants.
- [internal/channelevents](channelevents.md) — sibling event surface.
- [internal/config](config.md) — supplies `FeedCuratorConfig` tuning
  and the default path helpers for `TownSettings`.
- [gt](../binaries/gt.md).
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The 100 ms polling tick for the events file (`run`, `curator.go:134`)
  means feed latency is at least 100 ms and more commonly 200–300 ms.
  No fsnotify integration; if the poll interval needs tightening it's
  a single literal change.
- The ZFC design means dedup and aggregation are O(tail-window × lines)
  per relevant event. With a 30 s aggregate window and busy towns this
  is a handful of KB per call. The `tailReadSize = 1 MB` cap implicitly
  bounds the maximum window — windows exceeding 1 MB of events will
  start missing earlier events silently.
- `shouldDedupe` currently only handles `TypeDone`. Other high-volume
  types (e.g. repeated `TypeHandoff` from the same actor) would need
  their own case; right now they pass through untouched and rely on
  aggregation or consumer-side filtering.
- Fail-open on read errors means a truncated or partially-written feed
  file results in duplicate writes rather than lost events. Trade-off
  noted here.
