---
title: gt uninstall
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/uninstall.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, lifecycle, destructive]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [precondition-violation, partial-completion]
---

# gt uninstall

Removes Gas Town from the system: shell integration, wrapper scripts,
and the three XDG directories (state, config, cache). The workspace
(`~/gt`) is preserved unless `--workspace` is explicitly passed.
Interactive confirmation by default; `--force` bypasses.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/uninstall.go:25-47`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`).
Note: `install` **is** exempt (`root.go:65`), but `uninstall` is not —
so removing Gas Town still goes through the beads version check.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/uninstall.go:25-171`.

### Invocation

```
gt uninstall [--workspace] [--force|-f]
```

### Behavior

`runUninstall` (`uninstall.go:57-150`):

1. **Confirmation gate** (`uninstall.go:58-85`): unless `--force`, prints
   the "This will remove Gas Town from your system" header and the
   bulleted list of removal targets (shell integration, wrapper
   scripts, state dir, config dir, cache dir). If `--workspace` is
   set, additionally prints a bold warning that the workspace will be
   deleted. Reads `y/yes` from stdin; anything else prints `Aborted.`
   and returns nil.
2. **Shell integration** (`uninstall.go:92-96`): `shell.Remove()`. On
   failure, appends to `errors` slice and continues.
3. **Wrapper scripts** (`uninstall.go:98-102`): `wrappers.Remove()`.
   The wrappers are `~/bin/gt-codex`, `~/bin/gt-gemini`, and
   `~/bin/gt-opencode` per the Long text at `uninstall.go:33`.
4. **State dir** (`uninstall.go:104-108`): `os.RemoveAll(state.StateDir())`
   — typically `~/.local/state/gastown/`. Only the true "does not
   exist" case is not treated as a failure.
5. **Config dir** (`uninstall.go:110-114`):
   `os.RemoveAll(state.ConfigDir())` — typically `~/.config/gastown/`.
6. **Cache dir** (`uninstall.go:116-120`):
   `os.RemoveAll(state.CacheDir())` — typically `~/.cache/gastown/`.
7. **Workspace (opt-in)** (`uninstall.go:122-131`): only if
   `--workspace`. Calls `findWorkspaceForUninstall` which looks for
   `~/gt` or `~/gastown` and requires the candidate to contain a
   `mayor/` subdirectory (`uninstall.go:152-171`).
8. **Error summary** (`uninstall.go:133-140`): if any step appended to
   `errors`, print them under a warning banner and return
   `uninstall incomplete`.
9. **Success message** (`uninstall.go:142-149`): prints
   `✓ Gas Town has been uninstalled` and echoes the `go install` +
   `gt install ~/gt --shell` reinstall incantation in dim text.

### `findWorkspaceForUninstall` (`uninstall.go:152-171`)

Two hardcoded candidates:
- `$HOME/gt`
- `$HOME/gastown`

For each, checks that `<candidate>/mayor` exists (`os.Stat`). Returns
the first match, or empty string if neither is a real workspace.
**This means workspaces in non-standard locations will be silently
skipped under `--workspace`** — no error, no warning. The state/config
dirs are still removed but the workspace is left in place.

### Subcommands

None (terminal command).

### Flags

Declared in `init()` (`uninstall.go:49-55`):

| Flag                  | Type | Default | Purpose                                            |
|-----------------------|------|---------|----------------------------------------------------|
| `--workspace`         | bool | `false` | Also remove the workspace directory (DESTRUCTIVE)  |
| `--force` / `-f`      | bool | `false` | Skip confirmation prompts                          |

### What is NOT removed

The `gt` binary itself is not touched. Uninstall removes what
`install` set up (shell hooks, wrappers, state) but it does not
uninstall the `gt` binary — the reinstall hint at `uninstall.go:146`
tacitly confirms this by telling the user to `go install ... gt`
before re-running `gt install`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/uninstall.go:29-45` — Cobra `Long` text on `uninstallCmd`.
- `/home/kimberly/repos/gastown/internal/cmd/uninstall.go:50-51` — `--workspace` flag help text.

### Verbatim

`uninstallCmd.Long` (`uninstall.go:29-45`):

> Completely remove Gas Town from the system.
>
> By default, removes:
>   - Shell integration (~/.zshrc or ~/.bashrc)
>   - Wrapper scripts (~/bin/gt-codex, ~/bin/gt-gemini, ~/bin/gt-opencode)
>   - State directory (~/.local/state/gastown/)
>   - Config directory (~/.config/gastown/)
>   - Cache directory (~/.cache/gastown/)
>
> The workspace (e.g., ~/gt) is NOT removed unless --workspace is specified.
>
> Use --force to skip confirmation prompts.
>
> Examples:
>   gt uninstall                    # Remove Gas Town, keep workspace
>   gt uninstall --workspace        # Also remove workspace directory
>   gt uninstall --force            # Skip confirmation

`--workspace` flag help (`uninstall.go:50-51`):

