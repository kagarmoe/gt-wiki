---
title: internal/atomicfile
type: package
status: verified
topic: gastown
created: 2026-05-20
updated: 2026-05-20
sources:
  - /home/kimberly/repos/gastown/internal/atomicfile/atomicfile.go
tags: [package, platform-service, filesystem, atomic-write, json, post-v1.0.0]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# internal/atomicfile

A single-file leaf package providing atomic file-write primitives
(write-to-temp + rename). New in v1.0.1; it is the dependency-free successor
to the `AtomicWrite*` helpers that previously lived in [internal/util](util.md).

**Go package path:** `github.com/steveyegge/gastown/internal/atomicfile`
**File count:** 1 go file (`atomicfile.go`, 90 lines), 1 test file
**Imports (notable):** stdlib only (`encoding/json`, `os`, `path/filepath`)
**Imported by (notable):** broadly consumed — [agent](agent.md)
(`internal/agent/state.go`), [config](config.md) (`internal/config/loader.go`),
[crew](crew.md), [daemon](daemon.md) (`internal/daemon/types.go`),
[doctor](doctor.md), [dog](dog.md), [doltserver](doltserver.md),
[polecat](polecat.md) (`internal/polecat/namepool.go`), and [quota](quota.md),
among others.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/atomicfile/atomicfile.go:1-90`.

Every write follows the same crash-safe pattern: write to a uniquely-named
temp file in the **same directory** as the target, then `os.Rename` it over
the destination. The rename is atomic on POSIX systems, so a crash mid-write
never leaves a partially-written target — a reader sees either the old file
or the complete new one.

### Public API

- `atomicfile.WriteFile(path string, data []byte, perm os.FileMode) error`
  (`atomicfile.go:56-89`) — the core primitive. Creates a temp file via
  `os.CreateTemp(dir, base+".tmp.*")` (the `*` gets a random suffix so
  concurrent writers never collide), writes the data, `Close`s, `Chmod`s to
  the caller's `perm` (CreateTemp defaults to 0600), then `os.Rename`s over
  the target. On any error it removes the temp file before returning.
- `atomicfile.WriteJSON(path string, v interface{}) error`
  (`atomicfile.go:17-24`) — marshals `v` with `json.MarshalIndent` (two-space
  indent) and calls `WriteFile` with mode `0644`.
- `atomicfile.WriteJSONWithPerm(path, v, perm)` (`atomicfile.go:26-34`) — as
  `WriteJSON` but with a caller-supplied mode.
- `atomicfile.EnsureDirAndWriteJSON(path, v)` (`atomicfile.go:36-43`) —
  `os.MkdirAll(dir, 0755)` then `WriteJSON`.
- `atomicfile.EnsureDirAndWriteJSONWithPerm(path, v, perm)`
  (`atomicfile.go:45-53`) — directory creation plus a custom file mode.

### Why this exists as its own package

The package doc-comment (`atomicfile.go:1-5`) is explicit: it is kept as a
**leaf package with no internal dependencies** so that even low-level
packages — including ones that `internal/util` transitively depends on — can
write atomically without creating an import cycle. The previous home for this
logic, `internal/util/atomic.go`, could not be imported by those low-level
packages without a cycle.

## Relationship to internal/util (the move)

In the v1.0.0 baseline these primitives lived in
`internal/util/atomic.go` under different names. v1.0.1 moved them into this
leaf package and renamed the write entry points:

| Old (`util`, ≤ v1.0.0) | New (`atomicfile`, v1.0.1+) |
|---|---|
| `util.AtomicWriteJSON` | `atomicfile.WriteJSON` |
| `util.AtomicWriteJSONWithPerm` | `atomicfile.WriteJSONWithPerm` |
| `util.AtomicWriteFile` | `atomicfile.WriteFile` |
| `util.EnsureDirAndWriteJSON` | `atomicfile.EnsureDirAndWriteJSON` (name kept) |
| `util.EnsureDirAndWriteJSONWithPerm` | `atomicfile.EnsureDirAndWriteJSONWithPerm` (name kept) |

`internal/util/atomic.go` was deleted in the same change. See [util](util.md).

## Failure modes

- **Marshal failure** (`WriteJSON*`): a non-serialisable value returns the
  `json.Marshal` error before any file is touched.
- **Temp-file or write failure** (`WriteFile`): the temp file is removed and
  the error is returned; the target is never modified.
- **Cross-device rename:** because the temp file is created in the target's
  own directory, `os.Rename` stays within one filesystem and avoids
  `EXDEV`. Pointing a write at a path whose directory does not exist still
  fails at `CreateTemp` unless the `EnsureDir*` variants are used.

## Related wiki pages

- [internal/util](util.md) — former home of these primitives; still holds the
  rest of the shared utility surface.
- [go-packages inventory](../inventory/go-packages.md) — package enumeration.
- Consumers: [config](config.md), [daemon](daemon.md), [doctor](doctor.md),
  [polecat](polecat.md), [quota](quota.md), [doltserver](doltserver.md).

## Notes / open questions

- New in v1.0.1; not yet reached by a Phase 3 drift audit (no `phase3_*`
  frontmatter). Created during the v1.0.0 → v1.1.0 release sync (Stage A).
- The atomic-rename guarantee is POSIX-specific, as the doc-comment notes;
  Windows rename-over-existing semantics differ but are not exercised by the
  package's own code.
