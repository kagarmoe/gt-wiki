---
title: internal/wrappers
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/wrappers/wrappers.go
  - /home/kimberly/repos/gastown/internal/wrappers/scripts/gt-codex
  - /home/kimberly/repos/gastown/internal/wrappers/scripts/gt-gemini
  - /home/kimberly/repos/gastown/internal/wrappers/scripts/gt-opencode
tags: [package, wrappers, install, shell-scripts, embed]
phase3_audited: 2026-04-14
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# internal/wrappers

Installs and removes the small shell-script wrappers that Gas Town puts
on the user's `$PATH` for non-Claude agentic coding tools: `gt-codex`,
`gt-gemini`, and `gt-opencode`. Each wrapper runs `gt prime` first
(injecting the Gas Town role context for the session) and then execs the
underlying tool. The scripts themselves are embedded via `embed.FS`
under `scripts/`, so the binary ships them — there is no separate
scripts directory to maintain in sync.

**Go package path:** `github.com/steveyegge/gastown/internal/wrappers`
**File count:** 1 go file, 1 test file, plus a `scripts/` subdirectory
with three plain shell scripts.
**Imports (notable):** stdlib only (`embed`, `fmt`, `os`,
`path/filepath`). No gastown-internal imports.
**Imported by (notable):** the [`gt install`](../commands/install.md)
command (calls `Install`) and the [`gt shell`](../commands/shell.md) /
uninstall path (calls `Remove`). `BinDir` is referenced by any doctor
or install-status path that needs to point the user at
`~/bin`.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/wrappers/wrappers.go`.

### The wrapper list

`wrappers` is a hard-coded slice `{"gt-codex", "gt-gemini",
"gt-opencode"}` appearing in both `Install` (`wrappers.go:26`) and
`Remove` (`wrappers.go:48`). Adding a new wrapper requires:

1. Adding the script under `scripts/`.
2. Updating the slice in both places — there is no constant or
   single list to edit.

### `Install() error` (`wrappers.go:16-40`)

1. Resolve the target bin dir via `binPath()` — `~/bin` unconditionally.
2. `os.MkdirAll(binDir, 0755)` — create it if missing.
3. For each wrapper name:
   - `scriptsFS.ReadFile("scripts/" + name)` — pull the embedded
     content out of the `embed.FS`.
   - `os.WriteFile(destPath, content, 0755)` — write it executable.

The 0755 mode means the wrappers are executable by everyone, not just
the owner. `os.WriteFile` clobbers any existing file at the destination
unconditionally — there is no "preserve existing" branch like in
`templates.CreateMayorCLAUDEmd`. An installation always wins over
whatever was there before.

### `Remove() error` (`wrappers.go:42-57`)

Symmetric: for each wrapper, `os.Remove(destPath)`, swallowing
`os.IsNotExist` so removing a partially-installed or already-removed
state is idempotent.

### `binPath() / BinDir()` (`wrappers.go:59-70`)

- `binPath()` (unexported) returns `~/bin` from `os.UserHomeDir()`,
  propagating any home-dir error.
- `BinDir()` (exported) calls `binPath()` but **swallows the error**
  and returns the empty string on failure. Callers that just want a
  path to show the user (e.g., "installed to <dir>") use the exported
  one; callers that need to actually write there use the internal one
  via `Install`/`Remove`.

### Embedded scripts (`//go:embed scripts/*`, `wrappers.go:13-14`)

The `embed.FS` at `scriptsFS` captures everything under `scripts/`.
On-disk at build time:

- `scripts/gt-codex`
- `scripts/gt-gemini`
- `scripts/gt-opencode`

Each is a small shell script that does `gt prime` first and then runs
the wrapped tool. The scripts are data to this package — they aren't
parsed or templated; the same bytes that sit in `scripts/` during
development get written verbatim to `~/bin/`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/wrappers/wrappers.go:1-2` — ABOUTME header

### Verbatim
> ABOUTME: Manages wrapper scripts for non-Claude agentic coding tools.
> ABOUTME: Provides gt-codex and gt-opencode wrappers that run gt prime first.

## Drift

### ABOUTME header omits gt-gemini wrapper
- **Claim source:** ABOUTME header at `wrappers.go:1-2`
- **Docs claim:** "Provides gt-codex and gt-opencode wrappers" — lists two wrappers.
- **Code does:** Installs three wrappers: `gt-codex`, `gt-gemini`, `gt-opencode` (`wrappers.go:26`). The `gt-gemini` script exists in `scripts/` and is embedded via `embed.FS`.
- **Category:** `cobra drift` (in-code docstring contradicts code)
- **Severity:** `wrong`
- **Fix tier:** `code` (update ABOUTME header to list all three wrappers)
- **Release position:** `in-release` (`gt-gemini` existed at v1.0.0; the ABOUTME header was already incomplete at that tag)

## Notes / open questions

- Wrapper list duplicated in `Install` and `Remove` — two copies of the
  same slice (`wrappers.go:26` and `:48`) that must stay in sync.
  Trivially refactorable into a package-level `var wrapperNames = [...]`
  but not currently done.
- `BinDir()` returns `""` on error while `binPath()` returns an error.
  Different error semantics for the same underlying call. A caller that
  accidentally uses `BinDir()` for a write target will fail with
  "open : no such file or directory" instead of a meaningful
  home-dir error.
- `~/bin` is assumed to be on `$PATH`. The package doesn't check this.
  `gt install` or `gt doctor` is the right place to warn the user, not
  here.
- There is no "update in place" path — `Install` just overwrites. A
  user who modified their `gt-codex` script to add logging will lose
  it on the next `gt install` run. Probably fine given the scripts
  are trivial, but worth noting.
- The embedded scripts themselves (not shown in detail here) are short
  enough that any behavior change to "how wrappers inject Gas Town
  context" happens by editing them as plain shell, not by touching
  this Go file.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [gt install](../commands/install.md) — primary caller of
  `wrappers.Install`.
- [gt shell](../commands/shell.md) — shell-integration command that
  sits alongside wrapper installation on the "make Gas Town visible
  from the user's normal shell" side.
- [go-packages inventory](../inventory/go-packages.md).
