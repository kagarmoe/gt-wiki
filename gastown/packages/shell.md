---
title: internal/shell
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/shell/integration.go
tags: [package, shell, integration, rc-file, hook, zsh, bash]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/shell

Shell-integration installer: rewrites the user's `.bashrc` or `.zshrc`
to source a Gas Town hook script that detects when the user `cd`s into
a git repo and (if Gas Town is enabled and the repo isn't ignored)
offers to add it as a rig or exports `GT_TOWN_ROOT` / `GT_RIG`. The
package owns both the install/remove paths in the RC file and the
literal shell script that gets written out. Backs the
[`gt shell`](../commands/shell.md) install surface.

**Go package path:** `github.com/steveyegge/gastown/internal/shell`
**File count:** 1 go file (`integration.go`), 1 test file.
**Imports (notable):** stdlib (`fmt`, `os`, `path/filepath`, `strings`)
plus `github.com/steveyegge/gastown/internal/state` for
`state.ConfigDir()` (hook script install location) and
`state.SetShellIntegration` (records which shell was configured).
**Imported by (notable):** the [`gt shell`](../commands/shell.md)
command group (also `gt install --shell`, which calls the same
functions).

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/shell/integration.go`.

### Install / Remove

- `Install() error` (`integration.go:25-38`):
  1. `DetectShell()` picks `zsh` or `bash` from `$SHELL`.
  2. `writeHookScript()` writes `~/.config/gastown/shell-hook.sh`
     with the literal `shellHookScript` string
     (`integration.go:77-85`, perms 0644).
  3. `addToRCFile(rcPath)` edits the shell's RC file
     (`integration.go:87-108`).
  4. `state.SetShellIntegration(shell)` records the install.
- `Remove() error` (`integration.go:40-54`):
  1. `removeFromRCFile(rcPath)` strips the managed block.
  2. Deletes `~/.config/gastown/shell-hook.sh`; missing file is OK.
- `DetectShell() string` (`integration.go:56-65`) inspects `$SHELL`.
  Ends-with check: `zsh` wins if present, then `bash`, default `zsh`.
  Non-zsh-non-bash shells get treated as zsh.
- `RCFilePath(shell) string` (`integration.go:67-75`) — `.bashrc` for
  bash, `.zshrc` for everything else.

### Managed block in the RC file

The block is delimited by two markers
(`integration.go:15-18`):

```
# --- Gas Town Integration (managed by gt) ---
[[ -f "<config-dir>/shell-hook.sh" ]] && source "<config-dir>/shell-hook.sh"
# --- End Gas Town ---
```

- `addToRCFile(path)` (`integration.go:87-108`):
  - Reads the RC file (non-existence is OK — becomes an empty string).
  - If `markerStart` is already present, defers to `updateRCFile` to
    replace the block in place — idempotent.
  - Otherwise writes a backup to `<rcPath>.gastown-backup` (only if
    the RC file was non-empty) and appends the new block.
- `removeFromRCFile(path)` (`integration.go:110-138`):
  - Locates `markerStart` and the subsequent `markerEnd`.
  - Trims one trailing newline after the block and one leading
    newline before the block to avoid blank-line accretion on
    repeated install/remove cycles.
- `updateRCFile(path, content)` (`integration.go:140-152`) — in-place
  block replacement used when the markers already exist. Returns an
  error on `malformed Gas Town block in <path>` when `markerEnd` is
  missing.

### The embedded shell hook script (`shellHookScript`)

Lives as a string literal at `integration.go:154-304`. Structure:

- **Feature gates**:
  - `_gastown_enabled` — reads `$GASTOWN_DISABLED`, `$GASTOWN_ENABLED`,
    then checks `~/.local/state/gastown/state.json` for
    `"enabled": true`.
  - `_gastown_ignored` — walks up from `$PWD` looking for a
    `.gastown-ignore` marker.
  - `_gastown_already_asked` / `_gastown_mark_asked` — uses
    `~/.cache/gastown/asked-repos` so the "add to Gas Town?" prompt
    never fires twice for the same repo root.
- **`_gastown_offer_add`** — the interactive "Add '<repo>' to Gas Town?
  [y/N/never]" prompt. `y` runs `gt rig quick-add <repo> --yes` and
  if that outputs a `GT_CREW_PATH=<path>` line, `cd`s into that path
  and re-runs `_gastown_hook`. `never` creates `.gastown-ignore` in
  the repo. Non-TTY stdin short-circuits out — the prompt only fires
  in interactive shells.
- **`_gastown_hook`** — the per-prompt hook. Checks enabled, checks
  ignored, checks `git rev-parse --git-dir`, then:
  - Tries to read the cached mapping from
    `~/.cache/gastown/rigs.cache` (format `<repo_root>:<shell-exports>`,
    `eval`'d directly).
  - On cache miss, runs `gt rig detect <repo_root>` and `eval`s the
    output (which may set `GT_TOWN_ROOT` / `GT_RIG`).
  - If detection found a town, fires
    `gt rig detect --cache <repo>` in the background to populate
    the cache.
  - If `_GASTOWN_OFFER_ADD` is set (only `_gastown_chpwd_hook` sets
    it), runs `_gastown_offer_add` for repos not yet in a town.
- **`_gastown_chpwd_hook`** — thin wrapper that sets
  `_GASTOWN_OFFER_ADD=1` and calls `_gastown_hook`. Used to
  distinguish "user cd'd here" from "shell prompt redraw".
- **Shell-specific activation**:
  - zsh: `add-zsh-hook chpwd _gastown_chpwd_hook; add-zsh-hook precmd
    _gastown_hook`.
  - bash: prepends `_gastown_chpwd_hook` to `PROMPT_COMMAND` (idempotent
    — checks for an existing `_gastown_hook` entry).

## Related wiki pages

- [gt shell](../commands/shell.md) — the CLI surface that calls
  `Install` / `Remove`.
- [gt install](../commands/install.md) — also calls `Install` via
  `gt install --shell`.
- [internal/state](../inventory/go-packages.md) — supplies
  `ConfigDir()` and `SetShellIntegration()`. (The
  `~/.local/state/gastown/state.json` consulted inside the hook
  script is the same file written by this package.)
- [gt](../binaries/gt.md).
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The hook script shebang is `#!/bin/zsh` even though the script is
  also sourced by bash. This is cosmetic — shebangs in sourced files
  are ignored — but worth noting for anyone who greps for shebangs.
- `DetectShell` falls through to `"zsh"` on any unrecognized `$SHELL`
  value. Fish, tcsh, pwsh users get a zsh RC file edit that will not
  load correctly — the script itself only handles zsh and bash (see
  the `case "${SHELL##*/}"` near `integration.go:290`). This is a
  known drift surface.
- `addToRCFile` writes `.gastown-backup` only once, non-destructively,
  and only if the RC file was non-empty. Repeated installs do not
  rotate the backup; the original first-install backup is preserved.
- The `_gastown_offer_add` path uses `read ... </dev/tty` so it
  survives when stdin is not a TTY — but it also requires `/dev/tty`
  to exist and be writable, which is not the case inside some CI /
  container environments. The interactive prompt simply returns
  early when `[[ -t 0 ]]` is false, so CI shouldn't hit this code
  path in practice.
- The rigs.cache eval-every-prompt pattern runs arbitrary code read
  from a cache file. Not a security issue in practice (the file is
  user-owned under `~/.cache/`), but worth flagging as a trust
  assumption.
