---
title: gt close
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/close.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, bd-wrapper, convoy, cascade]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt close

Close one or more beads (wrapper for `bd close`) with two gt-specific
extensions: depth-first `--cascade` child-closing, and post-close
convoy-completion propagation.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`close.go:21`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/close.go:19-275`.

### Invocation

```
gt close [bead-id...] [--cascade] [--comment|--reason <msg>] [--force] [bd flags...]
```

`DisableFlagParsing: true` at `close.go:39`, so all flags are raw and
forwarded to `bd close` unless the command handles them explicitly.

### Behavior

`runClose` (`close.go:47-108`):

1. **Handle `--help`** manually since `DisableFlagParsing` bypasses
   cobra's help handler (`close.go:49-51`).
2. **Extract `--cascade`** via `extractCascadeFlag` — this is the
   only gt-specific flag. Returns `(cascade bool, filteredArgs)`
   (`close.go:54`, `close.go:111-122`).
3. **Convert `--comment` → `--reason`** — in both `--comment` and
   `--comment=<value>` forms (`close.go:57-66`). This gives `gt close
   --comment "..."` an alias form for `bd close --reason "..."`.
4. **If `--cascade`**, close children first (depth-first)
   (`close.go:69-77`):
   - Extract bead IDs from the raw args.
   - For each top-level ID, call `closeChildren(id, visited, 0)`.
5. **Exec `bd close`** with the converted args forwarded, plus
   stdin/stdout/stderr wired through (`close.go:84-97`). Routes to the
   owning rig's directory via `resolveBeadDir(beadIDs[0])` and strips
   `BEADS_DIR` from the environment (comment: "bd no longer does
   cross-rig routing internally (removed in beads v0.62)").
6. **After a successful close**, invoke `checkConvoyCompletion` for
   every bead ID that was closed (`close.go:99-105`) — see below.

### `--cascade` depth-first child close (`close.go:137-203`)

Bounded recursion with two guard conditions:

- `maxCascadeDepth = 50` (`close.go:132`) — errors if the recursion
  goes deeper than 50 levels.
- `visited map[string]bool` prevents cycles; already-seen IDs are
  silently skipped.

At each level:

1. Run `bd children <parent> --json` with `resolveBeadDir(parent)` as
   `Dir` (`close.go:148-152`). Failures only print a warning.
2. Decode `[]childBead` (`close.go:125-128` — just `ID` and `Status`).
3. Skip children already `"closed"`.
4. **Recurse first** (depth-first) on each open child (`close.go:178`).
5. Then run `bd close <childIDs...> --reason "Parent <parentID>
   closed (cascade)" --force` in a single batched invocation
   (`close.go:189-202`).

### Convoy completion propagation (`close.go:249-275`)

`checkConvoyCompletion` implements the "ZFC principle: the closure
event propagates at the source (bd close) rather than relying solely
on daemon event polling" (`close.go:244-247`):

1. Find town root (bail silently on error).
2. Compute `hqBeadsDir = <townRoot>/.beads`.
3. Open the HQ beads store via `beadsdk.Open(ctx, hqBeadsDir)` from
   `github.com/steveyegge/beads`.
4. Resolve the `gt` binary path via `os.Executable()` with fallback to
   `exec.LookPath("gt")`.
5. For each closed bead, call `convoy.CheckConvoysForIssue(ctx, store,
   townRoot, beadID, "Close", nil, gtPath, nil)` from
   `internal/convoy`.

This is best-effort — if the workspace lookup, store open, or gt
resolution fails, the function returns silently and the daemon's
event polling + deacon patrol serve as the backup path.

### `extractBeadIDs` flag-parser (`close.go:207-240`)

Since `DisableFlagParsing: true`, the command must manually find
bead-ID positionals while skipping flag values. The list of
value-taking flags is hard-coded:

```go
valueFlags := map[string]bool{
    "--reason": true, "-r": true,
    "--session": true,
    "--actor": true,
    "--db": true,
    "--dolt-auto-commit": true,
    "--comment": true,
}
```

### Subcommands

None.

### Flags

`DisableFlagParsing: true` means cobra doesn't register flags. The
gt-specific extensions handled before forwarding are `--cascade`,
`--comment[=value]`, `--reason[=value]` (forwarded), and all other bd
flags. The `Long` text at `close.go:23-38` calls out `--force` and
`--cascade`; everything else is bd-specific.

### Related commands

- [done](done.md) — the polecat-side of work completion. `gt close`
  is for general bead closure; `gt done` is for polecats signaling
  merge-queue-ready work.
- [cat](cat.md), [bead](bead.md) — sibling bd-wrappers.
- [compact](compact.md) — also calls bd (via `bd delete` for expired
  closed wisps).
- [convoy](convoy.md) — the convoy subsystem that receives completion
  events via `checkConvoyCompletion`.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Convoy completion check entirely swallowed:** `close.go:99-105` — `checkConvoyCompletion` is called after successful close, but the function at `close.go:249-275` silently returns on every error path (workspace not found, store open failure, executable lookup failure). No warning is emitted. **Absent** — if convoy completion detection fails, the convoy sits open indefinitely until the daemon's backup polling catches it.
- **Cascade child query failure swallowed:** `close.go:155-159` — if `bd children --json` fails, a warning is printed to stderr but execution continues, potentially leaving children open when the parent closes. **Present** — warning emitted.
- **Cascade child JSON parse failure swallowed:** `close.go:161-164` — if JSON parse of children output fails, warning printed and returned nil (no error). **Present** — warning emitted, cascade silently skipped.

## Notes / open questions

- **`--comment` alias doc lives only in the Long text.** Since
  `DisableFlagParsing: true` skips cobra flag registration, there's
  no `Flag:` documentation for `--comment`; users only learn about it
  from `gt close --help`'s `Long` section (`close.go:23-38`).
- **ZFC principle** — the comment at `close.go:243-248` references a
  design principle ("the closure event propagates at the source")
  without defining it. Worth a separate concept page to pin down.
- **Cross-database `beadsdk.Open`** — this file directly imports
  `github.com/steveyegge/beads` and opens a Dolt connection to the
  HQ database. Unlike most bd-wrappers that shell out, this one has
  a native library dependency on beads for the convoy hook.
