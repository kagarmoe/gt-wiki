---
title: gt memories
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/memories.go
  - /home/kimberly/repos/gastown/internal/cmd/remember.go
tags: [command, work, memory, beads, kv-store]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt memories

List or search memories stored in the beads key-value store. Read side
of the memory trio with [remember](remember.md) and [forget](forget.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") — set on `memories.go:16`
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Batch-plan correction:** `gt memories` is in `GroupWork`, not
> ungrouped. See the same note on [commit](commit.md).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/memories.go:20-142`.

### Invocation

```
gt memories [search-term] [--type feedback|project|user|reference|general]
```

`Use: "memories [search-term]"` (`memories.go:21`). `Args:
cobra.MaximumNArgs(1)` (`memories.go:39`).

### Flags

| flag | type | default | description |
|------|------|---------|-------------|
| `--type` | string | `""` | Filter by memory type (`memoriesTypeFilter` on `memories.go:12`) |

Valid types: `feedback`, `project`, `user`, `reference`, `general`
(`memories.go:15`). Validated against `validMemoryTypes` from
`remember.go:21-27`. Unknown types return `invalid memory type %q —
valid types: feedback, project, user, reference, general`.

### Behavior

`runMemories` on `memories.go:43-132`:

1. **List all kv** via `bdKvListJSON()` (`memories.go:44-47`, primitive
   on `remember.go:219-231`) — shells out to `bd kv list --json`.
2. **Normalize search** to lowercase (`memories.go:50-53`).
3. **Validate `--type`** against `validMemoryTypes`.
4. **Filter** (`memories.go:69-89`): iterate every kv entry, skip any
   key that doesn't start with `memoryKeyPrefix = "memory."` (defined
   on `remember.go:16`). For each:
   - Extract `memType`, `shortKey` via `parseMemoryKey(k)` from
     `remember.go:117-133` (which handles both typed and legacy-untyped
     kv keys).
   - Apply `--type` filter.
   - Apply case-insensitive substring search across `shortKey`, value,
     and `memType`.
5. **Sort** (`memories.go:91-96`): by `memTypeRank` first — using
   `memoryTypeOrder = ["feedback", "user", "project", "reference",
   "general"]` from `remember.go:31` — then by short key ascending.
   This is the same order used by `gt prime` injection.
6. **Render**:
   - No matches → friendly message depending on whether a search term,
     type filter, or neither was given (`memories.go:98-107`).
   - Otherwise, print bold header like `Memories [feedback] matching
     "foo" (3):`, then per-type groups with `[feedback]` dim labels
     between groups (`memories.go:119-129`).

### Subcommands

None.

### Default sort order

Within the printed output, types appear in the priority order used by
`gt prime`'s context injection: `feedback` → `user` → `project` →
`reference` → `general`. This means the most-behavioral memories
(`feedback`) come first in both the CLI listing and the prime injection.

## Related commands

- [remember](remember.md) — the write side of the trio. Defines the
  type map, key prefix, and `parseMemoryKey` used here.
- [forget](forget.md) — the delete side of the trio.
- [prime](prime.md) — injects stored memories into agent context at
  session start; the same `memoryTypeOrder` governs injection priority.
- [bead](bead.md) — the underlying `bd kv *` plumbing this command
  shells out to.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Lists stored memories from beads. Read-only query with error propagation.

## Notes / open questions

- **Shell-out per listing.** Every `gt memories` invocation shells out
  to `bd kv list --json`, parses the full JSON map, then filters
  in-process. For a workspace with hundreds of memories this is still
  fast, but scales linearly with total kv size (not memory-prefixed
  subset).
- **Search matches `memType` too.** `gt memories feedback` returns every
  memory whose type is `feedback` — same result as `gt memories --type
  feedback` — plus any memory whose key or value contains the string
  `"feedback"`. That overlap is probably intentional but could be
  surprising.
- **Legacy untyped memories**. `parseMemoryKey` treats any
  `memory.<key>` without a recognized-type first segment as `general`.
  If a future type name collides with an existing untyped key prefix,
  those entries would silently reclassify.
- **No JSON output** — unlike [krc stats](krc.md) and [health](health.md),
  `gt memories` has no `--json` mode. Machine consumers must parse the
  human-readable output or call `bd kv list --json` directly.
