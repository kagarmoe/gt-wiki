---
title: gt activity
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/activity.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
  - /home/kimberly/repos/gastown/internal/events/events.go
tags: [command, diagnostics, events, activity-feed]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 2}
---

# gt activity

Emits activity events into the Gas Town activity feed at `<town-root>/.events.jsonl`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` set in
`/home/kimberly/repos/gastown/internal/cmd/activity.go:28-38`)
**Beads-exempt:** no (not in `beadsExemptCommands` map at
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`, cross-referenced in
[../binaries/gt.md](../binaries/gt.md))
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` map at
`/home/kimberly/repos/gastown/internal/cmd/root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/activity.go:28-196`.

### Invocation

```
gt activity emit <event-type> [flags]
```

The top-level `activity` command is a cobra parent with no `Run` of its own
(`activity.go:28-38`). All work happens in the single registered subcommand
`activity emit` (`activity.go:40-71`).

### Behavior

`runActivityEmit` (`activity.go:90-196`):

1. Requires a Gas Town workspace — calls `workspace.FindFromCwdOrError`
   (`activity.go:94-97`). Fails with "not in a Gas Town workspace" if not
   inside one.
2. Resolves the actor (`activity.go:99-103`): uses `--actor` flag, or calls
   `detectActor()` (defined in `internal/cmd/sling.go` — cross-referenced
   here as a sibling helper).
3. Switches on the event type (`activity.go:108-182`) to build the payload:
   - `patrol_started`, `patrol_complete` — require `--rig`. Uses
     `events.PatrolPayload` (`activity.go:109-113`).
   - `polecat_checked` — requires `--rig` and `--polecat`. Defaults
     `--status` to `"checked"` if unset (`activity.go:115-122`).
   - `polecat_nudged` — requires `--rig` and `--polecat`
     (`activity.go:124-128`).
   - `escalation_sent` — requires `--rig`, `--target`, and `--to`
     (`activity.go:130-134`).
   - `merge_started`, `merged`, `merge_failed`, `merge_skipped` — refinery
     events with a flexible payload (`activity.go:136-150`).
   - Anything else — generic payload built from whichever flags are set
     (`activity.go:152-181`).
4. Calls `events.LogFeed(eventType, actor, payload)`
   (`activity.go:185-187`) to append a JSON line to
   `<town-root>/.events.jsonl` (the file path is defined as
   `events.EventsFile` in the [`internal/events`](../packages/events.md) package).
5. Prints a confirmation line with the actor and payload JSON
   (`activity.go:189-193`).

Event type constants come from the `internal/events` package (imported at
`activity.go:8`). Event types referenced include `TypePatrolStarted`,
`TypePolecatChecked`, `TypePolecatNudged`, `TypeEscalationSent`,
`TypePatrolComplete`, `TypeMergeStarted`, `TypeMerged`, `TypeMergeFailed`,
and `TypeMergeSkipped`.

### Subcommands

- `emit <event-type>` — emit an activity event (the only subcommand;
  `activity.go:40-71`, `activity.go:86`).

Running `gt activity` without `emit` prints the Long help text
(`activity.go:32-37`).

### Flags

All flags are defined on `activity emit` (`activity.go:75-85`), not on
`activity` itself:

| Flag        | Type   | Default | Purpose                                                     |
|-------------|--------|---------|-------------------------------------------------------------|
| `--actor`   | string | `""`    | Actor emitting the event (auto-detected via `detectActor`)  |
| `--rig`     | string | `""`    | Rig the event is about                                      |
| `--polecat` | string | `""`    | Polecat involved (for `polecat_checked`, `polecat_nudged`)  |
| `--target`  | string | `""`    | Target of the action (for escalation)                       |
| `--reason`  | string | `""`    | Reason for the action                                       |
| `--message` | string | `""`    | Human-readable message                                      |
| `--status`  | string | `""`    | Status for `polecat_checked` (working/idle/stuck/checked)   |
| `--issue`   | string | `""`    | Issue ID (for `polecat_checked`)                            |
| `--to`      | string | `""`    | Escalation target (mayor/deacon) for `escalation_sent`      |
| `--count`   | int    | `0`     | Polecat count (for patrol events)                           |

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/activity.go:43-68` — Cobra `Long` text on `activityEmitCmd`.

### Verbatim

> Emit an activity event to the Gas Town activity feed.
>
> Supported event types for witness patrol:
>   patrol_started   - When witness begins patrol cycle
>   polecat_checked  - When witness checks a polecat
>   polecat_nudged   - When witness nudges a stuck polecat
>   escalation_sent  - When witness escalates to Mayor/Deacon
>   patrol_complete  - When patrol cycle finishes
>
> Supported event types for refinery:
>   merge_started    - When refinery starts a merge
>   merge_complete   - When merge succeeds
>   merge_failed     - When merge fails
>   queue_processed  - When refinery finishes processing queue

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Refinery event type names in Cobra `Long` don't match event constants

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/activity.go:52-56`.
- **Docs claim:** the `Long` text advertises refinery event types `merge_started`, `merge_complete`, `merge_failed`, and `queue_processed` as the supported set.
- **Code does:** `runActivityEmit` switches on `events.TypeMergeStarted`, `events.TypeMerged`, `events.TypeMergeFailed`, `events.TypeMergeSkipped` at `/home/kimberly/repos/gastown/internal/cmd/activity.go:136`. The constants resolve to the string values `"merge_started"`, `"merged"`, `"merge_failed"`, `"merge_skipped"` at `/home/kimberly/repos/gastown/internal/events/events.go:67-70`. Users who type `merge_complete` or `queue_processed` fall through to the generic-event default branch (`activity.go:152`) — the event is still emitted but without the refinery-specific payload shape, and the documented names never produce a refinery-typed payload.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — edit the `Long` text in `activity.go` to name the actual constants (`merged` and `merge_skipped`; drop `queue_processed` or document what constant it would map to if added).
- **Release position:** `in-release` (text identical at `v1.0.0:internal/cmd/activity.go`).

## Related

- [feed](feed.md) — reads and streams the same `.events.jsonl` file that
  `activity emit` writes into.
- [audit](audit.md) — `collectFeedEvents` (`audit.go:405-448`) reads the
  same event file for the cross-source provenance timeline.
- [log](log.md) — the complementary townlog command; `townlog` events and
  `.events.jsonl` events are distinct streams both surfaced by
  [audit](audit.md).
- [../packages/activity.md](../packages/activity.md) — the
  activity-tracking color/age classification library used by the
  status/feed readers on the same event stream.
- [../binaries/gt.md](../binaries/gt.md) — parent binary and
  `persistentPreRun` context.
- [README.md](README.md) — command tree index.

## Failure modes

No significant failure modes. All error paths return proper errors to
cobra. The `events.LogFeed` call at `activity.go:185` propagates its
error. The only suppression is `json.Marshal(payload)` at
`activity.go:190` which discards the error for the confirmation
message, but the event was already written by that point.

## Notes / open questions

- `detectActor` is imported from `sling.go` per the note at
  `activity.go:198`. Its resolution rules (GT_ROLE, GT_SESSION, etc.) are
  not documented here yet.
- The events-file path is `<town-root>/.events.jsonl` via
  `events.EventsFile` (seen in use at `audit.go:408`); an explicit
  `internal/events` package page would make this authoritative.
- The event-type set accepted by the switch is a subset of the full
  `events.Type*` constant list — generic events fall through to the
  default branch with a flexible payload.
