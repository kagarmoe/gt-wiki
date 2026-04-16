---
title: gt remember
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/remember.go
tags: [command, work, memory, beads, kv-store]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt remember

Store a persistent memory in the beads key-value store — the write
side of the memory trio with [memories](memories.md) and
[forget](forget.md). Also the source file that defines the shared
memory type registry, key prefix, and parse/sanitize helpers.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") — set on `remember.go:39`
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Batch-plan correction:** `gt remember` is in `GroupWork`, not
> ungrouped. See the same note on [commit](commit.md).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/remember.go:43-113`.

### Invocation

```
gt remember "insight" [--key <slug>] [--type feedback|project|user|reference]
```

`Use: remember "insight"` (`remember.go:44`). `Args:
cobra.ExactArgs(1)` (`remember.go:65`). Exactly one positional — the
memory content.

### Flags

Declared in `init()` on `remember.go:36-41`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--key` | string | `""` | Explicit key slug (default: auto-generated from content) |
| `--type` | string | `""` | Memory type: `feedback`, `project`, `user`, `reference` (default: `general`) |

### Shared registry (`remember.go:16-31`)

```go
const memoryKeyPrefix = "memory."

var validMemoryTypes = map[string]string{
    "feedback":  "Guidance or corrections from users ...",
    "project":   "Ongoing work context, goals, deadlines, decisions",
    "user":      "Info about the user's role, preferences, expertise",
    "reference": "Pointers to external resources (URLs, tools, dashboards)",
    "general":   "Uncategorized memories (default)",
}

var memoryTypeOrder = []string{"feedback", "user", "project", "reference", "general"}
```

- **`memoryKeyPrefix`** is the namespace every memory kv key lives
  under (`memory.<type>.<slug>` or legacy `memory.<slug>`).
- **`validMemoryTypes`** is the closed set of types (exactly five).
  Users cannot invent a new type.
- **`memoryTypeOrder`** is the injection priority for `gt prime`:
  behavioral corrections first, then user context, then the rest.
  Also used by [memories](memories.md) for display order and by
  [forget](forget.md) for lookup precedence.

### Behavior

`runRemember` on `remember.go:69-113`:

1. **Validate content non-empty** (`remember.go:71-73`).
2. **Validate / default `--type`** (`remember.go:76-84`): lowercase and
   trim; if set, must be in `validMemoryTypes`; if empty, default to
   `general`.
3. **Derive key** (`remember.go:86-89`): use `--key` if provided,
   otherwise `autoKey(content)`.
4. **Sanitize key** via `sanitizeKey(key)` (`remember.go:92`, body on
   `remember.go:172-192`).
5. **Compose full key** = `memory.<type>.<key>` (`remember.go:94`).
6. **Detect existing** via `bdKvGet(fullKey)` — pick verb
   `Stored` vs. `Updated` (`remember.go:97-101`).
7. **Write** via `bdKvSet(fullKey, content)` (`remember.go:103-105`,
   `bd kv set` shell-out on `remember.go:195-199`).
8. **Print** `✓ <verb> memory: <displayKey>` via `style.Success.Render`.
   `displayKey` is `<type>/<key>` unless `type == general`, in which
   case it's just `<key>`.

### `autoKey` (`remember.go:136-169`)

1. Split content on whitespace, lowercase, take up to 5 words.
2. Strip any non-`[a-z0-9]` character from each word.
3. Drop empty results.
4. If nothing survives, hash the content with SHA-256 and use the first
   4 bytes as hex (`remember.go:158-161`) — the fallback key for
   unicode/symbol-only content.
5. Join with `-`, cap length at 40 chars.

### `sanitizeKey` (`remember.go:172-192`)

1. Lowercase.
2. Spaces → `-`, dots → `-`.
3. Strip everything that isn't `[a-z0-9-]`.
4. Collapse runs of `-` into a single `-`.
5. Trim leading/trailing `-`.

Guarantees that user-provided `--key` values become valid kv identifiers.

### `parseMemoryKey` (`remember.go:117-133`)

Inverse of `<prefix><type>.<shortkey>`. Handles two cases:

1. **Typed** — first segment after `memory.` is in `validMemoryTypes`
   → returns `(type, restAfterFirstDot)`.
2. **Legacy untyped** — no recognized type prefix → returns `("general",
   restAfterPrefix)`.

This is called by both [memories](memories.md) and [forget](forget.md)
during their own filter passes.

### `bd kv` shell-outs

Four helpers at `remember.go:194-231`:

- `bdKvSet(key, value)` → `bd kv set <key> <value>`.
- `bdKvGet(key)` → `bd kv get <key>`; returns trimmed stdout.
- `bdKvClear(key)` → `bd kv clear <key>`.
- `bdKvListJSON()` → `bd kv list --json`; parses to `map[string]string`.

## Related commands

- [memories](memories.md) — the read/search side. Uses
  `validMemoryTypes`, `memoryKeyPrefix`, `parseMemoryKey`,
  `memTypeRank`.
- [forget](forget.md) — the delete side. Uses the same registry.
- [prime](prime.md) — injects stored memories into agent context at
  session start; `memoryTypeOrder` dictates injection order.
- [bead](bead.md) — the underlying `bd kv *` primitives this command
  delegates to.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **The five-type closed set is defined exactly once**, in this file.
  Anywhere in the codebase that sees a memory type must import it from
  here — worth verifying no parallel type lists exist elsewhere.
- **Auto-key collision risk.** Two different content strings that share
  the first 5 cleaned words yield the same slug. `gt remember` silently
  overwrites (printing `Updated`) when this happens. Explicit `--key`
  is safer for anything you want to refer to later.
- **Not transactional with `gt prime`.** A `gt remember` mid-session
  does not retroactively inject into the running agent's context —
  the memory only takes effect on the *next* `gt prime`. This is by
  design but easy to forget.
- **Shell-out overhead.** Every `gt remember` is at least two `bd kv`
  shell-outs (one `get`, one `set`). Bulk-import use cases should
  batch through `bd` directly.
