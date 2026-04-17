---
title: internal/constants
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/constants/constants.go
tags: [package, constants, platform-service, directories, timeouts, roles, emojis]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/constants

Single-file grab bag of shared constants: directory names that define the
Gas Town on-disk layout, file names for config/state, tmux session
prefixes, role/emoji tables, beads custom-type registrations, branch-name
helpers, rate-limit regex sources, and a legacy block of timing defaults
that predate `config.OperationalConfig`. Every part of the codebase that
needs "what is the mayor directory called" or "what does a polecat branch
name look like" reaches into this package instead of hard-coding the
string.

**Go package path:** `github.com/steveyegge/gastown/internal/constants`
**File count:** 1 go file, 1 test file.
**Imports (notable):** stdlib `time` only.
**Imported by (notable):** widely — directly by
[`internal/config`](config.md) (directory/file constants drive all config
path helpers), [`internal/session`](session.md) (tmux prefixes, role
names, timeouts), [`internal/workspace`](workspace.md) (via
`config.LoadTownConfig`), the witness/mayor/deacon command paths, and
effectively anything that constructs a town/rig path or mentions a role.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/constants/constants.go`.

### Timing constants (`constants.go:13-107`)

A block of `time.Duration` defaults for session management. The package
doc comment explicitly marks these **deprecated as single source of
truth**: `internal/config.OperationalConfig` now holds the live values
(with per-town override via `settings/config.json`), and the compiled-in
defaults in `config/operational.go` match these constants. New code is
supposed to read from `OperationalConfig`; the constants remain so old
call sites and tests keep compiling and for documentation.

Notable entries with comments worth reading in place:

- `ClaudeStartTimeout = 180 * time.Second` — documented rationale is the
  first-turn gt-prime patrol context round-trip that regularly exceeds
  60 s on Opus (`constants.go:17-22`).
- `StartupNudgeVerifyDelay = 25 * time.Second` and
  `StartupNudgeMaxRetries = 2` (`constants.go:73-85`) — both cite GH#3031
  as the bug that forced the current tuning.
- `MinHandoffCooldown = 2 * time.Minute` (`constants.go:87-92`) — tagged
  gt-058d; prevents tight restart loops where a patrol agent completes
  quickly on an idle rig and immediately re-hands-off.
- `GUPPViolationTimeout = 30 * time.Minute` (`constants.go:94-101`) —
  explicitly called out as the **single source of truth** for GUPP
  ("Gas Town Universal Propulsion Principle") — referenced by daemon
  lifecycle patrol, TUI feed stuck detection, and the web fetcher worker
  status. This one genuinely belongs here, not in config.

### Directory and file names (`constants.go:109-165`)

- Directory names: `DirMayor = "mayor"`, `DirPolecats = "polecats"`,
  `DirCrew = "crew"`, `DirRefinery = "refinery"`, `DirWitness =
  "witness"`, `DirRig = "rig"`, `DirBeads = ".beads"`, `DirRuntime =
  ".runtime"`, `DirSettings = "settings"`.
- File names: `FileRigsJSON`, `FileTownJSON`, `FileConfigJSON`,
  `FileAccountsJSON`, `FileHandoffMarker = "handoff_to_successor"`
  (written by `gt handoff` before respawn, cleared by `gt prime` after
  detection — the guard against the handoff-loop bug),
  `FileLastHandoffTS`, `FileQuotaJSON`.

Below the constants, a set of path-helper functions (`constants.go:349-
407`) construct the canonical paths from a town root or rig path:
`MayorRigsPath`, `MayorTownPath`, `MayorConfigPath`, `MayorAccountsPath`,
`MayorQuotaPath`, `TownRuntimePath`, `RigRuntimePath`, `RigMayorPath`,
`RigBeadsPath`, `RigPolecatsPath`, `RigCrewPath`, `RigSettingsPath`.
Each is a one-liner `return townRoot + "/" + DirMayor + "/" +
FileTownJSON`-style concatenation.

### Beads custom types (`constants.go:167-207`)

- `BeadsCustomTypes = "agent,role,rig,convoy,slot,queue,event,message,
  molecule,gate,merge-request"` — the comma-separated list Gas Town
  registers with beads at install time. The doc comment names each type
  and the Gas Town subsystem that uses it. Extracted from beads core in
  v0.46.0, hence the explicit registration.
