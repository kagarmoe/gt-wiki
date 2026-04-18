---
title: internal/cli
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cli/name.go
tags: [package, platform-service, cli, naming, env-var]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 2}
---

# internal/cli

A single-file helper package whose sole job is to resolve the Gas Town CLI
command name (default `gt`) with one-time caching and an environment-variable
override.

**Go package path:** `github.com/steveyegge/gastown/internal/cli`
**File count:** 1 go file, 1 test file
**Imports (notable):** stdlib only (`os`, `sync`)
**Imported by (notable):** [`cmd/gt`](../binaries/gt.md) root command
(`/home/kimberly/repos/gastown/internal/cmd/root.go:13`) and any code path
that needs to produce help text, error messages, or generated shell snippets
that embed the user-visible binary name.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cli/name.go:1-26`.

### Public API

- `cli.Name() string` — returns the resolved CLI command name
  (`/home/kimberly/repos/gastown/internal/cli/name.go:17-25`). Reads
  `GT_COMMAND` from the environment on first call and caches the result via
  `sync.Once`. Falls back to the string literal `"gt"` if the env var is
  unset or empty.

### Internals / Notable implementation

- Package-level mutable state: `var name string` and `var nameOnce sync.Once`
  at `/home/kimberly/repos/gastown/internal/cli/name.go:9-12`. This is the
  single gotcha of the package: because `nameOnce` only fires once per
  process, changing `GT_COMMAND` after the first `cli.Name()` call has no
  effect. Tests that mutate the env var must reset both the variable and the
  `Once` (or run in a subprocess).
- No `init()` function; the cache is populated lazily on first caller.

### Usage pattern

`cli.Name()` is called in exactly two contexts:

1. **At program startup** — [`gt`](../binaries/gt.md)'s root cobra command
   sets `rootCmd.Use = cli.Name()` so every subcommand's help text and
   completion output uses the overridden name.
2. **In user-facing strings** — error messages and hints that tell the user
   what to run next (e.g. "run `gt doctor`") substitute `cli.Name()` so
   copy-paste suggestions work even when the binary has been renamed.

### Why this exists

The comment at
`/home/kimberly/repos/gastown/internal/cli/name.go:14-16` explains: another
popular tool called `gt` exists (Graphite, the stacked-PR CLI). Users who
already have `gt` on their `PATH` can install Gas Town under a different
name and set `GT_COMMAND` so all help output and error hints reflect the
chosen alias.

## Related wiki pages

- [gt](../binaries/gt.md) — main binary; embeds `cli.Name()` in its root
  cobra command and help text.
- [internal/config](config.md) — reads many other `GT_*` env vars and forms
  the broader config surface that `GT_COMMAND` lives within.
- [go-packages inventory](../inventory/go-packages.md) — package listing
  that classifies `cli` as a thin package.

## Notes / open questions

- `GT_COMMAND` is not documented in any `.md` under `gastown/docs/` as of
  this page's write; it lives only in the code comment. Drift candidate for
  a future pass if the package ever grows help text.
- The `sync.Once` cache is process-wide and not reset between tests; the
  companion `name_test.go` presumably uses subprocess tests or resets
  private state via export_test.go. (Not verified — source-grounded only.)
