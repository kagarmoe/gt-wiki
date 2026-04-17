---
title: .golangci.yml
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/.golangci.yml
tags: [file, lint, golangci-lint, quality]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# .golangci.yml

golangci-lint v2 configuration at
`/home/kimberly/repos/gastown/.golangci.yml`. Governs what `golangci-lint
run` checks across the project. Five linters enabled, heavy
per-package exclusion list that suppresses `gosec` findings inside
`internal/` and `_test.go` files.

`golangci-lint` itself is **not** wired into [flake.nix](flake-nix.md)'s
dev shell or [Makefile](makefile.md)'s targets — developers install it
separately and invoke it by hand.

## What it actually does

### Schema version

`.golangci.yml:1`:

```yaml
version: "2"
```

golangci-lint v2 config format (distinct from v1 — v2 introduced the
nested `linters.settings` / `linters.exclusions` shape used below).

### Run settings

`.golangci.yml:3-5`:

```yaml
run:
  timeout: 5m
  tests: false
```

- `timeout: 5m` — 5-minute cap on the full lint run.
- `tests: false` — **skip test files entirely** during linting.
  This means the project's test code is never linted. Note: some of
  the `exclusions.rules` below still reference `_test\.go` paths —
  those rules are effectively dead code given `tests: false`, but
  they'd re-activate if `tests` were flipped on.

### Linters enabled

`.golangci.yml:7-14`:

```yaml
linters:
  default: 'none'
  enable:
    - errcheck
    - gosec
    - misspell
    - unconvert
    - unparam
```

`default: 'none'` turns off every golangci-lint default, then enables
exactly five:

- **`errcheck`** — unhandled error returns.
- **`gosec`** — security issue scanner (G-code findings).
- **`misspell`** — spelling in comments/strings.
- **`unconvert`** — unnecessary type conversions.
- **`unparam`** — unused function parameters.

Everything else (`govet`, `staticcheck`, `ineffassign`, etc.) is
**disabled**. This is a minimal-gate policy — the set is smaller than
golangci-lint's default.

### Linter settings

#### errcheck

`.golangci.yml:17-33`:

```yaml
errcheck:
  exclude-functions:
    - (*database/sql.DB).Close
    - (*database/sql.Rows).Close
    - (*database/sql.Tx).Rollback
    - (*database/sql.Stmt).Close
    - (*database/sql.Conn).Close
    - (*os.File).Close
    - (net.Conn).Close
    - os.RemoveAll
    - os.Remove
    - os.Setenv
    - os.Unsetenv
    - os.Chdir
    - os.MkdirAll
    - fmt.Sscanf
    - fmt.Fprintf
    - fmt.Fprintln
```

Sixteen functions whose error returns may be ignored without
errcheck complaint. Seven are `Close`/`Rollback` methods on database
and OS handles. Five are `os` package file operations. Three are
`fmt` print/scan functions.

The `fmt.Fprintf`/`fmt.Fprintln` exclusions are broad — stderr writes
via `fmt.Fprintln(os.Stderr, ...)` won't be flagged for ignoring
errors.

#### misspell

`.golangci.yml:35-36`:

```yaml
misspell:
  locale: US
```

US-English spell checker.

### Exclusions

`.golangci.yml:38-108`. A long rule list, grouped by category below.

#### `gosec` G304 (file inclusion via variable)

Two exclusions (`.golangci.yml:41-50`):

1. In `_test\.go`: G304 off. Comment: "File inclusion via variable in
   tests is safe (test data)".
2. In `internal/`: G304 off. Comment: "Config/state file loading uses
   constructed paths, not user input".

#### `gosec` G306 (file permissions)

Two exclusions (`.golangci.yml:51-55`):

1. In `_test\.go`: G306 off. Comment: "File permissions 0644 in tests
   are acceptable (test fixtures)".

#### `gosec` G306/G302 combined

One exclusion (`.golangci.yml:56-61`) for `internal/`:

```
Internal packages write non-sensitive operational data files
```

Matches both `G306` and `G302` in `internal/`.

#### `gosec` G302/G301 with specific permission patterns

