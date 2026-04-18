---
title: internal/git
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/git/git.go
  - /home/kimberly/repos/gastown/internal/git/copy_unix.go
  - /home/kimberly/repos/gastown/internal/git/copy_windows.go
tags: [package, git, subprocess, clone, worktree, merge, gh, zfc, platform-service]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/git

Subprocess-level wrapper around the `git` CLI. Every git operation in
Gas Town goes through here: clones (regular / bare / partial /
reference-shared / branch-scoped in ~12 permutations), worktree
lifecycle, branch management, merges (including squash and
fast-forward-only), status/diff/rev parsing, upstream remote setup, and
a handful of `gh`-CLI bridges for PR operations. The package is almost
2,300 lines in a single `git.go` file plus tiny per-platform shims for
cross-filesystem directory moves.

**Go package path:** `github.com/steveyegge/gastown/internal/git`
**File count:** 3 go files (`git.go`, `copy_unix.go`, `copy_windows.go`),
3 test files.
**Imports (notable):** stdlib (`bytes`, `encoding/json`, `errors`,
`os`, `os/exec`, `path/filepath`, `runtime`, `strings`), and
[`internal/util`](util.md) for `SetDetachedProcessGroup` — every
subprocess call installs the detached-process-group wrapper so git
invocations don't flash console windows on Windows and don't outlive
their parent.
**Imported by (notable):** [`internal/refinery`](refinery.md) for
merge-queue and integration-branch work, the
[`gt commit`](../commands/commit.md) / [`gt release`](../commands/release.md)
/ [`gt mq`](../commands/mq.md) command paths, and the
[`gt stale`](../commands/stale.md) build-freshness checks. In short,
anywhere Gas Town shells out to git.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/git/git.go`.

### Core wrapper

- `Git` struct (`git.go:61-64`): `{workDir, gitDir}`. `NewGit(workDir)`
  for a working copy, `NewGitWithDir(gitDir, workDir)` for bare repos
  where `--git-dir=<path>` is prepended to every command.
- `run(args...) (string, error)` (`git.go:90-112`) is the chokepoint.
  Sets `cmd.Dir = workDir`, installs `util.SetDetachedProcessGroup`,
  captures stdout + stderr into separate `bytes.Buffer`s, and on
  failure returns a `*GitError` via `wrapError`. On success, returns
  trimmed stdout.
- `runWithEnv(args, extraEnv)` (`git.go:115-135`) — variant that
  appends `extraEnv` to `os.Environ()`. Used by `PushWithEnv` to pass
  `GT_INTEGRATION_LAND=1` for the pre-push hook on integration landings.

### GitError (ZFC-friendly)

`GitError` struct (`git.go:22-39`) carries `Command`, `Args`, raw
`Stdout`, raw `Stderr`, and the underlying `Err`. The doc comment
explicitly frames this as a ZFC ("agents observe raw output and decide")
surface: `Error()` is human-readable, but callers that need to
distinguish error classes should inspect `Stdout`/`Stderr`
programmatically. `wrapError` deliberately does not classify or re-wrap
errors into typed variants — the raw text is the contract.

`Unwrap()` returns the underlying exec error, so `errors.Is` against
`*exec.ExitError` still works.

### Clone matrix (`git.go:165-421`)

The package has roughly a dozen clone variants — the cartesian product
of `{regular, bare}` × `{full, single-branch, partial}` ×
`{with-reference, without}` × `{default-branch, explicit-branch}`. All
funnel through a single internal `cloneInternal(url, dest, opts
cloneOptions)` (`git.go:177`) which:

1. Creates a temp directory with `os.MkdirTemp("", "gt-clone-*")`.
2. Runs `git clone` **inside the temp dir**, isolating it from any git
   repo at the process CWD.
3. `moveDir`s the clone into the real destination.
4. Post-clone: calls `configureRefspec` for bare repos (so `git fetch`
   populates `refs/remotes/origin/*`, GH#286 fix) and
   `configureHooksPath` for non-bare repos (so
   `.githooks/pre-push` runs when present).

Clone options (`cloneOptions`, `git.go:166-173`): `bare`, `reference`,
`singleBranch`, `depth`, `branch`, `filter`. `singleBranch+depth=1` is
the default for `Clone`/`CloneWithReference`/`CloneBranch` — partial
clones use `filter` instead (e.g. `"blob:none"`, `"tree:0"`) and skip
`depth` since the filter handles size reduction.

Windows-specific detail: for non-bare reference clones on Windows, the
internal clone prepends `-c core.symlinks=true` (`git.go:196-198`).

`moveDir` (`git.go:44-58`) tries `os.Rename` first and falls back to
`copyDirPreserving` + `os.RemoveAll` on EXDEV (cross-filesystem)
errors. That's the only job of `copy_unix.go` and `copy_windows.go`:

- `copy_unix.go:13-17` — shells out to `cp -a` (preserves symlinks,
  perms, timestamps, all attributes).
- `copy_windows.go:15-38` — shells out to `robocopy <src> <dest> /E
  /COPYALL /SL /R:0 /W:0`, treating exit codes 0–7 as success. Comment
  explicitly notes "This Windows implementation has not been tested on
  Windows" — a drift flag.

### Operations surface (highlights)

The public method set is large; grep `git.go` for the full list. Major
groupings:

- **Clone variants**: `Clone`, `CloneWithReference`, `CloneBranch`,
  `CloneBranchWithReference`, `CloneBare`, `CloneBareWithBranch`,
  `CloneBarePartial`, `CloneBarePartialWithBranch`,
  `CloneBarePartialWithReference`,
  `CloneBarePartialWithReferenceAndBranch`,
  `CloneBranchPartial`, `CloneBranchPartialWithReference`,
  `CloneBareWithReference`, `CloneBareWithReferenceAndBranch`.
- **Worktree**: `WorktreeAdd`, `WorktreeAddFromRef`,
  `WorktreeAddDetached`, `WorktreeAddExisting`,
  `WorktreeAddExistingForce`, `WorktreeRemove`, `WorktreeMove`,
  `WorktreePrune`, `WorktreeList` (returns `[]Worktree`).
- **Branch**: `CheckoutNewBranch`, `CreateBranch`, `CreateBranchFrom`,
  `BranchExists`, `RemoteBranchExists`, `PushRemoteBranchExists`,
  `RemoteTrackingBranchExists`, `RefExists`, `DeleteBranch`,
  `DeleteRemoteBranch`, `ListBranches`, `ResetBranch`.
- **Merge**: `Merge`, `MergeNoFF`, `MergeFFOnly`, `MergeSquash`
  (preserves branch HEAD commit message for the squash),
  `AbortMerge`, `CheckConflicts` (test-merge that always aborts),
  `GetConflictingFiles`, `Rebase`, `AbortRebase`.
- **State**: `Status` (returns `*GitStatus`, filters out
  skip-worktree files via `skipWorktreeFiles`), `CurrentBranch`,
  `DefaultBranch`, `RemoteDefaultBranch` (checks `origin/HEAD`, then
  master/main, falls back to `"main"`), `HasUncommittedChanges`,
  `Rev`, `IsAncestor`, `CommitsAhead`, `CountCommitsBehind`,
  `StashCount`, `UnpushedCommits`, `BranchCreatedDate`, `DiffStat`,
  `DiffNameOnly`.
- **Remote**: `RemoteURL`, `AddRemote`, `SetRemoteURL`, `ConfigurePushURL`
  (sets push URL without touching fetch URL — used for fork workflows),
  `ClearPushURL` (`--unset-all`; exit code 5 is treated as "not set"),
  `GetPushURL`, `AddUpstreamRemote`, `GetUpstreamURL`,
  `HasUpstreamRemote`, `FetchUpstream`, `Remotes`, `ConfigGet`,
  `Fetch`, `FetchPrune`, `FetchBranch`, `FetchBranchShallow`, `Pull`,
  `Push`, `PushWithEnv`, `ListRemoteRefs`, `ListPushRemoteRefs`.
- **Staging**: `Add`, `Commit`, `CommitAll`, `ResetFiles`, `ShowFile`,
  `CheckoutFileFromRef`, `RmCached`.
- **GitHub bridges** (shell out to `gh`, not the HTTP client —
  see [internal/github](github.md) for the HTTP side): `HasOpenPR`,
  `FindPRNumber`, `IsPRApproved`, `GhPrMerge`.
- **Submodules**: `InitSubmodules`, `InitSparseCheckout`,
  `IsSparseCheckoutConfigured`, `RemoveSparseCheckout`,
  `SubmoduleChanges`, `PushSubmoduleCommit`,
  plus internal `submoduleURL`, `submoduleDefaultBranch`,
  `submoduleReferencePath`, `isValidSubmoduleReference`,
  `hasTrackedGitmodules`.
- **Stale/contamination**: `CheckBranchContamination`,
  `PruneStaleBranches`.
- **Uncommitted work classification**: `CheckUncommittedWork`
  returns `*UncommittedWorkStatus` with helpers `Clean`,
  `CleanExcludingBeads`, `CleanExcludingRuntime`, and `String`. The
  helpers know that `.beads/*` (via `isBeadsPath`) and `.runtime/*`
  (via `isGasTownRuntimePath`) are allowed to be dirty without
  counting as "uncommitted work".
- **Push-tracking**: `BranchPushedToRemote` returns whether a branch
  is on a remote and how many commits ahead the local is.

### Post-clone configuration

- `configureHooksPath` (`git.go:321-342`) sets `core.hooksPath` to the
  repo's `.githooks/` directory when present. This is what ensures Gas
  Town agents pick up the pre-push hook that blocks pushes to non-main
  branches inside the town — "internal PRs are not allowed" (comment,
  `git.go:319-320`).
- `configureRefspec` (`git.go:352-411`) fixes the bare-clone refspec
  problem noted in GH#286: without this, `git fetch` only updates
  `FETCH_HEAD` and `refs/remotes/origin/*` never appears, which breaks
  worktrees. When `singleBranch` is set, the refspec is narrowed to the
  default branch to avoid "some local refs could not be updated" on
  repos with many branches.

## Related wiki pages

- [internal/refinery](refinery.md) — heaviest consumer; all merges and
  integration-branch work land here.
- [internal/github](github.md) — HTTP-based GitHub API client (REST +
  GraphQL). Orthogonal to `git`'s `gh`-CLI bridges: the HTTP client is
  used from the merge queue's long-lived path, the `gh` bridges are
  used for quick PR existence checks.
- [internal/util](util.md) — supplier of `SetDetachedProcessGroup`.
- [gt commit](../commands/commit.md), [gt release](../commands/release.md),
  [gt mq](../commands/mq.md) — command-level entry points.
- [gt stale](../commands/stale.md), [gt doctor](../commands/doctor.md).
- [gt](../binaries/gt.md).
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Git subprocess cleanup:** A handful of `_ =` calls in git
  operation wrappers for process cleanup. **Present** — errors in
  the git wrapper are propagated from the main operations; only
  cleanup is suppressed.

### Cross-platform concerns
- **copy_unix.go / copy_windows.go:** File-copy platform shims.
  The Windows implementation comment at `copy_windows.go:14` notes
  "This Windows implementation has not been tested on Windows."
  **Untested** — explicitly documented as untested.

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `cp` | `-a` | `<src> <dest>` (Unix) | runtime | `copy_unix.go:14` |
| `robocopy` | (Windows) | `/E /COPYALL /SL /R:0 /W:0` | hardcoded | `copy_windows.go:25` |
| `git` | (dynamic) | `args...` (Run) | caller | `git.go:96` |
| `git` | (dynamic) | `args...` (Output) | caller | `git.go:119` |
| `git` | (dynamic) | `args...` (CombinedOutput) | caller | `git.go:220` |
| `git` | `config` | `-C <repo> core.hooksPath .githooks` | hardcoded | `git.go:328` |
| `git` | `config` | `--git-dir <dir> remote.origin.fetch <refspec>` | hardcoded | `git.go:360` |
| `git` | `symbolic-ref` | `--git-dir <dir> HEAD` | hardcoded | `git.go:375` |
| `git` | `fetch` | `--git-dir <dir> --depth 1 origin` | hardcoded | `git.go:381` |
| `git` | `fetch` | `--git-dir <dir> --depth 1 origin <refspec>` | runtime | `git.go:393` |
| `git` | `fetch` | `--git-dir <dir> origin` | hardcoded | `git.go:402` |
| `gh` | `pr list` | `--head <branch> --state open --json number --limit 1` | runtime | `git.go:883` |
| `gh` | `pr list` | `--head <branch> --state open --json number --limit 1` | runtime | `git.go:897` |
| `gh` | `pr view` | `<prNumber> --json reviewDecision` | runtime | `git.go:923` |
| `gh` | (dynamic) | `args...` | caller | `git.go:944` |
| `git` | (dynamic) | `args...` (worktree ops) | caller | `git.go:1057` |
| `git` | `rev-parse` | `-C <repo> --is-shallow-repository` | hardcoded | `git.go:1396` |
| `git` | `config` | `-C <repo> core.sparseCheckout` | hardcoded | `git.go:1408` |
| `git` | `sparse-checkout disable` | `-C <repo>` | hardcoded | `git.go:1418` |
| `git` | (dynamic) | `args...` (worktree) | caller | `git.go:2034` |
| `git` | `ls-files` | `-C <repo> --error-unmatch .gitmodules` | hardcoded | `git.go:2055` |
| `git` | `sparse-checkout init` | `-C <repo> --cone` | hardcoded | `git.go:2063` |
| `git` | (dynamic) | `args...` (sparse-checkout set) | caller | `git.go:2072` |
| `git` | `config -f` | `<tmp> --get-regexp ^submodule` | runtime | `git.go:2168` |
| `git` | `config -f` | `<tmp> --get submodule.<name>.url` | runtime | `git.go:2197` |
| `git` | `push` | `-C <path> <remote> <sha>:refs/heads/<branch>` | runtime | `git.go:2221` |
| `git` | `symbolic-ref` | `-C <sub> refs/remotes/<remote>/HEAD` | runtime | `git.go:2239` |
| `git` | `rev-parse` | `-C <sub> --verify --quiet <ref>` | runtime | `git.go:2254` |
| `git` | `ls-remote` | `-C <sub> --exit-code <remote> refs/heads/<candidate>` | runtime | `git.go:2263` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| gitmodules temp file | `.gitmodules` content | Submodule config parsing | `git.go:2156` |

## Notes / open questions

- The Windows `robocopy` branch in `copy_windows.go` explicitly
  documents that it has not been tested on Windows. Cross-filesystem
  clones there are a live drift surface.
- Clone permutations are duplicated at the public surface (14+
  methods) rather than exposed as a single `CloneOpts` struct. The
  internal plumbing already uses `cloneOptions`; the public API is
  backward-compat cruft. Flagging as a potential simplification
  target, not a bug.
- Several methods shell out to `gh` (the GitHub CLI) — `HasOpenPR`,
  `FindPRNumber`, `IsPRApproved`, `GhPrMerge`. These are labelled
  fail-open: if `gh` isn't installed or authenticated, they return
  `false` / error. See comments at `git.go:882` and `919`.
- `CheckConflicts` always aborts — it's a "would this merge cleanly?"
  probe. Callers must not rely on the working tree having been
  mutated.
- `CheckUncommittedWork` specifically ignores `.beads/` and
  `.runtime/` dirty state. This is load-bearing for GT's workflows
  where bd backups and daemon runtime files are always dirty; it is
  not safe to generalize `CheckUncommittedWork` to arbitrary Gas Town
  repos that don't share those conventions.
- Every clone creates a temp directory and moves the result — a real
  cost on very large clones. The design prioritizes process-CWD
  isolation over speed.
