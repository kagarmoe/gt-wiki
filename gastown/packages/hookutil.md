---
title: internal/hookutil
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/hookutil/roletype.go
tags: [package, hooks, role-classification, leaf]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/hookutil

Single source of truth for the autonomous/interactive role classification
used by all agent hook installer packages. Exposes exactly one function:
`IsAutonomousRole(role string) bool`. The classification decides whether a
role needs automatic mail injection on startup (autonomous, no human at
the keyboard) or whether it will be driven interactively by a human
operator.

**Go package path:** `github.com/steveyegge/gastown/internal/hookutil`
**File count:** 1 go file, 1 test file
**Imports (notable):**
[`internal/constants`](constants.md) for the role name constants
(`constants.RolePolecat`, etc.). No other imports. Deliberately minimal so
every hook installer package can depend on it without fan-in headaches.
**Imported by (notable):** every agent hook installer package that
distinguishes autonomous from interactive startup behavior — claude,
gemini, cursor, and the runtime fallback logic that decides "does this
session need a kickoff mail injection?" See [`internal/hooks`](hooks.md)
for the higher-level hook system this slots into.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/hookutil/roletype.go`.

### Public API

- `IsAutonomousRole(role string) bool` (`roletype.go:15-22`) — returns
  `true` for roles that operate without human prompting and therefore need
  automatic mail injection on startup. Returns `false` for everything
  else.

### Classification (hard-coded)

The switch at `roletype.go:17-21` is the entire classification:

**Autonomous (returns true):**
- `constants.RolePolecat`
- `constants.RoleWitness`
- `constants.RoleRefinery`
- `constants.RoleDeacon`
- `"boot"` — the string literal `"boot"`, not a constant. This is a
  minor inconsistency: every other entry goes through `constants.*`.

**Interactive (returns false):**
- `constants.RoleMayor`
- `constants.RoleCrew`
- Anything else, including unrecognized or future roles.

The default behavior for unknown roles is therefore "treat as
interactive" — a safe default, since an unknown role that *should* be
autonomous would fail to get its mail injection and surface as "why isn't
this agent doing anything?", whereas an unknown interactive role that
accidentally gets mail injection would paste surprising text into a
human's session.

## Docs claim

The package and function doc (`roletype.go:1`, `:7-14`) describe the
package as "shared utilities for agent hook installers" and state plainly
that `IsAutonomousRole` is "the single source of truth for the
autonomous/interactive classification used by all hook installer packages
(claude, gemini, cursor, etc.) and the runtime fallback logic."

## Drift

- **`"boot"` as a literal.** The switch case for `"boot"` at
  `roletype.go:17` uses a bare string, while every other arm uses a
  `constants.Role*` symbol. If a `constants.RoleBoot` exists (or is
  added later) without this file being updated, the two will diverge
  silently. Low risk but worth a lint pass.
- **"Single source of truth" is a social contract, not an enforced one.**
  Nothing stops another package from defining its own autonomous/
  interactive list inline. The claim is only true as long as reviewers
  notice. The [`internal/hooks`](hooks.md) page should be cross-checked
  to confirm no parallel list exists there.

## Notes / open questions

- No plural or bulk API — callers iterating over many roles will pay for
  the switch dispatch per call, but this is a nanosecond-scale concern
  for hook installers that run a handful of times at startup.
- The rationale comment lists five autonomous roles but the switch
  includes five cases (polecat, witness, refinery, deacon, boot) — count
  matches. The comment groups "mayor, crew (and anything else)" as
  interactive, which matches the `default` arm.

## Related wiki pages

- [gt](../binaries/gt.md) — top-level binary.
- [internal/hooks](hooks.md) — higher-level hook system that consumes
  this classification.
- [internal/constants](constants.md) — source of the `RolePolecat`,
  `RoleWitness`, `RoleRefinery`, `RoleDeacon` constants used in the
  switch.
- [go-packages inventory](../inventory/go-packages.md).
