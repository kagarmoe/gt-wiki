---
title: gt forget
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/forget.go
  - /home/kimberly/repos/gastown/internal/cmd/remember.go
tags: [command, work, memory, beads, kv-store]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt forget

Remove a stored memory from the beads key-value store. One-third of the
memory trio with [remember](remember.md) and [memories](memories.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") — set on `forget.go:12`
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

> **Batch-plan correction:** `gt forget` is in `GroupWork`, not
> ungrouped. See the same note on [commit](commit.md).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/forget.go:16-86`.

### Invocation

```
gt forget <key>
```

`Use: "forget <key>"` (`forget.go:17`). `Args: cobra.ExactArgs(1)`
(`forget.go:28`). No flags.

### Behavior

`runForget` on `forget.go:32-86`:

1. **Strip `memory.` prefix** if the user included it
   (`forget.go:36`). The full key in the kv store is
   `memory.<type>.<shortkey>`, but users typically reference the short
   form shown by `gt memories`.
2. **Type/key form** (`forget.go:39-54`): if the key contains `/`, split
   on the first `/`. If the left side is a recognized memory type
   (checked against `validMemoryTypes` from
   [remember.go:21-27](remember.md)), build
   `memory.<type>.<shortkey>`, verify it exists via `bdKvGet`, clear it
   via `bdKvClear`, and report `Forgot memory: <type>/<shortkey>`.
3. **Typed-first lookup** (`forget.go:57-71`): iterate
   `memoryTypeOrder = ["feedback", "user", "project", "reference",
   "general"]` from `remember.go:31`. For each type, build
   `memory.<type>.<key>`, call `bdKvGet`, and if the value is non-empty,
   `bdKvClear` it. Display key is prefixed with `<type>/` unless the
   type is `general`.
4. **Legacy untyped fallback** (`forget.go:74-85`): try
   `memory.<key>` (the pre-typed-memory format). If found, clear it;
   otherwise return `memory %q not found`.
5. On success, prints `✓ Forgot memory: <displayKey>` via
   `style.Success.Render`.

### Subcommands

None.

### Flags

None.

### Example keys

From the `Long` help on `forget.go:20-28`:

```
gt forget refinery-worktree
gt forget feedback/dont-mock-db
gt forget hooks-package-structure
```

The `feedback/dont-mock-db` form exercises the type/key path; the other
two exercise the typed-first-lookup path (searching every type in
order).

### Key-store primitives

`bdKvClear` and `bdKvGet` shell out to `bd kv clear` / `bd kv get`
respectively — defined in `remember.go:202-216`. They are the same
primitives `remember` and `memories` call.

## Related commands

- [remember](remember.md) — the write side; defines `memoryKeyPrefix`,
  `validMemoryTypes`, and `memoryTypeOrder` (the structures this
  command walks).
- [memories](memories.md) — the list/search side of the trio.
- [prime](prime.md) — injects memories into agent context at session
  start; the injection order is `memoryTypeOrder`.
- [bead](bead.md) — the underlying issue/kv plumbing (`bd kv *`).
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Three lookup strategies, no error leak.** If the type/key form is
  given but the type is unrecognized, the command silently falls
  through to the typed-first lookup — which may succeed unexpectedly
  if a memory with that literal key exists under some other type. A
  user typing `feedback/typo` could wind up deleting `general/feedback`
  if that existed and `feedback` is not in `validMemoryTypes`. (In
  practice `feedback` is always in the map, so this is theoretical.)
- **`memoryTypeOrder` determines priority** for both the lookup and
  `gt prime`'s injection — whichever type is matched first wins. If a
  short key exists under two types, only the higher-priority one is
  deleted per `gt forget` invocation.
- **Not transactional.** Each `bd kv clear` is a separate beads
  operation. A mid-flight failure could leave a partially-deleted state
  — though typical usage removes a single kv entry.
