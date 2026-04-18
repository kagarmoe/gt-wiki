---
title: internal/version
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/version/stale.go
tags: [package, platform-service, version, git, build]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [precondition, silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 1}
---

# internal/version

Provides the build-time commit hash for the `gt` binary and implements the
"is my binary stale vs. the source repo?" check that powers the
[`gt stale`](../commands/stale.md) command and various auto-rebuild
warnings. The entire package is one file, `stale.go`.

**Go package path:** `github.com/steveyegge/gastown/internal/version`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib `os`, `os/exec`, `runtime/debug`, `strings`,
plus [`internal/util`](util.md) for `SetDetachedProcessGroup`.
**Imported by (notable):** [`gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:20`), the [`gt
stale`](../commands/stale.md) command, and the [`gt
version`](../commands/version.md) command. The `cmd` package calls
`version.SetCommit` with the ldflags-injected hash at program startup.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/version/stale.go`.

### Public API

- `version.Commit` (package-level `var`) — holds the commit hash the
  binary was built from (`stale.go:18-19`). Intended to be set at build
  time via ldflags (`go build -ldflags '-X
  .../internal/version.Commit=<hash>'`). If unset, `resolveCommitHash`
  falls back to reading `vcs.revision` from `debug.ReadBuildInfo()`, which
  Go populates automatically for builds from a git worktree
  (`stale.go:33-47`).
- `version.SetCommit(commit string)` — setter the `cmd` package calls from
  its own ldflags glue so the hash flows through
  (`stale.go:250-252`).
- `version.ShortCommit(hash string) string` — returns the first 12
  characters of a hash (`stale.go:50-55`). Used for display in version
  output and error messages.
- `version.StaleBinaryInfo` struct — return value of `CheckStaleBinary`
  (`stale.go:22-30`). Fields: `IsStale`, `IsForward` (repo HEAD is a
  descendant of the binary commit, i.e. safe to rebuild to a newer tree),
  `OnMainBranch`, `BinaryCommit`, `RepoCommit`, `CommitsBehind`, `Error`.
- `version.CheckStaleBinary(repoDir string) *StaleBinaryInfo` — the main
  entry point (`stale.go:75-150`). Reads the binary commit via
  `resolveCommitHash`, shells out to `git rev-parse HEAD` in `repoDir`,
  then compares. Designed to be **fast and non-blocking**: errors are
  captured in `info.Error` and never propagate as failures.
- `version.GetRepoRoot() (string, error)` — locates the git repo holding
  the gastown source (`stale.go:156-199`). Tries, in order:
  1. `$GT_ROOT/gastown` and `$GT_ROOT/gastown/mayor/rig` (env agents
     always have `GT_ROOT`);
  2. Conventional `$HOME`-based paths (`~/gt/gastown`, `~/gastown`,
     `~/src/gastown`, plus their `mayor/rig` variants);
  3. The CWD's git toplevel as a last resort.
  A candidate must contain `cmd/gt/main.go` (`hasGtSource`) to be
  accepted.

### Internals / Notable implementation

- **Commit comparison uses prefix matching.** `commitsMatch`
  (`stale.go:59-69`) requires at least 7 character overlap to avoid false
  positives and treats one hash being a prefix of the other as a match.
  This handles mixed short-vs-full hash input from different sources.
- **Stale detection has two false-positive guards** (`stale.go:108-127`):
  1. If the binary commit isn't present in the target repo's object store
     at all — different clone, different commit history — the function
     bails out without marking the binary stale. Comment explicitly cites
     the case where `GetRepoRoot` finds a different clone
     (`mayor/rig`) than the one that built the running binary (`crew/
     woodhouse`).
  2. If all commits between `BinaryCommit` and `HEAD` exclusively touch
     `.beads/` files (bd backup commits), the binary is treated as
     up-to-date. This is the `onlyBeadsChanges` helper at
     `stale.go:220-232`, citing GH#2596.
