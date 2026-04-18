---
title: gt trail
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/trail.go
tags: [command, work, trail, audit, activity, git-log, hook-events, beads]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt trail

Show recent agent activity across three axes: git commits (by agent
email domain), recent beads, and hook/unhook events. Three aliases:
`recent`, `recap`. Bare `gt trail` defaults to showing agent commits.

**Parent:** [gt](../binaries/gt.md) (root command), aliases
`recent`, `recap`
**Group:** `GroupWork` ("Work Management") (`trail.go:29`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/trail.go` (509
lines).

### Invocation

```
gt trail [--since 1h] [--limit 20] [--json] [--all]
gt trail commits [flags]
gt trail beads [flags]
gt trail hooks [flags]
gt recent                # alias
gt recap --since 24h     # alias
```

Bare `gt trail` runs `runTrailCommits` (`trail.go:54` — the parent
`RunE` is set to the commits runner).

### Subcommands

Registered at `trail.go:105-107`:

| subcommand | source | one-liner |
|---|---|---|
| `commits` | `:57-71`, `runTrailCommits :125-219` | Recent git commits, filtered by the configured agent email domain unless `--all`. |
| `beads` | `:73-83`, `runTrailBeads :240-331` | Recent beads via `beads query` with fallback to `beads list`. |
| `hooks` | `:85-95`, `runTrailHooks :343-397` | Hook/unhook events parsed from the town's event feed file. |

### Commits pipeline (`runTrailCommits`, `trail.go:125-219`)

1. **Resolve agent email domain.** Start with
   `DefaultAgentEmailDomain`; if `workspace.FindFromCwd` finds a
   town root, try `config.LoadOrCreateTownSettings` and override
   from `settings.AgentEmailDomain` (`:127-134`).
2. **Build git-log args.**
   `git log --format=%H|%h|%an|%ae|%aI|%ar|%s -n<limit*2>` — double
   limit to have headroom after filtering (`:136-150`).
3. **Apply `--since`** via `parseDuration` (other helper) and
   `--since=<RFC3339>` (`:143-150`).
4. **Parse and filter.** For each pipe-delimited line, detect agent
   by `strings.HasSuffix(email, "@"+domain)`. Skip non-agents
   unless `--all`. Stop after `--limit` matches.
5. **Output.** JSON or colored text with agent authors bolded.

### Beads pipeline (`runTrailBeads`, `trail.go:240-331`)

1. `findBeadsDir` (`:470-479`) — local beads dir, falling back to
   `findMailWorkDir()`.
2. Try `beads query --format {{.ID}}|{{.Title}}|... --limit N
   --sort -updated_at` (`:248-268`). `BEADS_DIR` env is set to the
   resolved dir.
3. On error, fallback to `runTrailBeadsSimple` (`:333-341`) which
   just runs `beads list --limit N` with stdout/stderr inherited.
4. Parse pipe-delimited lines into `BeadEntry`; print with status
   coloring (`open`=success, `in_progress`=warning, `done/merged`
   =info).

### Hooks pipeline (`runTrailHooks`, `trail.go:343-397`)

1. Require town root.
2. Parse `--since` if set.
3. `readHookTrailEntries(townRoot/events.EventsFile, since, limit)`
   (`:399-468`) — reads the events file, iterates **backward**
   (newest first), unmarshals each JSON line into `events.Event`,
   keeps only `TypeHook`/`TypeUnhook`, respects the since cutoff,
   and emits `HookEntry{Type, Actor, Bead, Timestamp, TimeRel}`.
4. Output: `<timestamp> <actor> hooked|unhooked <bead>` with a
   relative-time subline.

### Types

```go
// trail.go:114-123 (commit)
type CommitEntry struct { Hash, ShortHash, Author, Email string
    Date time.Time; DateRel, Subject string; IsAgent bool }

// trail.go:222-229 (bead)
type BeadEntry struct { ID, Title, Status, Agent string
    UpdatedAt time.Time; UpdateRel string }

// trail.go:232-238 (hook)
type HookEntry struct { Type, Actor, Bead string
    Timestamp time.Time; TimeRel string }
```

### Flags (persistent on `trailCmd`, `:99-102`)

| flag | default | notes |
|---|---|---|
| `--since <duration>` | `""` | e.g. `1h`, `24h`, `7d`. Parsed by `parseDuration`. |
| `--limit <n>` | `20` | Cap items. For commits, fetched as `2*limit` and filtered. |
| `--json` | `false` | |
| `--all` | `false` | Commits only: include non-agent authors. |

### Helpers

- `relativeTime(t)` (`:481-509`) — "just now" / "N minutes ago" /
  "N hours ago" / "N days ago". Used by both beads and hooks
  display paths.
- `findBeadsDir` (`:470-479`) — delegates to `findLocalBeadsDir`
  with a fallback to `findMailWorkDir`.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **JSON parse errors in events file:** `readHookTrailEntries` at `trail.go:430-431` — if `json.Unmarshal` fails on an event line, it silently `continue`s, skipping the entry with no warning. **Absent** — corrupted event lines are invisible to the user.
- **Date parse errors on commits:** `trail.go:171` — `time.Parse(time.RFC3339, parts[4])` error is discarded with `date, _ := ...`. The commit still renders but with a zero `Date` value. **Absent** — could display incorrect time data.
- **Date parse errors on beads:** `trail.go:287` — same pattern: `time.Parse(time.RFC3339, parts[4])` error discarded. **Absent**.
- **Config load errors suppressed:** `trail.go:129-132` — `workspace.FindFromCwd()` and `config.LoadOrCreateTownSettings()` errors are silently caught; falls back to `DefaultAgentEmailDomain`. **Present** — graceful fallback, but no warning that custom domain config was ignored.

## Notes / open questions

- **Three different data sources, one display.** git log, `beads
  query`, and the JSONL event feed are three unrelated plumbing
  paths. The command unifies them as "recent activity" but there
  is no correlation (you can't see `gt-abc` hooked → committed →
  done on the same display).
- **Agent detection by email domain.** `trail commits` filters by
  `@<configured-domain>` suffix. Out-of-the-box that's
  `DefaultAgentEmailDomain`; configurable via
  `config.TownSettings.AgentEmailDomain` (see [config](config.md)).
  Non-agent humans committing with the configured domain will be
  counted as agents.
- **Events file parsing is "read whole file, iterate backward".**
  (`:417-423`) — fine at current event-log sizes, but if the feed
  grows into gigabytes this becomes O(whole file) per invocation.
- **No `--rig` filter.** Unlike [ready](ready.md) and
  [orphans](orphans.md), `trail` doesn't scope to a single rig;
  it always runs against the cwd's git repo and the town-level
  events file.
- **`recent`/`recap` aliases** make discovery awkward — `gt help`
  will list the command once with the aliases noted, but search
  and tab-completion on "recap" should still find it.
- **Related.** [audit](audit.md) (the Diagnostics-group audit
  surface — overlapping concerns), [feed](feed.md) (events feed
  writer), [hook](hook.md) (the event producer),
  [changelog](changelog.md) (release-notes-ish activity view).
