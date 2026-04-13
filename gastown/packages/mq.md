---
title: internal/mq
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/mq/id.go
tags: [package, data-layer, merge-queue, id-generation]
---

# internal/mq

Thin "merge queue" helper package. A single file provides merge-request ID
generation for the merge-queue workflow driven by the
[`gt mq`](../commands/mq.md) command and the refinery.

**Go package path:** `github.com/steveyegge/gastown/internal/mq`
**File count:** 1 non-test go file (`id.go`, 59 lines), 1 test file
(`id_test.go`).
**Imports (notable):** stdlib only — `crypto/rand`, `crypto/sha256`,
`encoding/hex`, `fmt`, `time`. No internal gastown dependencies.
**Imported by (notable):** Merge-queue command code and the refinery that
creates merge-request records. Despite its name, `internal/mq` is **not**
the implementation of a message queue — mail queuing lives in
[`internal/mail`](mail.md); this package is purely an ID minter for merge
requests.

## What it actually does

The package doc comment reads "Package mq provides merge queue
functionality" (`id.go:1`). The only thing it actually provides is
deterministic-with-entropy ID generation for merge requests. The rest of
the merge-queue machinery — queue state, branch rebasing, refinery lock,
etc. — lives in `internal/refinery` and in the merge-related commands.

### Public API

- `GenerateMRID(prefix, branch string) string` (`id.go:22-28`) —
  production generator. Combines `branch`, `time.Now().UnixNano()`, and
  8 random bytes from `crypto/rand`, hashes with SHA-256, and takes the
  first 10 hex chars. Returns `"<prefix>-mr-<10charhash>"`, e.g.
  `gt-mr-abc1234567`.
- `GenerateMRIDWithTime(prefix, branch string, timestamp time.Time)
  string` (`id.go:40-42`) — deterministic variant used by tests. Takes
  a caller-supplied timestamp and passes `nil` randomBytes so identical
  inputs produce identical IDs. **Not safe for production** — two calls
  with the same inputs collide.

### Internals / Notable implementation

`generateMRIDInternal` (`id.go:45-58`) is the shared implementation:

```go
input := fmt.Sprintf("%s:%d:%x", branch, timestamp.UnixNano(), randomBytes)
hash := sha256.Sum256([]byte(input))
hashStr := hex.EncodeToString(hash[:])[:10]
return fmt.Sprintf("%s-mr-%s", prefix, hashStr)
```

The comment at `id.go:55` notes why 10 hex chars were chosen:

> 10 hex chars = 40 bits → ~1.1 trillion values per namespace. Birthday
> paradox: ~1% collision at ~150K IDs (vs 577 with 6 chars).

So the prior scheme used 6 hex chars and was expanded after collision
risk was observed in practice. The `crypto/rand.Read` error is ignored
at `id.go:25` with the comment "crypto/rand.Read only fails on broken
system" — the fallback is that the hash still includes the branch name
and nanosecond timestamp, which are effectively unique for sequential
calls.

### Usage pattern

A caller (typically refinery or the `gt mq` command) creates a merge
request by minting an ID:

```go
mrID := mq.GenerateMRID("gt", "polecat/Nux/gt-xyz")
// → "gt-mr-abc1234567"
```

That ID becomes the key for a merge-request bead or merge-queue entry
stored in the [`internal/beads`](beads.md) data plane. The rest of the
workflow (rebasing, merging, status tracking) is handled elsewhere.

## Related wiki pages

- [gt](../binaries/gt.md) — hosts `gt mq`.
- [gt mq](../commands/mq.md) — user-facing merge queue command.
- [gt refinery](../commands/refinery.md) — the service that owns the
  merge queue loop and calls this package.
- [internal/beads](beads.md) — where merge-request records live once an
  ID has been minted.
- [internal/events](events.md) — emits `merge_started`, `merged`,
  `merge_failed`, `merge_skipped` events tagged by MR ID.
- [go-packages inventory](../inventory/go-packages.md)
- [go.mod](../files/go-mod.md) — stdlib-only; no external deps.

## Notes / open questions

- The `mr` in `mq-mr` stands for "merge request". The `mq` package name
  is the only place in the codebase that conflates "merge queue" with
  the two-letter abbreviation one might expect for "message queue"; the
  actual message-queue implementation is in
  [`internal/mail`](mail.md)'s Router (`sendToQueue` and related).
- The collision-math comment is a clean piece of commit archaeology:
  the expansion from 6 to 10 hex chars was a reaction to real collisions
  in the merge queue. Worth preserving if the ID format is ever changed
  again.
- Package is a one-file utility and will probably stay that way unless
  the ID scheme grows metadata (e.g., rig, actor). A larger merge-queue
  "library" lives in `internal/refinery` — not in `internal/mq`.
