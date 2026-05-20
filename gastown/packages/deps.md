---
title: internal/deps
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-05-20
sources:
  - /home/kimberly/repos/gastown/internal/deps/beads.go
  - /home/kimberly/repos/gastown/internal/deps/dolt.go
  - /home/kimberly/repos/gastown/internal/deps/claude.go
  - /home/kimberly/repos/gastown/internal/deps/version.go
tags: [package, diagnostics, dependencies, prereqs, versioning, beads, dolt, claude-code]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/deps

Prerequisite checks for the external binaries Gas Town depends on:
`bd` (beads) and `dolt` (both required), plus `claude` (Claude Code,
optional — added in v1.0.1). Defines the minimum supported version for
each, implements a small semver comparator, and — for beads only — knows
how to go-install a missing or outdated copy into `~/.local/bin`.

**Go package path:** `github.com/steveyegge/gastown/internal/deps`
**File count:** 4 go files, 3 test files
**Imports (notable):** stdlib (`context`, `os/exec`, `regexp`, `strconv`,
`strings`, `time`, `path/filepath`) plus [`internal/util`](util.md) for
`SetDetachedProcessGroup`.
**Imported by:**
[`internal/doctor`](doctor.md) (dolt, beads, and claude binary checks),
[`internal/daemon`](../inventory/go-packages.md),
[`internal/cmd/rig.go`](../commands/rig.md),
[`internal/cmd/install.go`](../commands/install.md), and
`internal/cmd/beads_version.go` (no wiki page yet). The package
is the single source of truth for `MinBeadsVersion` / `MinDoltVersion`
across all of them — `doctor/*_binary_check_test.go` even reads the
constants directly so version bumps propagate automatically.

## What it actually does

Source files: `beads.go`, `dolt.go`, `claude.go`, `version.go`.

### Public API

**Version constants and install hints:**

- `MinBeadsVersion = "0.57.0"` (`beads.go:19`) — bump this when Gas Town
  requires new beads features.
- `BeadsInstallPath = "github.com/steveyegge/beads/cmd/bd@latest"`
  (`beads.go:22`) — the go-install path used by both
  `EnsureBeads` and rendered into user-facing error messages.
- `MinDoltVersion = "1.82.4"` (`dolt.go:16`).
- `DoltInstallURL = "https://github.com/dolthub/dolt#installation"`
  (`dolt.go:19`) — dolt is not go-installable from source like beads, so
  `deps` only knows how to tell the user where to go, not how to install
  it.
- `MinClaudeCodeVersion = "2.0.20"` (`claude.go:16`) — the version that
  introduced the Skills system Gas Town relies on (crew-commit,
  ghi-list, pr-list, pr-sheriff). See
  `docs/design/claude-code-minimum-version.md`.
- `RecommendedClaudeCodeVersion = "2.1.3"` (`claude.go:20`) — where
  skills and slash commands merged into a unified system. Versions
  between the minimum and recommended are tolerated (`ClaudeCodeOldButOK`).
- `ClaudeCodeInstallURL = "https://claude.ai/claude-code"`
  (`claude.go:23`) — like dolt, Claude Code is not go-installable, so
  `deps` only points the user at the install page.

**Status enums:**

- `BeadsStatus` (`beads.go:25-32`): `BeadsOK | BeadsNotFound | BeadsTooOld
  | BeadsUnknown` — no `ExecFailed` variant; beads either isn't in PATH
  or is, and "we ran it but couldn't parse the output" maps to
  `BeadsUnknown`.
