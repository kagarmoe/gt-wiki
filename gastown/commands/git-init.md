---
title: gt git-init
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/gitinit.go
tags: [command, workspace, git, setup, gitignore, branch-protection]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# gt git-init

Initialize or configure git for an existing Gas Town HQ: write a
Gas-Town-tailored `.gitignore`, run `git init` if needed, install a
branch-protection hook, and optionally create a GitHub repo via
`gh`.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`gitinit.go:22`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** yes (in `branchCheckExemptCommands` on
`root.go:81-91`; makes sense — the hook it installs is precisely
what branch-check relies on)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/gitinit.go` (495
lines). Registered on `rootCmd` in `gitinit.go:48-52`.

Despite the file's size, only the top ~200 lines are the CLI
command itself. The rest is the embedded `HQGitignore` constant
(`gitinit.go:55-148`), branch-protection helpers
(`InstallPreCheckoutHook`, `InstallBranchProtection`,
`IsBranchProtectionInstalled` — `gitinit.go:413-495`), and the
`BranchProtectionScript` shell snippet (`gitinit.go:371-406`) that
is embedded in the git post-checkout hook.

### Invocation

```
gt git-init [--github=<owner>/<repo>] [--public]
```

No positional args. Operates on the current directory (must be
inside a Gas Town HQ).

### Behavior

`runGitInit` (`gitinit.go:150-206`):

1. **Find the HQ root.** `workspace.Find(cwd)` — aborts with
   `"not inside a Gas Town HQ (run 'gt install' first)"` if the
   current directory is not under an HQ.
2. **Create or update `.gitignore`** via `createGitignore`
   (`gitinit.go:208-238`). If a file already exists and contains
   the marker `"Gas Town HQ"`, it is left alone. Otherwise, the
   embedded `HQGitignore` constant (`gitinit.go:55-148`) is
   appended (or used to create a new file).
3. **Run `git init`** if `.git/` does not exist (`gitinit.go:240-251`).
4. **Install branch protection** via `InstallPreCheckoutHook` —
   which actually routes to `InstallBranchProtection`
   (`gitinit.go:419-477`). This **prepends** the
   `BranchProtectionScript` to the `.git/hooks/post-checkout`
   hook (after the shebang, if one exists). The script auto-reverts
   the town root back to `main` (or `master`) whenever a non-main
   branch is checked out. The comment at `gitinit.go:369-374`
   notes explicitly that git does NOT support a "pre-checkout"
   hook, so the code detects bad checkouts after the fact and
   reverts them. The obsolete `pre-checkout` hook is best-effort
   removed if it was previously installed (`gitinit.go:428-434`).
   Installation is a soft failure: a warning is printed but
   `runGitInit` still reports success.
5. **Create GitHub repo** via `createGitHubRepo` if `--github` is
   set (`gitinit.go:253-299`). Requires the `gh` CLI on `PATH`,
   parses `owner/repo`, calls `ensureInitialCommit`
   (`gitinit.go:303-326`) to create an initial commit if one
   doesn't exist, then runs `gh repo create --source <hq>
   --private|--public --push`.

### Subcommands

None. `gt git-init` is a leaf command.

### Flags

| flag | type | default | source |
|---|---|---|---|
| `--github <owner/repo>` | string | `""` | `gitinit.go:49` |
| `--public` | bool | `false` (so repos are private by default) | `gitinit.go:50` |

## The embedded `.gitignore`

`HQGitignore` (`gitinit.go:55-148`) excludes, among others:

- Runtime state: `state.json`, `*.lock`, `*.flock`, `heartbeat.json`,
  `activity.json`, `.events.jsonl`, `.feed.jsonl`, `*.pid`.
- `daemon/`, `logs/`, `.dolt-data/`, `**/.dolt/`, `**/.doltcfg/`,
  `events/`, `beads_hq/`.
- Agent workspaces: `**/polecats/`, `**/deacon/dogs/`,
  `**/mayor/rig/`, `**/refinery/rig/`, `**/crew/`.
- `**/.runtime/`.
- OS and editor files: `.DS_Store`, `*.swp`, `.vscode/`, `.idea/`.

The "Explicitly track" section at the bottom (`gitinit.go:144-147`)
is effectively a comment: nothing is explicitly un-ignored, but
`.beads/` has its own `.gitignore` that handles database files
separately.

## Branch-protection script

`BranchProtectionScript` (`gitinit.go:371-406`) is the Bourne shell
snippet injected into `.git/hooks/post-checkout`. The logic:

- Checks `$3` (1 for branch checkout, 0 for file checkout) — only
  acts on branch checkouts.
- If the current branch is `main` or `master`, silent ok.
- Otherwise, prints the auto-revert warning and runs `git checkout
  main` (falling back to `master`). If both fail, prints
  `"✗ Failed to revert - please run: git checkout main"`.

The identifier string is `BranchProtectionMarker =
"Gas Town branch protection"` (`gitinit.go:364`), used by both
`IsBranchProtectionInstalled` and by the install flow to skip
re-installing.

## Related commands

- [install.md](install.md) — `gt install --git` reuses
  `InitGitForHarness` (`gitinit.go:331-361`), which is the same
  scaffolding as `runGitInit` minus the "find HQ root" preamble.
  Meaning: **everything `gt git-init` does, `gt install --git` also
  does**. `gt git-init` is the retrofit path for HQs created
  without `--git`.
- [uninstall.md](uninstall.md) — inverse of `install`. Neither
  `install` nor `uninstall` un-writes the `.gitignore` or removes
  the branch-protection hook (worth verifying — out of scope here).
- [doctor.md](doctor.md) — diagnostics.

## Docs claim

### Source
- `internal/cmd/gitinit.go:25-30` — Cobra `Long` text, numbered steps

### Verbatim
> This command:
>   1. Creates a comprehensive .gitignore for Gas Town
>   2. Initializes a git repository if not already present
>   3. Optionally creates a GitHub repository (private by default)

## Drift

### gitInitCmd.Long lists 3 steps; code performs 4 operations
- **Claim source:** Cobra `Long` text at `internal/cmd/gitinit.go:25-27`
- **Docs claim:** The Long text lists 3 numbered steps: (1) .gitignore, (2) git init, (3) GitHub repo.
- **Code does:** `runGitInit` (`gitinit.go:150-206`) performs 4 operations: (1) .gitignore (`gitinit.go:168-170`), (2) git init (`gitinit.go:173-181`), (3) **branch-protection hook installation** via `InstallPreCheckoutHook` (`gitinit.go:184-186`), (4) GitHub repo creation (`gitinit.go:189-193`). Step 3 — the branch-protection post-checkout hook — is omitted from the Long text entirely. This is a significant behavioral feature: the hook auto-reverts the HQ to `main` on any non-main branch checkout.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — add branch-protection hook installation as step 3 (renumber GitHub to step 4) in `gitInitCmd.Long`
- **Release position:** `in-release` — `InstallPreCheckoutHook` present at v1.0.0

See [gastown/drift/README.md](../drift/README.md) for the consolidated corrections list.

## Notes / open questions

- **Exported helpers.** `InitGitForHarness`, `InstallBranchProtection`,
  and `IsBranchProtectionInstalled` are all exported from this file
  (and intentionally so, per comments). `install.go` calls
  `InitGitForHarness`. Anything else that calls
  `IsBranchProtectionInstalled` is worth hunting — likely `gt
  doctor` or a related health check.
- **No `--gitignore-only` flag.** If a user wants to refresh the
  Gas Town `.gitignore` block without touching anything else,
  there's no way to do it — the "already configured" check
  short-circuits on any file containing `"Gas Town HQ"`, so edits
  to the canonical block won't update.
- **`ensureInitialCommit` uses `git add .`**. For an HQ that the
  user has been editing before running `gt git-init --github=...`,
  this stages everything in the working tree and commits it with
  the message `"Initial Gas Town HQ"`. Could be surprising if the
  tree already has secrets or generated files that slipped past
  the new `.gitignore`.
- **Auto-revert is post-hoc.** A checkout happens, then is
  reverted. If tools notice the transient state (e.g. file-system
  watchers), they may trigger on it. Not a bug, but worth knowing.