- `BeadsCustomTypesList()` — slice form of the same.
- `BeadsCustomStatuses = "staged_ready,staged_warnings"` — the two
  convoy-staging statuses.
- `BeadsCustomStatusesList()` — slice form.

### Branch name prefixes (`constants.go:210-223`)

- `BranchMain = "main"`, `BranchBeadsSync = "beads-sync"`,
  `BranchPolecatPrefix = "polecat/"`,
  `BranchIntegrationPrefix = "integration/"`.

### Tmux session prefixes and role names (`constants.go:225-256`)

- `SessionPrefix = "gt-"` — for rig-level services (witness, refinery,
  crew, polecat, plus per-rig agents).
- `HQSessionPrefix = "hq-"` — for town-level services (`hq-mayor`,
  `hq-deacon`). Comment explicitly points at
  `session.MayorSessionName()` / `session.DeaconSessionName()` as the
  correct constructors rather than direct concatenation.
- `RoleMayor`, `RoleDeacon`, `RoleWitness`, `RoleRefinery`, `RolePolecat`,
  `RoleCrew` — the six canonical role name strings.

### Role emojis (`constants.go:258-341`)

Centralized emoji table — the Gas Town visual identity. `EmojiMayor` 🎩,
`EmojiDeacon` 🐺, `EmojiWitness` 🦉, `EmojiRefinery` 🏭, `EmojiCrew` 👷,
`EmojiPolecat` 😺. `RoleEmoji(role string) string` maps a role name to
its emoji, returning `"❓"` for unknown roles.

### Molecule formula names (`constants.go:281-321`)

String identifiers for patrol and dog workflows, used with
`bd mol wisp <name>` and for matching active patrol wisps by title
prefix: `MolDeaconPatrol`, `MolWitnessPatrol`, `MolRefineryPatrol`,
`MolDogReaper`, `MolDogJSONL`, `MolDogCompactor`, `MolDogCheckpoint`,
`MolDogDoctor`, `MolDogBackup`, `MolConvoyFeed`, `MolConvoyCleanup`.
`PatrolFormulas()` returns the three patrol formulas as a slice.

### Rate-limit pattern sources (`constants.go:412-435`)

Two exported `[]string` vars compiled elsewhere with `(?i)` for case-
insensitive matching against tmux pane content:

- `DefaultRateLimitPatterns` — hard 429/quota patterns (`"You've hit
  your .*limit"`, `"limit · resets ..."`, `"API Error: Rate limit
  reached"`, `"OAuth token revoked"`, `"OAuth token has expired"`,
  etc.). Explicitly narrowed to Claude's actual rate-limit text to avoid
  false positives from agent discussion.
- `DefaultNearLimitPatterns` — soft "approaching limit" patterns
  (`"\d{2,3}%\s*(of\s*)?(your\s*)?(daily\s*)?(usage|limit|quota)"`, etc.)
  used for proactive rotation before the hard 429 lands.

### Supported shells (`constants.go:343-345`)

`SupportedShells = []string{"bash", "zsh", "sh", "fish", "tcsh", "ksh",
"pwsh", "powershell"}` — the set of shell binaries Gas Town recognizes
when detecting whether a tmux pane is at a shell prompt.

## Related wiki pages

- [internal/config](config.md) — consumes directory and file constants
  when building path helpers and loading operational config.
- [internal/session](session.md) — uses the tmux prefixes, role names,
  and several timing constants.
- [internal/workspace](workspace.md) — depends on `DirMayor`,
  `FileTownJSON`, and friends to locate a town root.
- [gt](../binaries/gt.md) — indirect consumer via essentially every
  subcommand that constructs a path or mentions a role.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The timing block is labeled "deprecated as single source of truth" but
  `GUPPViolationTimeout` is explicitly reaffirmed as canonical in its
  own comment — the deprecation label therefore applies selectively.
  Grep callers before touching any of these: some commands may still be
  reading the constant directly rather than going through
  `config.OperationalConfig`.
- `BeadsCustomTypes` exists both as a comma-separated string and as a
  `[]string` slice via `BeadsCustomTypesList()`. Changes must be made in
  both places — there is no generator wiring them together. Same for
  statuses.
- Path helpers use `+` concatenation with hard-coded `"/"` separators
  rather than `filepath.Join`. This is fine on Linux/macOS, and tmux /
  runtime paths never land on Windows, but anything that calls these
  path helpers on Windows will produce forward-slash paths.