- `DoltStatus` (`dolt.go:22-30`): `DoltOK | DoltNotFound | DoltTooOld |
  DoltExecFailed | DoltUnknown`. Dolt *does* distinguish `ExecFailed`
  (binary present but `dolt version` returned a non-zero exit) from
  `Unknown` (returned zero but output didn't match the regex), because
  broken dolt installs are a real operational hazard.
- `ClaudeCodeStatus` (`claude.go:26-35`): `ClaudeCodeOK | ClaudeCodeNotFound
  | ClaudeCodeTooOld | ClaudeCodeOldButOK | ClaudeCodeExecFailed |
  ClaudeCodeUnknown`. The richest of the three enums. It carries both
  the `DoltStatus`-style `ExecFailed`/`Unknown` split **and** a unique
  `OldButOK` variant for the "above minimum but below recommended"
  middle band — Claude Code has a two-tier version policy, where `bd`
  and `dolt` have only a single floor.

**Check functions:**

- `CheckBeads() (BeadsStatus, string)` (`beads.go:36-67`) — calls
  `exec.LookPath("bd")`, then runs `bd version` under a 10-second
  `context.WithTimeout` (justified at `beads.go:45-47` as "necessary:
  under heavy CI load (parallel test packages), even a trivial shell
  script can take >3s to start"), parses the output via
  `parseBeadsVersion`, and compares against `MinBeadsVersion`.
- `CheckDolt() (DoltStatus, string, string)` (`dolt.go:35-64`) — same
  shape as `CheckBeads`, with three key differences: (1) it
  additionally returns a diagnostic `detail` string for the
  `DoltExecFailed` case containing `fmt.Sprintf("at %s: %s", path,
  detail)`; (2) it uses `cmd.CombinedOutput()` instead of `cmd.Output()`
  so stderr ends up in the detail string; (3) the `DoltUnknown` case
  returns the raw (trimmed) output so the user sees what went wrong
  parsing.
- `CheckClaudeCode() (ClaudeCodeStatus, string)` (`claude.go:39-68`) —
  same shape as `CheckDolt`: `exec.LookPath("claude")`, then
  `claude --version` under a 10-second `context.WithTimeout` with
  `util.SetDetachedProcessGroup`, captured via `cmd.CombinedOutput()`.
  Returns `ClaudeCodeNotFound` if not on PATH, `ClaudeCodeExecFailed`
  if the version command errors, `ClaudeCodeUnknown` if the output
  doesn't parse, then the three-way version verdict:
  below `MinClaudeCodeVersion` → `ClaudeCodeTooOld`; below
  `RecommendedClaudeCodeVersion` → `ClaudeCodeOldButOK`; else
  `ClaudeCodeOK`. Returns the parsed version string alongside the status
  for `TooOld`/`OldButOK`/`OK`.

**Installer (beads only):**

- `EnsureBeads(autoInstall bool) error` (`beads.go:72-95`) — the
  "check, and optionally fix" variant. `BeadsOK` → `nil`; `BeadsTooOld`
  → error with a "go install" hint (**does not auto-upgrade** even when
  `autoInstall == true`, because an upgrade might be a breaking
  change); `BeadsUnknown` → `nil` with an implicit warning (proceed
  anyway); `BeadsNotFound` → error or call `installBeads()` depending
  on `autoInstall`.
- `installBeads()` (`beads.go:100-122`, unexported) — runs `go install
  github.com/steveyegge/beads/cmd/bd@latest` with **`GOBIN` forced to
  `~/.local/bin`** via `appendGOBIN`. This is load-bearing: the
  comment at `beads.go:97-99` and `beads.go:124-126` explains that the
  default `$GOPATH/bin` (`~/go/bin/`) creates a stale shadow copy
  problem, so Gas Town canonicalizes on `~/.local/bin`. Post-install it
  re-runs `CheckBeads` to verify the install landed on PATH and that
  the version is now compatible.

**Version comparator (both binaries use this):**

- `CompareVersions(a, b string) int` (`version.go:10-23`) — returns
  `-1 / 0 / 1` for `<`, `==`, `>`. Only compares the first three
  components; ignores `-rc`, `-dev`, build metadata, etc.
- `ParseVersion(v string) [3]int` (`version.go:28-35`) — naive
  `strings.Split(".")` → `strconv.Atoi` with non-numeric parts
  silently defaulting to 0. The comment warns: callers **must**
  pre-sanitize via regex (which `parseBeadsVersion` and
  `parseDoltVersion` both do).

### Internals / Notable implementation

- `parseBeadsVersion` (`beads.go:144-152`) matches
  `bd version (\d+\.\d+\.\d+)`, tolerating trailing `(dev: ...)` noise.
  `parseDoltVersion` (`dolt.go:67-74`) matches `dolt version
  (\d+\.\d+\.\d+)`. Both regexes are anchored to the binary's own
  version-line prefix, so an attacker substituting a malicious binary
  named `bd` would at minimum need to print a plausible version line.
- `parseClaudeCodeVersion` (`claude.go:72-80`) is looser: it matches
  `^(\d+\.\d+\.\d+)` against the trimmed output (handling both
  `"2.1.101 (Claude Code)"` and a bare `"2.1.101"`). Unlike the bd/dolt
  parsers it is *not* anchored to a binary-name prefix, so any leading
  semver triple is accepted.
- `appendGOBIN` (`beads.go:127-141`) walks the existing environ to
  replace any inherited `GOBIN=...` rather than stacking a second
  definition — otherwise `go install` would use the last entry and Gas
  Town couldn't override a user-exported GOBIN.
- All four exec calls (`bd version`, `dolt version`, `claude --version`,
  `go install`) run with `util.SetDetachedProcessGroup`, so a hanging
  binary can be cleanly killed via its process group.
- No retry logic. Every exec call is one-shot with a 10s timeout.

### Usage pattern

`deps` is the shared foundation under every diagnostic surface that
checks external tooling:

- [`internal/doctor/beads_binary_check.go`](doctor.md) and
  `dolt_binary_check.go` translate `CheckBeads`/`CheckDolt` results into
  doctor `CheckResult`s with `FixHint` strings rendered from
  `BeadsInstallPath` / `DoltInstallURL`.
- Command-side installers in
  [`gt install`](../commands/install.md) and
  [`gt rig`](../commands/rig.md) call `EnsureBeads(autoInstall=true)`
  during setup flows.
- `gt beads-version` exposes `MinBeadsVersion` / `CheckBeads` output
  directly so users can see what's required vs. installed (command page
  not yet mapped in the wiki).
- [`internal/daemon`](../inventory/go-packages.md) checks binary
  freshness on startup (not yet mapped).

The doctor tests at `doctor/beads_binary_check_test.go:76-90` import
`deps.MinBeadsVersion` directly so the tests stay in sync whenever the
constant is bumped — this is the intended pattern for downstream code:
**never hardcode versions, always read `deps.Min*Version`**.

## Related wiki pages

- [gt](../binaries/gt.md) — parent binary.
- [internal/doctor](doctor.md) — primary consumer; wraps `CheckBeads` /
  `CheckDolt` / `CheckClaudeCode` into doctor `CheckResult`s with fix
  hints.
- [gt doctor](../commands/doctor.md) — CLI that surfaces these checks.
- [gt health](../commands/health.md) and [gt vitals](../commands/vitals.md)
  — sibling diagnostic commands (they don't currently import `deps`
  directly).
- [gt install](../commands/install.md) and [gt rig](../commands/rig.md) —
  auto-install entry points.
- [internal/util](util.md) — `SetDetachedProcessGroup` wrapper used on
  every subprocess in this package.
- [go-packages inventory](../inventory/go-packages.md).

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `bd` | `version` | (none) | hardcoded | `beads.go:49` |
| `go` | `install` | `BeadsInstallPath` | hardcoded | `beads.go:103` |
| `dolt` | `version` | (none) | hardcoded | `dolt.go:43` |
| `claude` | `--version` | (none) | hardcoded | `claude.go:47` |

## Notes / open questions

- Beads can auto-install; dolt cannot. The asymmetry is intentional
  (dolt is a large binary distributed via GitHub releases, not
  go-installable), but it means `gt install` flows must handle a
  missing dolt differently from a missing bd.
- `BeadsTooOld` returns an error even when `autoInstall == true`
  (`beads.go:85-87`). The comment in this file doesn't say why;
  presumably because bumping bd across minor versions could invalidate
  existing beads databases and a silent upgrade would be dangerous.
  Worth confirming against beads release notes if this ever needs to
  change.
- `ParseVersion` silently collapses non-numeric parts to 0, so
  `"1.82.4-rc1"` compared via raw `ParseVersion` would equal `"1.82.4"`
  — but all three parse functions strip suffixes via regex before
  calling it, so this is only a risk if a new caller forgets the regex
  layer.
- Only three components are compared. A version bump to `1.82.4.1`
  would be treated as equal to `1.82.4`.
- Claude Code is the only dependency `deps` treats as **optional** and
  the only one with a **two-tier** version policy (minimum +
  recommended). The package itself just returns `ClaudeCodeNotFound`;
  the "absence is fine, Gas Town supports other agents" decision lives
  in [`internal/doctor`](doctor.md)'s `ClaudeBinaryCheck`, which maps
  `NotFound` to `StatusOK`. `CheckClaudeCode` has no installer
  counterpart to `EnsureBeads`.
