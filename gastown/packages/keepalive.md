---
title: internal/keepalive
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/keepalive/keepalive.go
tags: [package, diagnostics, keepalive, daemon, sentinel-pattern]
phase3_audited: 2026-04-14
phase3_findings: [wiki-stale, implementation-status-partial]
phase3_severities: [wrong, incomplete]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 2}
---

# internal/keepalive

Best-effort agent-activity signaling via a single JSON file at
`<townRoot>/.runtime/keepalive.json`. Every `gt` invocation touches this
file so the daemon can tell whether an agent is actively running commands
or idle.

**Go package path:** `github.com/steveyegge/gastown/internal/keepalive`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib only (`encoding/json`, `os`, `path/filepath`,
`strings`, `time`) plus [`internal/workspace`](workspace.md) for town-root
discovery.
**Imported by:** **no in-tree importers at HEAD.** `rg '"github.com/steveyegge/gastown/internal/keepalive"'` returns zero results. The expected `gt`-root persistent prerun integration that would call `Touch` on every command does not exist in the import graph (see "Notes / open questions"). The package is defined but unused.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/keepalive/keepalive.go`.

### Public API

**Type:**

- `State struct { LastCommand string; Timestamp time.Time }`
  (`keepalive.go:46-49`) — the wire format persisted as
  `keepalive.json`. `last_command` is a free-form string, not a
  structured `{cmd, args}` tuple.

**Touch functions (writer side):**

- `Touch(command string)` (`keepalive.go:53-55`) — thin convenience
  wrapper that calls `TouchWithArgs(command, nil)`.
- `TouchWithArgs(command string, args []string)`
  (`keepalive.go:59-72`) — resolves the town root via
  `workspace.FindFromCwd`, silently returns when not inside a workspace,
  and then builds a space-joined `"command arg1 arg2 ..."` string before
  delegating to `TouchInWorkspace`. No shell-quoting.
- `TouchInWorkspace(workspaceRoot, command string)`
  (`keepalive.go:76-96`) — ensures `<root>/.runtime/` exists, marshals
  `State{LastCommand: command, Timestamp: time.Now().UTC()}`, and
  `os.WriteFile`s it to `<root>/.runtime/keepalive.json` with mode
  `0644`. Every error path is silently swallowed with `return` — no
  logging, no panic, no error return. Comment at `keepalive.go:95`:
  "non-fatal: status file for debugging".

**Read side:**

- `Read(workspaceRoot string) *State` (`keepalive.go:103-117`) — reads
  and unmarshals `<root>/.runtime/keepalive.json`. **Returns `nil` (not
  an error) when the file doesn't exist, can't be read, or contains
  invalid JSON.** This is the nil-sentinel pattern documented at length
  in the package doc comment.
- `(s *State) Age() time.Duration` (`keepalive.go:132-137`) — `nil`-
  receiver-safe. On a non-nil `State` it returns `time.Since(s.Timestamp)`.
  **On a nil receiver it returns `24 * time.Hour * 365`**, a sentinel
  "maximally stale" value chosen specifically to exceed any practical
  idle threshold without risking `MaxInt64` overflow arithmetic.

### Internals / Notable implementation

- **The nil-sentinel pattern is the load-bearing design choice.** The
  package doc (`keepalive.go:14-32`) explicitly motivates it: daemon code
  wants to write `if keepalive.Read(root).Age() > 5*time.Minute { ... }`
  without a nil guard, treating "no keepalive" and "stale keepalive" as
  the same idle condition. Returning `365 days` from `Age()` on a nil
  receiver makes that idiom compile and do the right thing.
- **All writes are fire-and-forget.** The rationale at `keepalive.go:5-9`:
  keepalive signals are non-critical; failures (disk full, permission
  errors) should not interrupt `gt` commands; the daemon already tolerates
  missing signals via timeout. This is why `TouchInWorkspace` has three
  separate silent-`return` error paths and one `_ = os.WriteFile(...)`
  explicit discard.
- **Timestamps are UTC** (`keepalive.go:86`). The daemon's consumer side
  must also work in UTC to avoid tz-drift false positives.

### Usage pattern

Writer side: any CLI command or daemon-adjacent tool that wants to assert
"an agent was active" would call `keepalive.Touch("command name")`. The
intended integration point is the root `gt` cobra `PersistentPreRun`, so
that **every** subcommand implicitly signals activity — but as of HEAD
there are **no in-tree importers** of this package. The Phase 2 note
that `internal/web/api.go` imports it was incorrect: `web/api.go` uses
a local `keepalive` variable for SSE heartbeat ticking, not the
`internal/keepalive` package.

Reader side: the daemon loop (`internal/daemon/*`) and doctor
(`internal/doctor/*`) have been mapped (Batches 7-8) and **neither**
imports `internal/keepalive`. The `Read`/`Age()` API has zero consumers.

## Implementation status

- **Status:** `partial`
- **Severity:** `incomplete`
- **Docs describe:** Package doc and API design assume `Touch` is called from every `gt` invocation via `PersistentPreRun`, with `Read`/`Age()` consumed by the daemon for idle detection. The nil-sentinel pattern (`Age()` returning 365 days on nil receiver) is designed for zero-guard daemon consumption.
- **Code provides:** The package itself is fully implemented (`keepalive.go:46-137`) with Touch/Read/Age API. However, there are zero importers anywhere in the codebase — neither the writer side (`PersistentPreRun`) nor the reader side (`internal/daemon`, `internal/doctor`) has been wired up. The package is a complete library with no consumers.
- **Release position:** `in-release`
- **Fix tier:** `preserve-as-vision`
- **Resolution:** Preserve as vision doc. The package is ready to use; the integration just hasn't been wired. Not a bug, not dead code — it's waiting for the `PersistentPreRun` hookup and daemon-side consumption.

## Related wiki pages

- [gt](../binaries/gt.md) — the binary expected to call `Touch` on every
  invocation.
- [internal/workspace](workspace.md) — used to locate the town root
  before writing the keepalive file.
- [internal/doctor](doctor.md) — the diagnostic package; `Age()` was
  designed to feed into doctor-style threshold checks but doctor does
  not currently import this package.
- [gt doctor](../commands/doctor.md), [gt health](../commands/health.md),
  [gt vitals](../commands/vitals.md) — sibling diagnostic CLI surfaces.
- [go-packages inventory](../inventory/go-packages.md).

## Outgoing calls

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| keepalive status file | JSON data | Keepalive status for debugging | `keepalive.go:95` |

## Notes / open questions

- **Zero importers at HEAD.** Phase 2 stated `internal/web/api.go`
  imported this package. That was incorrect — `web/api.go` uses a local
  `keepalive` variable for SSE heartbeat ticking (`time.NewTicker(15 *
  time.Second)`), not the `internal/keepalive` package. The grep for
  `"github.com/steveyegge/gastown/internal/keepalive"` returns zero
  results. The `Touch`/`Read`/`Age()` API is entirely unused in the
  codebase. **wiki-stale finding (phase-2-incomplete):** Phase 2
  mistook a local variable name for a package import.
- **No daemon consumer of `Read` either.** `internal/daemon/` and
  `internal/doctor/` have been mapped (Batches 7-8) and neither imports
  `internal/keepalive`. The 365-day nil-sentinel pattern and the
  `Read`/`Age()` API have zero consumers anywhere in the import graph.
- The full-command string built by `TouchWithArgs` is not shell-quoted
  and is stored as-is. Arguments containing spaces or newlines will
  corrupt the `last_command` field. Acceptable because this field is
  documented as debugging-only, but worth noting if anyone ever tries
  to reconstruct invocations from it.