One unscoped exclusion (`.golangci.yml:63-65`):

```yaml
- linters:
    - gosec
  text: "G302.*0700|G301.*0750"
```

Directory perms `0700` and `0750` excluded everywhere (no `path:`
scope). Comment: "Directory/file permissions 0700/0750 are
acceptable".

#### `gosec` G404 (weak random in `internal/`)

`.golangci.yml:67-70`: `math/rand` allowed in `internal/`. Comment:
"math/rand is fine for jitter/backoff calculations, not security".

#### `gosec` G104 (unhandled errors in `internal/`)

`.golangci.yml:72-75`. Covered by errcheck exclusions already — this
turns off gosec's duplicate check.

#### `gosec` G204 (subprocess execution in `internal/`)

`.golangci.yml:77-81`. Comment: "Safe subprocess launches with
validated arguments (internal tools)". This is a meaningful waiver —
gastown runs `bd`, `dolt`, `tmux`, `git`, etc. as subprocesses, and
without this exclusion every `exec.Command` call would be flagged.

#### `gosec` G115 (integer narrowing in `internal/`)

`.golangci.yml:83-86`. Comment: "Integer narrowing conversions in
internal code are intentional".

#### `gosec` G117 (hardcoded password detection in `internal/`)

`.golangci.yml:88-91`. Comment: "Password fields in config/DB structs
are not hardcoded secrets".

#### `gosec` G201 (SQL string formatting in `internal/`)

`.golangci.yml:93-97`. Comment: "Internal code builds queries with
database/table names from internal config, not user input".

Important context: gastown talks to Dolt via raw SQL with dynamic
database/table names (per [../README.md](../README.md) coordinates —
beads data lives in Dolt). This exclusion is what lets
`fmt.Sprintf("SELECT * FROM `%s`", dbName)` pass lint.

#### `gosec` G702–G705 (taint analysis false positives)

`.golangci.yml:99-103`. Broadly suppresses gosec's taint analysis
inside `internal/`.

#### `errcheck` in test files

`.golangci.yml:105-108`:

```yaml
- path: '_test\.go'
  linters:
    - errcheck
  text: "Error return value of .*(Close|Rollback|RemoveAll|Setenv|Unsetenv|Chdir|MkdirAll|Remove|Write).* is not checked"
```

Unchecked cleanup errors allowed in test files. Note: dead given
`run.tests: false` above — but written such that flipping `tests: on`
wouldn't immediately flood the output.

### Issues

`.golangci.yml:110-111`:

```yaml
issues:
  uniq-by-line: true
```

Deduplicate findings that land on the same line from different
linters. A line reported by both errcheck and gosec shows up once.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — the code under `internal/cmd/`
  whose `exec.Command` calls benefit from the G204 exclusion.
- [flake.nix](flake-nix.md) — dev shell that does **not** ship
  `golangci-lint`. Users of `nix develop` need an external install.
- [Makefile](makefile.md) — has no `lint` target. There's no
  project-scripted lint invocation.
- [go.mod](go-mod.md) — the module tree this config lints over.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Notes / open questions

- **Who runs lint?** Not the Makefile, not the Nix dev shell, and the
  dockerfiles don't invoke it either. Likely only CI (see
  `.github/workflows/` — out of scope for this page).
- **`tests: false` plus `_test.go` exclusion rules.** The test-file
  rules are dead code under current config but are written to survive
  a `tests: true` toggle. Either the rules were retained defensively
  or `tests` was recently flipped off.
- The gosec exclusion list is nearly exhaustive for `internal/` — in
  practice gosec only inspects non-`internal/` code (which is
  basically `cmd/` entry points). The effective coverage of `gosec`
  in this project is small.
- `golangci-lint` version is not pinned in this file. The v2 schema
  implies v2.0 or later but the exact tool version is decided by
  whatever CI / developer machine runs the check.
- `errcheck`'s exclusion for `fmt.Fprintln` is broad enough to hide
  real bugs — e.g., a `fmt.Fprintln` to a pipe that's been closed
  will silently drop output.
