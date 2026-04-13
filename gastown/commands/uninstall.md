---
title: gt uninstall
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/uninstall.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, lifecycle, destructive]
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

- The workspace-detection heuristic is narrow: only `~/gt` and
  `~/gastown`, only if they contain a `mayor/` dir. Users who kept
  their town at `~/code/gastown` or `~/work/gt` will not have their
  workspace removed by `--workspace` and will get no warning.
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
