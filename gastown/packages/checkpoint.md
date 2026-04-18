---
title: internal/checkpoint
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/checkpoint/checkpoint.go
  - /home/kimberly/repos/gastown/internal/checkpoint/squash.go
tags: [package, checkpoint, crash-recovery, polecat, git, squash]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/checkpoint

Session checkpointing for crash recovery. When a polecat session dies
(context limit, crash, timeout), checkpoints allow the next session to
recover state and resume work. The package provides read/write/capture
operations for `.polecat-checkpoint.json` files and a WIP commit
squashing utility for cleaning up auto-checkpoint commits before merge.

**Go package path:** `github.com/steveyegge/gastown/internal/checkpoint`
**File count:** 2 non-test Go files (checkpoint.go 222 lines,
squash.go 128 lines; 350 total).
**Imports (notable):** [`internal/runtime`](runtime.md)
(`SessionIDFromEnv`), [`internal/util`](util.md)
(`SetDetachedProcessGroup`), `os/exec` (git subprocess calls).
**Imported by:** `internal/cmd/checkpoint_cmd.go` (CLI wiring),
`internal/cmd/prime_output.go` and `internal/cmd/prime_session.go`
(session priming reads checkpoint state).

## What it actually does

### Checkpoint struct (checkpoint.go:22-53)

The `Checkpoint` struct captures a polecat session's recoverable state:

- `MoleculeID` -- the molecule being worked (`checkpoint.go:25`).
- `CurrentStep` / `StepTitle` -- step-level progress within the
  molecule (`checkpoint.go:28-31`).
- `ModifiedFiles` -- files modified since the last commit
  (`checkpoint.go:34`).
- `LastCommit` -- SHA of the last git commit (`checkpoint.go:37`).
- `Branch` -- current git branch (`checkpoint.go:40`).
- `HookedBead` -- the bead ID on the agent's hook (`checkpoint.go:43`).
- `Timestamp` -- when the checkpoint was written (`checkpoint.go:46`).
- `SessionID` -- identifies the session that wrote it
  (`checkpoint.go:49`).
- `Notes` -- optional free-text context (`checkpoint.go:52`).

### File operations (checkpoint.go:55-116)

**File format:** `.polecat-checkpoint.json` in the polecat's working
directory (`checkpoint.go:20`).

- `Path(polecatDir string) string` (`checkpoint.go:56-58`) -- returns
  the checkpoint file path.
- `Read(polecatDir string) (*Checkpoint, error)` (`checkpoint.go:62-79`)
  -- loads and unmarshals the checkpoint. Returns `(nil, nil)` if no
  file exists, allowing callers to distinguish "no checkpoint" from
  "corrupt checkpoint."
- `Write(polecatDir string, cp *Checkpoint) error`
  (`checkpoint.go:82-107`) -- sets `Timestamp` if zero, fills
  `SessionID` from `runtime.SessionIDFromEnv()` (falls back to
  `pid-<N>`), writes indented JSON with `0600` permissions.
- `Remove(polecatDir string) error` (`checkpoint.go:110-116`) --
  deletes the checkpoint file, tolerating already-removed files.

### State capture (checkpoint.go:119-161)

`Capture(polecatDir string) (*Checkpoint, error)` (`checkpoint.go:119-161`)
-- creates a checkpoint by running three git subcommands:
1. `git status --porcelain` (`checkpoint.go:125-140`) -- parses
   modified files from the porcelain format (strips the 2-char status
   prefix + space).
2. `git rev-parse HEAD` (`checkpoint.go:143-149`) -- captures the
   current commit SHA.
3. `git rev-parse --abbrev-ref HEAD` (`checkpoint.go:152-158`) --
   captures the branch name.

All git commands use `util.SetDetachedProcessGroup` for process
isolation (`checkpoint.go:127, 144, 153`).

### Builder methods (checkpoint.go:164-181)

Fluent builder pattern for adding context to a captured checkpoint:
- `WithMolecule(moleculeID, stepID, stepTitle string)` --
  molecule/step context.
- `WithHookedBead(beadID string)` -- bead tracking.
- `WithNotes(notes string)` -- free-text context.

### Query methods (checkpoint.go:183-222)

- `Age() time.Duration` (`checkpoint.go:184-186`) -- time since
  checkpoint was written.
- `IsStale(threshold time.Duration) bool` (`checkpoint.go:189-191`) --
  true if age >= threshold.
- `Summary() string` (`checkpoint.go:194-222`) -- human-readable
  one-line summary (molecule, hooked bead, modified file count, branch).

### WIP commit squashing (squash.go)

The squash module manages auto-checkpoint commits created by the
`checkpoint_dog` during long-running polecat sessions.

**Constants:**
- `WIPCommitPrefix = "WIP: checkpoint (auto)"` (`squash.go:12`) --
  the commit message prefix that identifies auto-checkpoint commits.

**`CountWIPCommits(workDir, baseRef string) (int, error)`**
(`squash.go:15-39`) -- finds the merge-base of `baseRef` and `HEAD`,
lists commit subjects in that range, counts those starting with
`WIPCommitPrefix`.

**`SquashWIPCommits(workDir, baseRef string) (int, error)`**
(`squash.go:47-108`) -- collapses all commits from merge-base to HEAD
into a single commit:
1. Finds merge-base and lists commit subjects (`squash.go:48-62`).
2. Separates WIP from non-WIP subjects (`squash.go:66-77`).
3. `git reset --soft <merge-base>` to preserve changes as staged
   (`squash.go:81-83`).
4. Builds a combined commit message: first non-WIP subject as title,
   remaining non-WIP subjects as body bullets. If all commits were WIP,
   uses "squashed WIP checkpoint commits" (`squash.go:86-101`).
5. `git commit -m <combined>` (`squash.go:103-105`).

The comment at `squash.go:44-46` explains why this is safe: "Refinery
squash-merges polecat branches anyway -- individual commit history on
polecat branches is not preserved."

## Related

- [gt checkpoint](../commands/checkpoint.md) -- the CLI command that
  wraps this package (command page, distinct entity).
- [internal/runtime](runtime.md) -- `SessionIDFromEnv` used by
  `Write` to identify the session.
- [internal/util](util.md) -- `SetDetachedProcessGroup` for git
  subprocess isolation.
- [internal/polecat](polecat.md) -- the ephemeral worker whose state
  checkpoints protect.
- [internal/refinery](refinery.md) -- consumes polecat branches; the
  squash operation prepares branches for refinery's merge process.
- [molecule concept](../concepts/molecule.md) -- `MoleculeID` and
  `CurrentStep` in the checkpoint track molecule progress.
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- `Write` uses `os.WriteFile` with `0600` permissions
  (`checkpoint.go:102`), not `util.AtomicWriteJSON`. A crash mid-write
  could leave a corrupt checkpoint file. Since `Read` returns an
  unmarshal error for corrupt files (rather than silently falling back
  to defaults), a corrupt checkpoint would surface as an error to the
  recovery session. This is a defensible tradeoff -- checkpoints are
  best-effort recovery aids, not transactional state.
- The `Capture` function does not populate `MoleculeID`, `CurrentStep`,
  `HookedBead`, or `Notes` -- these must be added by the caller via
  the builder methods. This split keeps `Capture` focused on
  git-observable state.
- `SquashWIPCommits` uses `git reset --soft` followed by `git commit`,
  which rewrites history on the polecat branch. This is explicitly
  called out as safe because refinery squash-merges polecat branches.