> Also remove the workspace directory (DESTRUCTIVE)

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### `--workspace` is a hardcoded two-location scan, silently no-ops for any other workspace path

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/uninstall.go:38` ("The workspace (e.g., ~/gt) is NOT removed unless --workspace is specified") and the `--workspace` flag description at `uninstall.go:50-51` ("Also remove the workspace directory (DESTRUCTIVE)"). The Examples block at `uninstall.go:44` (`gt uninstall --workspace        # Also remove workspace directory`) reinforces the reading: `--workspace` removes "the" workspace — singular, the user's workspace, whatever that is.
- **Code does:** `findWorkspaceForUninstall` at `/home/kimberly/repos/gastown/internal/cmd/uninstall.go:152-171` ignores any state, config, or environment hint about where the user's workspace lives. It tries exactly two hardcoded candidates — `$HOME/gt` and `$HOME/gastown` — and for each, requires a `mayor/` subdirectory to exist (`uninstall.go:163-168`) before returning that path. If neither candidate is a real workspace, it returns `""`. The caller at `uninstall.go:122-131` (the `--workspace` branch of `runUninstall`) then sees the empty string and — critically — **does not error, does not warn the user**. The state/config/cache dirs are still removed (as they would be without `--workspace`), but the workspace at a non-standard path is silently left in place. A user who kept their town at `~/code/gastown`, `~/work/gt`, or any other path runs `gt uninstall --workspace`, reads the confirmation prompt (which warns "WORKSPACE WILL BE DELETED" at `uninstall.go:70-72`), types `y`, and then discovers after the fact that their workspace is untouched — only the XDG directories are gone.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — either (a) narrow the `Long` text at `uninstall.go:29-45` and the `--workspace` flag help at `uninstall.go:50-51` to spell out the two-candidate scan and the "silently skipped if elsewhere" behavior, or (b) expand `findWorkspaceForUninstall` at `uninstall.go:152-171` to discover the real workspace from config (`state.Load()` already runs upstream; the install flow records the workspace root somewhere that could be read back). The minimal doc-only fix is to add one line to the `Long`: "With --workspace, removes ~/gt or ~/gastown if present; workspaces at other paths are not detected and must be removed manually." A better product fix is path (b). Even with path (b), the confirmation prompt should warn when no workspace is found, rather than silently proceeding with the XDG-only removal.
- **Release position:** `in-release` (`uninstallCmd.Long` at `v1.0.0:internal/cmd/uninstall.go:25-46` byte-identical; `findWorkspaceForUninstall`'s two-candidate scan already present at v1.0.0).

## Failure modes

### Precondition violations (what does it assume?)

- **`findWorkspaceForUninstall` only checks two hardcoded paths:** `uninstall.go:158-161` tries `$HOME/gt` and `$HOME/gastown` only. A workspace at any other path (e.g., `~/code/gt`, `~/work/gastown`) is silently not found. With `--workspace`, the confirmation prompt warns "WORKSPACE WILL BE DELETED" at `uninstall.go:70-72`, the user types `y`, and the workspace is silently left in place. **Absent** — predicted bug surface: no warning when `--workspace` finds nothing to delete; the user believes the workspace was removed. (Also documented as cobra-drift in the Drift section above.)
- **`uninstall` is not beads-exempt:** `root.go:44-77` lists `install` as beads-exempt (`root.go:65`) but not `uninstall`. If the beads database is corrupt or Dolt is down, the `persistentPreRun` beads version check may block `gt uninstall` from running at all. A user trying to clean up a broken installation cannot uninstall. **Absent** — predicted bug surface: the one command designed to remove everything is gated on the subsystem most likely to be broken.

### Partial completion (what doesn't it clean up?)

- **Multi-step removal continues on failure, reports at end:** `uninstall.go:87-140` collects errors from each removal step into a `[]string` and only returns `fmt.Errorf("uninstall incomplete")` at the end. Individual steps are independent, so a failure in shell removal does not block state-dir removal. This is **present** — good design: each step is attempted regardless of prior failures, and the user gets a complete error summary. However, the user must manually clean up any failed steps; there is no retry mechanism.
- **`--workspace` with empty `findWorkspaceForUninstall` result silently no-ops:** `uninstall.go:122-131` checks `workspaceDir != ""` before attempting removal. If `findWorkspaceForUninstall` returns `""` (non-standard path), the block is skipped entirely — no error, no warning. Combined with the confirmation prompt that said "WORKSPACE WILL BE DELETED," this is a false promise. **Absent** — no warning that the workspace was not found after the user confirmed deletion.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [shell.md](shell.md) — `shell remove` is a strict subset of
  uninstall's shell-integration step.
- [disable.md](disable.md) — `disable --clean` is a softer option:
  flips the state to disabled and removes shell integration, but
  preserves wrappers, state, config, cache, and workspace.
- [doctor.md](doctor.md) — after uninstall, `gt doctor` will fail on
  almost every check; the reinstall flow rebuilds state.
- [install.md](install.md) — uninstall's counterpart; see Long
  text hint `gt install ~/gt --shell` at `uninstall.go:147`. The
  reinstall flow at `uninstall.go:142-149` points users back at
  `gt install ~/gt --shell`.

## Notes / open questions

- The workspace-detection heuristic is narrow → promoted to `## Drift` above (`--workspace` cobra-drift finding).
- State/config/cache paths are returned by `state.StateDir()`,
  `state.ConfigDir()`, `state.CacheDir()`
  (`uninstall.go:64-66`) — worth pinning in a `packages/state.md`
  page to document what XDG roots these resolve to across macOS vs
  Linux (macOS does not honor `XDG_*` by default in most Go
  libraries).
- The confirmation is read from `os.Stdin` via `bufio.NewReader`
  (`uninstall.go:77`), so piping `y\n` works but a non-TTY stdin may
  hang depending on `ReadString('\n')` semantics. Consider a CI test
  for `--force`.
- Re-running `uninstall` after a previous successful uninstall will
  print several warnings but ultimately return success as long as all
  components report their "already removed" state cleanly — the
  idempotency story is worth verifying.
