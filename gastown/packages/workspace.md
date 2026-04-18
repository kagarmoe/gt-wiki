---
title: internal/workspace
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/workspace/find.go
tags: [package, platform-service, workspace, discovery, filesystem]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [precondition]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 1}
---

# internal/workspace

Locates the Gas Town "town root" — the outermost directory containing a
`mayor/town.json` marker — by walking up from an arbitrary starting
directory. The single entry point every command uses to figure out "am I
inside a Gas Town, and if so, where does it live?"

**Go package path:** `github.com/steveyegge/gastown/internal/workspace`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib (`errors`, `os`, `path/filepath`), plus
[`internal/config`](config.md) for `LoadTownConfig` used by `GetTownName`.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command's
`persistentPreRun` and essentially every subcommand that needs to resolve
`townRoot` (seen wherever a command page mentions "locates the town root").
`config` is the only internal dependency; the `session` and `tmux`
packages then sit on top of the `townRoot` that `workspace` returns.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/workspace/find.go`.

### Public API

**Errors and constants:**

- `workspace.ErrNotFound` — sentinel error "not in a Gas Town workspace"
  (`find.go:14`).
- `workspace.PrimaryMarker = "mayor/town.json"` (`find.go:20`) — the
  canonical workspace marker file. `town.json` lives under `mayor/` together
  with other town-level config.
- `workspace.SecondaryMarker = "mayor"` (`find.go:25`) — fallback marker.
  Matches any directory containing a `mayor/` subdirectory, which is
  sometimes true for rig-level structures too; the Find algorithm keeps
  walking upward after a secondary hit so that the outermost town wins.

**Discovery functions:**

- `Find(startDir string) (string, error)` — walks up from `startDir`
  looking for the outermost directory containing either marker
  (`find.go:33-64`). Returns `("", nil)` when no marker is found anywhere on
  the walk up to the filesystem root — this is not an error. Absolute-path
  normalization happens first via `filepath.Abs`. **Symlinks are not
  resolved** (explicit comment at `find.go:32`), to stay consistent with
  `os.Getwd()`.
- `FindOrError(startDir string) (string, error)` — same as `Find` but
  converts the `("", nil)` "not found" case into
  `ErrNotFound` (`find.go:67-76`).
- `FindFromCwd() (string, error)` — convenience wrapper that calls
  `os.Getwd()` then `Find` (`find.go:79-85`).
- `FindFromCwdOrError() (string, error)` — the canonical startup call
  (`find.go:90-113`). Uses CWD-based walk first, then **falls back to
  `GT_TOWN_ROOT` and `GT_ROOT` env vars** if CWD walk fails. The fallback
  verifies the env-var path actually contains a workspace marker before
  accepting it. Finally returns `ErrNotFound` if both strategies fail.
- `FindFromCwdWithFallback() (townRoot, cwd string, err error)` — the
  most tolerant variant (`find.go:119-137`). Designed for callers like `gt
  done` that need to keep running even after their working directory has
  been deleted out from under them (comment: "polecat worktree nuked by
  Witness"). Falls back to `GT_TOWN_ROOT` when `os.Getwd()` itself errors,
  and returns `townRoot="valid"`, `cwd=""`, `err=nil` in that specific
  case.

**Workspace validation and town-name lookup:**

- `IsWorkspace(dir string) (bool, error)` — does `dir` itself qualify as a
  town root? Checks for `mayor/town.json` first, then `mayor/` directory
  (`find.go:142-162`).
- `GetTownName(townRoot string) (string, error)` — reads
  `<townRoot>/mayor/town.json` via `config.LoadTownConfig` and returns the
  `name` field (`find.go:167-174`). This is the identifier used by
  [`internal/session`](session.md) for generating tmux session names that
  don't collide across multiple Gas Town instances on one machine.
- `GetTownNameFromCwd() (string, error)` — convenience combo of
  `FindFromCwdOrError` + `GetTownName` (`find.go:178-184`).
- `MustGetTownName(townRoot string) string` — panicking variant used by
  code paths where a misconfigured town indicates a programming error
  rather than a runtime condition to report (`find.go:188-194`).

### Internals / Notable implementation

- The upward-walk loop always continues to the filesystem root rather than
  stopping at the first hit (`find.go:42-62`). This is load-bearing for
  nested workspace support: a rig or worktree may contain its own
  `mayor/town.json`, but commands always want the outermost `town.json`
  (the actual town), not the inner one.
- The primary/secondary distinction exists because `mayor/` alone
  sometimes matches rig-level structures. Primary (`mayor/town.json`) wins
  over secondary (`mayor/` dir) at resolution time (`find.go:57-61`).

### Usage pattern

Every command that needs to know where the town lives calls one of the
`FindFromCwd*` variants during its setup phase. The returned `townRoot`
then gets threaded down into [`config.LoadTownConfig`](config.md),
`session.InitRegistry`, and paths like
`<townRoot>/.runtime/pids/`. No caching — each call re-walks the
filesystem.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary whose persistent preruns
  bootstrap from `workspace.FindFromCwdOrError`.
- [internal/config](config.md) — consumed via
  `config.LoadTownConfig(<townRoot>/mayor/town.json)`.
- [internal/session](session.md) — downstream consumer that uses the
  resolved `townRoot` as the base for PID tracking and session name
  generation.
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Precondition violations (what does it assume?)
- **`MustGetTownName` panics on config error:** `find.go:188-194` calls
  `panic(fmt.Sprintf(...))` when `GetTownName` returns an error. If
  `mayor/town.json` is corrupt or unreadable, this crashes the process
  with no recovery path. **Absent** — callers that use `MustGetTownName`
  have no opportunity to handle the error; a corrupted `town.json` takes
  down the entire `gt` process rather than producing a user-facing error.
- **`Find` walks to filesystem root:** `find.go:42-62` walks upward
  without depth limit. On deeply nested mounts or unusual filesystem
  layouts, this traverses the entire path to `/`. **Present** — the loop
  terminates when `parent == current` (root reached), so it cannot
  infinite-loop, but there is no timeout or depth cap.

## Notes / open questions

- `GT_TOWN_ROOT` and `GT_ROOT` are treated as equivalent fallbacks
  (`find.go:100-107`). There is no documented precedence between them —
  they are tried in that list order. Possible drift if downstream code
  treats one as canonical and the other as legacy.
- The comment at `find.go:30-32` explicitly notes symlinks are not
  resolved. If a user's town lives behind a symlink, commands that
  stringify `townRoot` for logging or for comparing against tmux session
  attributes may produce path mismatches.
