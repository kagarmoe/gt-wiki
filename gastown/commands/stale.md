---
title: gt stale
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/stale.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, version, build, stale-binary]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: dev
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt stale

Reports whether the running `gt` binary was built from an older commit
than the current HEAD of the gastown repository. Supports human,
JSON, and exit-code-only modes.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/stale.go:16-35`)
**Beads-exempt:** no (not in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45-81`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/stale.go:55-141`.

### Invocation

```
gt stale [--json] [-q|--quiet]
```

### Behavior

`runStale` (`stale.go:55-108`):

1. Resolves the gastown repo root via `version.GetRepoRoot`
   (`stale.go:57-66`). On failure: in `--quiet` mode returns exit 2
   via `NewSilentExit(2)`; in `--json` mode outputs
   `{"error": "..."}`; otherwise wraps and returns the error.
2. Calls `version.CheckStaleBinary(repoRoot)` (`stale.go:69`). This
   returns a `StaleInfo` struct with `IsStale`, `IsForward`,
   `OnMainBranch`, `BinaryCommit`, `RepoCommit`, `CommitsBehind`, and
   an optional `Error`.
3. On `info.Error != nil`: same quiet/json/text error paths as above
   (`stale.go:72-80`).
4. In `--quiet` mode (`stale.go:83-88`): returns `SilentExit(0)` when
   stale, `SilentExit(1)` when fresh. **Note the inverted convention**:
   0 = "stale" (action needed), 1 = "fresh" (OK). The `Long` text at
   `stale.go:30-33` documents this explicitly.
5. Builds a `StaleOutput` (`stale.go:91-101`). `SafeToRebuild` is
   derived locally as `IsStale && IsForward && OnMainBranch`
   (`stale.go:92`) — the underlying `CheckStaleBinary` does not
   compute it.
6. Writes JSON or text (`stale.go:103-107`).

### Output formats

**JSON** — `outputStaleJSON` (`stale.go:110-114`):

```json
{
  "stale": false,
  "forward": true,
  "on_main_branch": true,
  "safe_to_rebuild": false,
  "binary_commit": "abc1234...",
  "repo_commit": "def5678...",
  "commits_behind": 0
}
```

`commits_behind` is omitted when zero (`omitempty` at `stale.go:51`).
`error` is omitted when empty.

**Text** — `outputStaleText` (`stale.go:116-141`):

- Stale: `⚠ Binary is stale`, then `Binary:` / `Repo:` short commits,
  optional `(N commits behind)` line, and either:
  - `✗ repo HEAD is NOT a descendant of binary commit (diverged or
    older)` when `!Forward`
  - `⚠ repo is not on main branch` when `!OnMainBranch`
  - `Safe to rebuild: run 'make build && make install'` when
    `SafeToRebuild`
  - `✗ NOT safe for automated rebuild (forward=…, main=…)` otherwise
- Fresh: `✓ Binary is fresh` and `Commit:` short hash.

### Exit codes

From `Long` text at `stale.go:30-33` and runtime behavior:

| Code | Meaning                                 |
|------|-----------------------------------------|
| 0    | Binary is stale (quiet) or no error     |
| 1    | Binary is fresh (quiet only)            |
| 2    | Error (could not determine staleness)   |

`NewSilentExit` (referenced at `stale.go:60, 74, 85, 87`) is a
cmd-package helper that suppresses usage output and returns the given
exit code.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`stale.go:37-41`):

| Flag             | Type | Default | Purpose                                    |
|------------------|------|---------|--------------------------------------------|
| `--json`         | bool | `false` | Output as JSON                             |
| `--quiet` / `-q` | bool | `false` | Exit code only, no stdout output           |

### SafeToRebuild semantics

Three conditions must hold simultaneously (`stale.go:91-97`):

1. `IsStale` — binary commit differs from repo HEAD
2. `IsForward` — repo HEAD is a descendant of binary commit (no
   divergence)
3. `OnMainBranch` — repo is on the main branch

This is the condition under which an automated agent or patrol is
allowed to run `make build && make install` without human review.

## Related

- [doctor](doctor.md) — `NewStaleBinaryCheck` is the doctor-side
  equivalent; doctor runs it as part of the infrastructure check
  block at `doctor.go:158`.
- [upgrade](upgrade.md) — when stale, the remediation is to rebuild
  and then run `gt upgrade` to pick up migrations. Upgrade also
  registers `NewStaleBinaryCheck` (`upgrade.go:123`).
- [version](version.md) — version's `init()` calls
  `version.SetCommit` to bootstrap the stale-check machinery.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  `persistentPreRun`'s stale warning that uses the same check.
- [../files/makefile.md](../files/makefile.md) — the `make build` /
  `make install` recipe that makes a binary fresh.
- [README.md](README.md) — command tree index.

## Failure modes

No significant failure modes. All error paths are handled explicitly
with three-way branching (quiet/json/text) at `stale.go:57-80`. The
`version.GetRepoRoot` and `version.CheckStaleBinary` errors are
properly propagated with appropriate exit codes (2 for errors). The
`NewSilentExit` pattern is clean and documented.

## Notes / open questions

- `version.CheckStaleBinary` and `version.GetRepoRoot` live in
  [internal/version](../packages/version.md), which documents how
  the binary commit is embedded at build time and how repo HEAD is
  resolved (git subprocess vs. pure-Go).
- The inverted exit codes in `--quiet` mode (0=stale, 1=fresh) are
  non-obvious and surprising for shell scripts; the `Long` text calls
  this out explicitly. Scripts commonly expect 0 for success, so a
  wrapper is advised.
- `CommitsBehind` is only populated when `!IsForward`'s opposite
  applies — the count is meaningful when repo is strictly ahead of
  the binary, but the JSON `omitempty` tag hides it when zero
  (`stale.go:51`).