- **Forward-only rebuild gate** (`stale.go:132-136`): `IsForward` is set
  only when `git merge-base --is-ancestor <binaryCommit> HEAD` succeeds.
  The comment calls out a real past bug: a crew worktree's HEAD was
  behind the binary's commit and auto-rebuild looped.
- **Build-branch allowlist** (`isBuildBranch`, `stale.go:241-247`): auto
  rebuilds only trigger on `main`, `master`, or any `carry/*` branch.
  Feature branches, fix branches, and polecat branches are explicitly
  excluded to prevent downgrade/crash-loop scenarios.
- Every subprocess call wraps itself in `util.SetDetachedProcessGroup` so
  a stale-check git invocation doesn't flash a console window on Windows
  or spawn a process group that outlives the parent.

### Usage pattern

- At startup, `cmd` init sets `version.Commit = <ldflags value>` by
  calling `version.SetCommit`.
- `gt version` reads `version.Commit` directly for output.
- `gt stale` (and the background daemon health checks that call into it)
  invoke `version.GetRepoRoot()` → `version.CheckStaleBinary(root)` and
  react to the returned struct.
- `ShortCommit` is used wherever a readable 12-char hash is needed in
  user-facing output.

## Related wiki pages

- [gt](../binaries/gt.md) — the binary whose staleness this package
  reports on.
- [gt stale](../commands/stale.md) — the command that surfaces
  `CheckStaleBinary` to users.
- [gt version](../commands/version.md) — consumes `version.Commit` for
  display.
- [internal/util](util.md) — source of `SetDetachedProcessGroup`, used
  by every git subprocess in this file.
- [go-mod](../files/go-mod.md) — dependency context.
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Precondition violations (what does it assume?)
- **Git available on PATH:** `CheckStaleBinary` shells out to `git
  rev-parse HEAD` at `stale.go:86-89` and `GetRepoRoot` tries `git
  rev-parse --show-toplevel` at `stale.go:189-191`. If `git` is not
  installed (container image without git, minimal Windows install), these
  fail. **Present** — errors are captured in `StaleBinaryInfo.Error`
  and propagated cleanly; callers degrade to "cannot determine
  staleness."
- **`repoDir` is a valid git repo:** `CheckStaleBinary` passes
  `repoDir` directly to `git` commands without checking `isGitRepo`
  first. The `git rev-parse HEAD` at `stale.go:86` will fail with an
  opaque git error if the directory doesn't exist or isn't a repo.
  **Present** — the error is wrapped and stored in
  `StaleBinaryInfo.Error`.

### Silent suppression (what errors are swallowed?)
- **Branch detection failure:** `git symbolic-ref --short HEAD` at
  `stale.go:98-104` — if the error return is non-nil (detached HEAD,
  shallow clone), `OnMainBranch` silently defaults to `false`. The
  branch output is discarded with `if branchOutput, err :=
  branchCmd.Output(); err == nil`. **Absent** — no log or warning
  when branch detection fails; `OnMainBranch=false` causes auto-rebuild
  to be suppressed on what might actually be `main`, silently degrading
  the stale-rebuild feature for detached-HEAD checkouts.
- **Commits-behind count failure:** `git rev-list --count` at
  `stale.go:139-146` — if the count command fails or the output can't
  be parsed, `CommitsBehind` silently stays 0 with no error propagated.
  **Absent** — user sees "stale" but with zero commits behind, which
  is misleading.

## Notes / open questions

- The fallback search list in `GetRepoRoot` is hard-coded; there is no
  env override like `GT_GASTOWN_SOURCE_DIR`. Agents running outside the
  standard `~/gt/` layout without `GT_ROOT` set will fall back to the
  CWD git toplevel and may fail to find the binary source.
- `Commit == ""` collapses to "cannot determine binary commit (dev
  build?)" (`stale.go:81-83`), so locally-built `go run ./cmd/gt` sessions
  always report as unknown rather than stale.
