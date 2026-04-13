---
title: gt audit
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/audit.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, provenance, git, beads, townlog, events]
---

# gt audit

Queries provenance data across four sources — git commits, beads issues,
town log events, and activity feed events — and prints a unified
time-sorted timeline filtered by actor and/or duration.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/audit.go:30-51`)
**Beads-exempt:** no (not in `beadsExemptCommands`; see
[../binaries/gt.md](../binaries/gt.md))
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/audit.go:30-571`.

### Invocation

```
gt audit [--actor <addr>] [--since <dur>] [--limit N] [--json]
```

Single terminal command (no subcommands). Runs `runAudit`
(`audit.go:73-145`).

### Behavior

`runAudit` (`audit.go:73-145`) collects entries from four sources, merges
them, sorts newest-first, applies a limit, and emits text or JSON.

1. Requires a Gas Town workspace — `workspace.FindFromCwdOrError`
   (`audit.go:74-77`).
2. Parses `--since` using a duration parser that adds a `d` (days) suffix
   on top of Go's `time.ParseDuration` (`audit.go:148-159`). Supports
   `"1h"`, `"30m"`, `"24h"`, `"7d"`, etc.
3. Collects entries from each source. Each collector is non-fatal — failures
   print a warning to stderr and continue (`audit.go:92-119`):
   - **`collectGitCommits`** (`audit.go:162-220`) — shells out to
     `git log --format=%H|%aI|%an|%s --all --author=<name> --since=<ts> -n 100`.
     If `--actor` is `greenplace/crew/joe`, uses `joe` as the git author
     (`extractAuthorName`, `audit.go:223-231`). Also matches the raw actor
     string against commit subject as a fallback
     (`audit.go:205-207`).
   - **`collectBeadsActivity`** (`audit.go:258-317`) — opens the beads DB
     at `<townRoot>/gastown/mayor/rig` (`audit.go:262`) via `beads.New` and
     calls `b.List({Status: "all", Priority: -1})`. Emits one entry per
     `created_by` match and one per `status=closed` `assignee` match.
     Only the gastown rig is queried — other rigs' beads are not included.
   - **`collectTownlogEvents`** (`audit.go:337-366`) — calls
     `townlog.ReadEvents(townRoot)` which reads
     `<townRoot>/logs/town.log` (the same file [log](log.md) reads). Event
     types rendered via `formatTownlogSummary` (`audit.go:369-402`):
     `spawn`, `done`, `handoff`, `handoff-NOPERSIST`, `crash`, `kill`,
     `nudge`, `wake`.
   - **`collectFeedEvents`** (`audit.go:405-448`) — reads the same
     [`.events.jsonl`](../packages/events.md) file that [activity](activity.md) writes and
     [feed](feed.md) streams. Rendered via `formatFeedSummary`
     (`audit.go:451-483`) for types `sling`, `merged`, `merge_failed`,
     `handoff`, `done`, `mail`.
4. Sorts all entries newest-first by timestamp (`audit.go:122-124`).
5. Applies `--limit N` (`audit.go:127-129`; default 50).
6. Prints either `outputAuditJSON` (`audit.go:485-489`) or
   `outputAuditText` (`audit.go:491-529`) grouped by day with
   color-coded source and type tags (`formatSource`/`formatType`,
   `audit.go:531-571`).

Actor matching (`matchesActor`, `audit.go:234-255`) is case-insensitive
and partial — matches if the stored name equals the actor, contains the
actor's last path segment, or contains the full actor string.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`audit.go:53-60`):

| Flag              | Type   | Default | Purpose                                                      |
|-------------------|--------|---------|--------------------------------------------------------------|
| `--actor`         | string | `""`    | Filter by actor (agent address or partial match)             |
| `--since`         | string | `""`    | Filter to events newer than duration (supports `d` suffix)   |
| `--limit` / `-n`  | int    | `50`    | Maximum entries to print                                     |
| `--json`          | bool   | `false` | Output as a JSON array of `AuditEntry` records               |

### AuditEntry shape

`AuditEntry` (`audit.go:63-71`):

```go
type AuditEntry struct {
    Timestamp time.Time
    Source    string // "git" | "beads" | "townlog" | "events"
    Type      string // "commit" | "bead_created" | "bead_closed" | "spawn" | ...
    Actor     string
    Summary   string
    Details   string
    ID        string // commit hash, bead ID, etc.
}
```

## Related

- [log](log.md) — pure townlog reader; `audit` reuses `townlog.ReadEvents`
  for one of its four sources.
- [feed](feed.md) — streams live events from `.events.jsonl`; `audit`
  reads the same file historically.
- [activity](activity.md) — writes to `.events.jsonl`, the source
  `collectFeedEvents` scans.
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Notes / open questions

- The beads location is hard-coded to `<townRoot>/gastown/mayor/rig`
  (`audit.go:262`). Audit results for non-`gastown` rigs appear only via
  git and townlog, not beads.
- `collectGitCommits` caps `git log` at `-n 100` regardless of `--limit`
  (`audit.go:180`), so large-limit audits can miss git history older than
  the 100 most recent commits.
- `extractAuthorName` assumes actor format `role/rig/name` with the
  human name last (`audit.go:223-231`). Actors that don't follow this
  pattern get passed to git `--author=` verbatim.
- `parseDuration` only honors a `d` suffix; `"7w"` or `"1mo"` fall through
  to `time.ParseDuration` and fail.
