---
title: internal/krc
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/krc/krc.go
  - /home/kimberly/repos/gastown/internal/krc/decay.go
  - /home/kimberly/repos/gastown/internal/krc/autoprune.go
tags: [package, platform-service, ttl, events, pruning, ephemeral-data]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/krc

Key Record Chronicle — configurable TTL management and auto-pruning for
Level 0 ephemeral operational data (patrol heartbeats, status checks,
session events, and other "operational noise that decays in forensic
value"). The package provides a TTL config with per-event-type durations,
a JSONL-file pruner with atomic rewrites, a forensic value decay model,
and an auto-prune scheduler that the daemon invokes on startup and on a
periodic interval.

**Go package path:** `github.com/steveyegge/gastown/internal/krc`
**File count:** 3 go files, 3 test files
**Imports (notable):** stdlib (`bufio`, `encoding/json`, `math`, `regexp`,
`sort`), plus [`internal/events`](events.md) for the canonical events-file
path constant.
**Imported by (notable):** the [`gt krc`](../commands/krc.md) command
surface (config / prune / stats / decay subcommands) and the daemon
auto-prune entry point.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/krc/krc.go`,
`/home/kimberly/repos/gastown/internal/krc/decay.go`,
`/home/kimberly/repos/gastown/internal/krc/autoprune.go`.

### Config and TTLs (`krc.go`)

- `Config` struct (`krc.go:31-48`) carries `DefaultTTL`, a
  `TTLs map[string]time.Duration` keyed by event-type pattern,
  `PruneInterval`, and `MinRetainCount`. JSON-serialized to
  `<townRoot>/.krc.yaml` (despite the `.yaml` extension the file is
  parsed as JSON — `krc.go:88-123`).
- `DefaultConfig()` (`krc.go:51-85`) hardcodes the starter TTL map:
  patrol/polecat noise at 24h, session heartbeats at 3d, nudge/handoff
  3d–7d, mail/merge at 30d, `session_death` at 30d, `mass_death` at 90d.
  Glob patterns (`patrol_*`, `merge_*`) are supported.
- `LoadConfig(townRoot)` / `SaveConfig(townRoot, cfg)` read and write the
  on-disk override (`krc.go:94-123`). Missing file returns
  `DefaultConfig()`.
- `Config.GetTTL(eventType)` (`krc.go:127-151`) resolves TTL by
  exact-match first, then glob patterns sorted longest-first for
  specificity, then `DefaultTTL`. `matchGlob` compiles each pattern into
  a regex with `.*` substitution (`krc.go:154-163`).

### Pruner (`krc.go`)

- `PruneResult` (`krc.go:166-174`) collects per-operation statistics:
  events processed / pruned / retained, bytes before/after, pruned-by-type
  histogram, duration.
- `Pruner` / `NewPruner(townRoot, config)` (`krc.go:177-188`) is the
  short-lived handle used to run one prune pass.
- `Pruner.Prune()` (`krc.go:192-228`) prunes two files in sequence:
  `<townRoot>/<events.EventsFile>` and `<townRoot>/.feed.jsonl`. Results
  are accumulated into a single `PruneResult`.
- `pruneFile(path)` (`krc.go:231-369`) is the core loop: scan JSONL
  lines, parse `{ts, type}`, apply `Config.GetTTL` to the parsed type,
  drop entries older than TTL, write retained lines to `<path>.tmp`,
  then atomic-rename over the original. Malformed lines and
  unparseable timestamps are retained defensively. Scanner buffer is
  bumped to 1MiB to tolerate long event payloads.

### Stats (`krc.go`)

- `Stats` (`krc.go:372-380`) describes the ephemeral surface: per-file
  size and event count, by-type histogram, by-age buckets
  (`0-1d`/`1-7d`/`7-30d`/`30d+`), oldest/newest timestamps, and a
  per-event-type TTL breakdown.
- `FileStats` and `TTLInfo` (`krc.go:383-395`) are the row types.
  `TTLInfo.ExpiresIn` is the time until the oldest non-expired event of
  that type falls over.
- `GetStats(townRoot, config)` (`krc.go:398-436`) reads both files via
  `getFileStats` (`krc.go:438-518`) which walks each line, buckets by
  age, and updates the per-type TTL info.

### Decay model (`decay.go`)

- `DecayCurve` iota (`decay.go:18-34`) with `DecayRapid`, `DecaySteady`,
  `DecaySlow`, `DecayFlat`. `defaultDecayCurves`
  (`decay.go:37-66`) maps patrol/polecat noise to rapid, session
  lifecycle to steady, hook/unhook/sling/done/error/recovery to slow,
  mail/session_death/merge_* to flat.
- `ForensicScore(eventType, age, ttl)` (`decay.go:71-82`) returns a
  float in [0, 1] given the event's age and its configured TTL. Uses
  `applyDecayCurve` (`decay.go:86-113`) which implements:
  rapid = `2^(-4·ratio)` (half-life at 25% TTL), steady = `1 - ratio`,
  slow = `2^(-ratio/0.75)` (half-life at 75% TTL), flat = full value
  until 90% TTL then linear drop to zero.
- `getDecayCurve(eventType)` (`decay.go:117-140`) matches exact patterns
  first, then glob patterns longest-first, falling back to
  `DecaySteady`.
- `DecayReport` / `DecayInfo` (`decay.go:152-172`) summarize
  per-event-type decay — count, TTL, curve, estimated avg age, avg
  score, min score, expired count — for human consumption via
  `gt krc decay`.
- `GenerateDecayReport(stats, config)` (`decay.go:175-241`) estimates
  average ages from `Stats.TTLBreakdown` (which doesn't carry individual
  event ages) and computes a total weighted score. The estimation is
  explicitly approximate; real ages are not retained post-Stats.

### Auto-prune scheduler (`autoprune.go`)

- `AutoPruneStateFile = ".krc-autoprune.json"` (`autoprune.go:12`) — the
  sibling file tracking "last prune time" so periodic runs can gate
  themselves on `PruneInterval`.
- `AutoPruneState` (`autoprune.go:15-21`) tracks `LastPruneTime`,
  `LastResult`, `PruneCount`, `TotalPruned`, `TotalBytesFreed`.
- `LoadAutoPruneState` / `SaveAutoPruneState` (`autoprune.go:30-65`)
  use an atomic write-to-temp-then-rename pattern.
- `AutoPruneState.ShouldPrune(interval)` (`autoprune.go:68-73`) returns
  true if we've never pruned OR if `time.Since(LastPruneTime) >=
  interval`.
- `AutoPrune(townRoot, config)` (`autoprune.go:94-117`) is the top-level
  daemon entry point: load state, check `ShouldPrune`, construct a
  Pruner, run one prune pass, record the result, save state. Returns
  `(result, ran, err)` so callers can distinguish "skipped because not
  time yet" from "ran and succeeded" from "errored".

## Related wiki pages

- [gt](../binaries/gt.md) — binary that exposes the krc subcommand tree.
- [gt krc](../commands/krc.md) — user-facing CLI surface for config,
  prune, stats, and decay report operations.
- [internal/events](events.md) — produces the JSONL data that krc then
  ages out. Supplies `events.EventsFile` as the canonical target path.
- [go-packages inventory](../inventory/go-packages.md).

## Outgoing calls

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| KRC config file | JSON data | Key rotation config persistence | `krc.go:119` |
| KRC temp file | rotation data | Atomic key file update | `krc.go:263` |
| autoprune state | JSON data | Autoprune state persistence | `autoprune.go:56` |

## Notes / open questions

- The config file extension is `.krc.yaml` but the on-disk format is
  JSON (`krc.go:88-110`). Editing it with a YAML tool will corrupt it.
- `DecayReport.AtRisk` is populated with `info.Expired` rather than
  a true "active events projected below 0.25" count (`decay.go:218-221`) —
  the comment acknowledges this is a rough estimate.
- `pruneFile` currently computes `result.EventsRetained = len(retained)`
  in both branches of the `MinRetainCount` check
  (`krc.go:330-335`) — the minimum-retain-count safety net is tracked
  in the field but not yet enforced against the retained slice.
