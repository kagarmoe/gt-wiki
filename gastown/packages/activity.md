---
title: internal/activity
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/activity/activity.go
tags: [package, activity, dashboard, ui-helper, status]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/activity

Last-activity tracking and color-coding helper for the dashboard and other
status surfaces. Takes a `time.Time` of "last observed activity" and
converts it into a short human-readable age plus a green/yellow/red/unknown
color class keyed to agent-stuckness thresholds.

**Go package path:** `github.com/steveyegge/gastown/internal/activity`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib `time` only. Deliberately strconv-free —
`formatInt` is hand-rolled at `activity.go:100-111` to keep the package a
near-zero-dependency leaf.
**Imported by (notable):** dashboard / status rendering paths that display
agent last-activity bars. Referenced indirectly from the
[`gt activity`](../commands/activity.md) command family and the
dashboard/feed UI.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/activity/activity.go`.

### Public API

**Color class constants** (`activity.go:9-14`):

- `ColorGreen`   — active: <5 minutes
- `ColorYellow`  — stale: 5-10 minutes
- `ColorRed`     — stuck: >10 minutes
- `ColorUnknown` — no activity data (zero time value)

These are plain strings (`"green"`, `"yellow"`, `"red"`, `"unknown"`) so
they double as CSS class names on the web dashboard and as lookup keys for
terminal styling.

**Thresholds** (`activity.go:18-21`):

- `ThresholdActive = 5 * time.Minute`
- `ThresholdStale  = 10 * time.Minute`

The package comment notes these are "configurable via operational.session
thresholds or worker_status in settings/config.json" but the package itself
doesn't read config — these constants are compile-time. Config overrides
must happen in the caller.

**Types and functions:**

- `Info` struct (`activity.go:24-29`): `LastActivity time.Time`,
  `Duration time.Duration`, `FormattedAge string`, `ColorClass string`.
  All four fields are always populated after `Calculate`, even in the
  unknown case.
- `Calculate(lastActivity time.Time) Info` (`activity.go:37-64`) — the
  single entry point. Handles the zero-time case (returns
  `ColorUnknown`/`"unknown"`), clamps negative durations from clock skew to
  zero (`activity.go:53-55`), then formats and colors.
- `(Info).IsActive() bool` / `IsStale() bool` / `IsStuck() bool`
  (`activity.go:126-138`) — color-class predicates for callers that want
  boolean state checks instead of string comparisons.

### Internals

- `formatAge` (`activity.go:68-79`) produces `"<1m"`, `"5m"`, `"2h"`,
  `"1d"` in that cascading order. No two-unit formatting ("1h5m") — the
  coarsest unit always wins.
- `formatInt` (`activity.go:100-111`) is a hand-rolled decimal formatter
  that avoids pulling in `strconv`. It optimizes the single-digit case
  with a direct rune conversion and then falls back to an iterative
  quotient/remainder loop that builds the result string from least- to
  most-significant digit. This is the only "why is this here?" piece in
  the package and the package comment calls it out as deliberate.
- `colorForDuration` (`activity.go:114-123`) is a three-arm switch keyed
  to the two thresholds; no smoothing or hysteresis.

## Docs claim

The package doc (`activity.go:1`) describes it as "last-activity tracking
and color-coding for the dashboard." The threshold comment
(`activity.go:16-17`) claims the values are "configurable via
operational.session thresholds or worker_status in settings/config.json"
but there is no code in this package that reads those settings — it's a
note to whoever wires the package into the dashboard.

## Drift

- **Configurability claim vs. implementation.** The thresholds are
  compile-time constants. If the dashboard or another caller wants
  configurable thresholds, it has to read the settings itself and call
  something other than `Calculate` — there is no `CalculateWithThresholds`
  variant in this package today. Low-impact for now (5m/10m is a
  reasonable default) but worth knowing before someone goes looking for
  the knob.

## Notes / open questions

- No tests observed in-page, but `activity_test.go` exists alongside the
  source — coverage scope unreviewed.
- `formatAge` rounds down by integer truncation, so 119 seconds renders as
  `"1m"` and 3599 seconds as `"59m"`. Intentional for dashboard brevity,
  but callers looking for sub-minute precision won't get it.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary; the dashboard/feed commands
  eventually render activity colors.
- [gt activity](../commands/activity.md) — the user-facing command
  surface for activity queries.
- [internal/events](events.md) — events feed the "last activity"
  timestamps that this package then color-codes.
- [go-packages inventory](../inventory/go-packages.md).
