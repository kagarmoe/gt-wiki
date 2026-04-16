---
title: gt init
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/init.go
  - /home/kimberly/repos/gastown/internal/rig/types.go
tags: [command, workspace, rig, setup, primitive]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift, wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# gt init

Initialize the **current directory** as a Gas Town rig — create the
standard agent subdirectories and wire up `.git/info/exclude` so git
ignores them.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`init.go:23`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/init.go` (189 lines).
Registered on `rootCmd` in `init.go:35-38`.

### Invocation

```
gt init [--force|-f]
```

No positional args. Operates on the **current working directory**.

### Behavior

`runInit` (`init.go:40-120`) does the following:

1. **Git-repo precheck** (`init.go:47-50`). Builds a `git.NewGit(cwd)`
   and calls `CurrentBranch()`. If that fails, aborts with
   `"not a git repository (run 'git init' first)"`. So `gt init` is
   meaningfully distinct from [git-init.md](git-init.md): the latter
   initializes git in an HQ, this one requires an already-initialized
   git repo and scaffolds a rig inside it.
2. **Already-initialized guard** (`init.go:52-56`). If a `polecats/`
   subdirectory exists and `--force` was not passed, aborts with
   `"rig already initialized (use --force to reinitialize)"`.
3. **Create agent directories** (`init.go:62-77`). Loops over
   `rig.AgentDirs` (the canonical list from `internal/rig/types.go:48-54`
   — contents are `polecats`, `crew`, `refinery/rig`, `witness`,
   `mayor/rig`). For each, runs `os.MkdirAll(..., 0755)` and drops
   a `.gitkeep` so the empty directory can be tracked if anyone
   chooses to.
4. **Update `.git/info/exclude`** via `updateGitExclude`
   (`init.go:122-158`). Appends a `# Gas Town agent directories`
   section with root-anchored patterns (`/polecats/`, `/witness/`,
   etc.). The comment at `init.go:143-145` explains why anchoring
   matters: an unanchored `refinery/` pattern would hide the source
   directory `internal/refinery/` when this command runs inside the
   gastown repo itself. Already-configured excludes are detected by
   substring-matching `"Gas Town"` (`init.go:138`).
5. **Register custom beads types** via `registerCustomTypes`
   (`init.go:163-189`). Best-effort: if `bd` is not on `PATH`, or
   `.beads/` does not exist yet, silently returns nil. Otherwise runs
   `bd config set types.custom <constants.BeadsCustomTypes>` in the
   workspace directory. Common bd errors (`"not initialized"`,
   `"no such file"`) are also swallowed.
6. **Auto-configure Dolt lifecycle patrols** (`init.go:101-108`). If
   `workspace.FindFromCwd()` finds a town root (i.e. this rig lives
   inside an already-installed HQ), calls
   `daemon.EnsureLifecycleConfigFile(townRoot)` to fill in missing
   defaults for reaper, compactor, doctor, backup, and maintenance
   patrols in `mayor/daemon.json`. The doc comment at
   `init.go:97-100` is explicit that this only fills missing config
   and never overwrites user settings.

### Subcommands

None. `gt init` is a leaf command with one flag.

### Flags

| flag | type | default | source |
|---|---|---|---|
| `--force` / `-f` | bool | `false` | `init.go:36` |

## Relationship to the other setup commands

`gt init` is the **rig-scaffolder primitive**. Three related commands:

- [install.md](install.md) — creates a **town HQ** (outer workspace
  containing `mayor/`, `.beads/`, `daemon.json`, etc.). This is where
  you start.
- [git-init.md](git-init.md) — initializes git and `.gitignore` for
  an existing HQ. Complements [install.md](install.md) when `--git`
  was not passed at install time.
- `gt init` (this page) — scaffolds the agent subdirs of a single
  rig inside an HQ. This is a **lower-level primitive** than
  [rig.md](rig.md) `add`, which clones a repo and does full rig
  provisioning. Most users will never call `gt init` directly —
  [rig.md](rig.md) `add` covers the common case.

See also:

- [rig.md](rig.md) — high-level rig management (the usual entry
  point).
- [doctor.md](doctor.md) — diagnostics that report when custom
  beads types are missing.

## Docs claim

### Source
- `internal/cmd/init.go:26-27` — Cobra `Long` text, agent directory enumeration

### Verbatim
> This creates the standard agent directories (polecats/, witness/, refinery/,
> mayor/) and updates .git/info/exclude to ignore them.

## Drift

### initCmd.Long lists 4 directories; code creates 5 (and with different paths)
- **Claim source:** Cobra `Long` text at `internal/cmd/init.go:26-27`
- **Docs claim:** Creates "the standard agent directories (polecats/, witness/, refinery/, mayor/)".
- **Code does:** `init.go:62-77` loops over `rig.AgentDirs` defined at `internal/rig/types.go:48-54`: `polecats`, `crew`, `refinery/rig`, `witness`, `mayor/rig`. Three discrepancies: (1) omits `crew` entirely — crew directories are unconditionally created; (2) says `refinery/` but the actual path is `refinery/rig/`; (3) says `mayor/` but the actual path is `mayor/rig/`.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — update `initCmd.Long` to list all 5 directories with correct paths
- **Release position:** `in-release` — `rig.AgentDirs` with all 5 entries present at v1.0.0

**wiki-stale (inline fix applied above):** Phase 2 wiki body at step 3 repeated the Long text's incorrect 4-directory list. Fixed inline to cite the actual `rig.AgentDirs` at `internal/rig/types.go:48-54`. **Phase 2 root cause: `phase-2-incomplete` (heuristic)** — Phase 2 trusted the Long text instead of reading the `rig.AgentDirs` slice definition.

See [gastown/drift/README.md](../drift/README.md) for the consolidated corrections list.

## Notes / open questions

- **Who actually calls `gt init`?** The "Next steps" output at
  `init.go:113-117` points to `gt rig add` next, suggesting this
  command is meant to bootstrap a rig when you already have a git
  clone locally and don't want [rig.md](rig.md) `add` to clone it
  again. Worth a note once the rig workflow page lands.
- **`rig.AgentDirs` contents.** The loop iterates the exported slice
  from `internal/rig`, so the canonical list (and whether it
  includes `refinery/rig` vs `refinery/`) lives there — not in this
  file. Adding a `packages/rig.md` page (Sub B) would be the right
  place to pin it down.
- **Graceful failures dominate.** Steps 4, 5, and 6 all print
  warnings and continue rather than aborting. Only the two
  hard-fail paths are "not a git repo" and "rig already
  initialized". That makes `gt init` usable as an idempotent
  repair tool for existing rigs, at the cost of being quiet about
  partial-state problems.
- **Lifecycle config is silently best-effort.** If the rig is
  standalone (no enclosing HQ), the step at `init.go:101-108` is
  skipped without any output. Operators debugging missing patrols
  may not notice.
