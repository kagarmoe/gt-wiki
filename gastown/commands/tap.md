---
title: gt tap
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/tap.go
  - /home/kimberly/repos/gastown/internal/cmd/tap_guard.go
  - /home/kimberly/repos/gastown/internal/cmd/tap_list.go
  - /home/kimberly/repos/gastown/internal/cmd/tap_polecat_stop.go
tags: [command, ungrouped, beads-exempt, hook, claude-code, policy]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift, wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt tap

Parent command group for Claude Code hook handlers that tap into tool
execution. Three subcommands are wired: `guard` (policy enforcement),
`list` (handler enumeration), and `polecat-stop-check` (idle-polecat
safety net). Called by Claude's PreToolUse/PostToolUse hooks.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** no
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`; tap is
excluded from command-usage logging so its own invocation from inside
a PreToolUse hook doesn't recurse into `bd` writes)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/tap.go:7-36`.

### Invocation

```
gt tap guard <policy-name>           # PreToolUse, exits 2 to block
gt tap list [--guards]               # enumerate registered handlers
gt tap polecat-stop-check            # Stop hook safety net for polecats
```

`Use: "tap"` (`tap.go:8`). The parent command has no `RunE` — it is a
pure subcommand group. Subcommands are registered from sibling files
via per-file `init()` blocks.

### Subcommand concept (from `Long` help, `tap.go:10-29`)

> Hook handlers for Claude Code PreToolUse and PostToolUse events.
>
> These commands are called by Claude Code hooks to implement policies,
> auditing, and input transformation. They tap into the tool execution
> flow to guard, audit, inject, or check.
>
> Subcommands:
>   guard   - Block forbidden operations (PreToolUse, exit 2)
>   audit   - Log/record tool executions (PostToolUse) [planned]
>   inject  - Modify tool inputs (PreToolUse, updatedInput) [planned]
>   check   - Validate after execution (PostToolUse) [planned]

### Example Claude Code settings integration

From the `Long` help (`tap.go:22-28`):

```json
{
  "PreToolUse": [{
    "matcher": "Bash(gh pr create*)",
    "hooks": [{"command": "gt tap guard pr-workflow"}]
  }]
}
```

### Flags

None on the parent. Subcommand flags are declared in their respective
files (not in this batch).

### Why beads-exempt

Per the batch brief: `tap` is "excluded from command-usage logging per
`persistentPreRun`". The beads-exempt map on `root.go:44-77` is what
root-command dispatch checks before running per-command telemetry and
`bd` write paths; `tap` is in it. Semantically: a PreToolUse hook
should be fast and side-effect-free, and should not itself enqueue
more Claude-visible tool-use events or stall on `bd` I/O.

### Subcommands

Registered from sibling files via `init()`:

- **`gt tap guard <policy>`** — blocks forbidden operations. Exits 2
  (Claude Code's convention for a PreToolUse veto that should surface
  the guard's stderr to the user). Registered at `tap_guard.go:65`.
  Has its own sub-subcommands: `pr-workflow` (`tap_guard.go:66`),
  `bd-init` (`tap_guard_bd_init.go:29`), `dangerous`
  (`tap_guard_dangerous.go:42`), `mol-patrol`
  (`tap_guard_mol_patrol.go:32`).
- **`gt tap list`** — lists all tap handlers (guards, audits, injectors,
  checks) from registry and built-in commands. Registered at
  `tap_list.go:31`.
- **`gt tap polecat-stop-check`** — safety net for the "idle polecat"
  problem: runs `gt done` on session Stop if the polecat has pending
  work. Registered at `tap_polecat_stop.go:38`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/tap.go:10-29` — Cobra Long text

### Verbatim
> Hook handlers for Claude Code PreToolUse and PostToolUse events.
>
> These commands are called by Claude Code hooks to implement policies,
> auditing, and input transformation. They tap into the tool execution
> flow to guard, audit, inject, or check.
>
> Subcommands:
>   guard   - Block forbidden operations (PreToolUse, exit 2)
>   audit   - Log/record tool executions (PostToolUse) [planned]
>   inject  - Modify tool inputs (PreToolUse, updatedInput) [planned]
>   check   - Validate after execution (PostToolUse) [planned]
>
> Hook configuration in .claude/settings.json:
>   {
>     "PreToolUse": [{
>       "matcher": "Bash(gh pr create*)",
>       "hooks": [{"command": "gt tap guard pr-workflow"}]
>     }]
>   }
>
> See ~/gt/docs/HOOKS.md for full documentation.

## Drift

### Long text omits two implemented subcommands (`list`, `polecat-stop-check`)
- **Claim source:** Cobra Long text at `tap.go:15-18`
- **Docs claim:** Subcommands list names only `guard`, `audit [planned]`, `inject [planned]`, `check [planned]`
- **Code does:** Three subcommands are actually wired: `guard` (`tap_guard.go:65`), `list` (`tap_list.go:31`), `polecat-stop-check` (`tap_polecat_stop.go:38`). The `list` and `polecat-stop-check` subcommands are completely absent from the Long text.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — update the Long text subcommand list to include `list` and `polecat-stop-check`, and mark `audit`/`inject`/`check` as `[planned]` or remove them
- **Release position:** `in-release` — all three sibling files exist at `v1.0.0`

### Long text advertises three unimplemented subcommands
- **Claim source:** Cobra Long text at `tap.go:16-18`
- **Docs claim:** `audit`, `inject`, `check` listed as `[planned]` subcommands
- **Code does:** No `tap_audit.go`, `tap_inject.go`, or `tap_check.go` exist anywhere in the tree. Zero code support.
- **Category:** `cobra drift` (the `[planned]` tag is honest about the status, but the enumeration alongside the implemented `guard` is misleading — readers see 4 subcommands and assume 3 will work)
- **Severity:** `wrong`
- **Fix tier:** `code` — either remove the planned entries from Long text or add explicit "NOT YET IMPLEMENTED" callouts
- **Release position:** `in-release`

→ [gastown/drift/README.md](../drift/README.md)

## Related commands

- [hooks](hooks.md) — the Claude Code hooks *manager* (plural). Where
  hooks get installed into `.claude/settings.json` — i.e. the command
  that configures tap handlers to actually fire. Distinct namespace
  from `tap` despite the name overlap.
- [signal](signal.md), [metrics](metrics.md) — sibling low-level tmux /
  observability primitives per the batch brief's cross-link hint.
- [guard](guard.md) — if a top-level `gt guard` exists, it would be a
  separate surface — but per the inventory this is subcommand-only
  (`gt tap guard ...`).
- [audit](audit.md) — a *different* audit command at the top level
  (reads past events); not the same as the planned `gt tap audit`
  (which would write events on PostToolUse).
- [prime](prime.md) — session-start hook; sibling Claude-integration
  command.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Failure modes

No failure modes discovered. Guard system for controlling which commands agents can run. Read-only policy evaluation that returns allow/deny. Guard checks are pure functions; no state mutations.

## Notes / open questions

- **Parent-only in this file.** `tap.go` defines only the parent
  cobra.Command; subcommands are registered from 7 sibling files:
  `tap_guard.go`, `tap_guard_bd_init.go`, `tap_guard_dangerous.go`,
  `tap_guard_mol_patrol.go`, `tap_list.go`, `tap_polecat_stop.go`,
  plus `tap_guard_dangerous_test.go`. → Phase 2 wiki-stale fix:
  Phase 2 said "only `guard` is actually wired" but missed `list` and
  `polecat-stop-check` by not running the sibling-file audit.
  **Phase 2 root cause: `phase-2-incomplete` (heuristic)** — Phase 2
  took the parent file in isolation without checking sibling `init()`
  registrations.
- **Three subcommands are planned and never landed.** → promoted to
  `## Drift` (Long text advertises unimplemented audit/inject/check).
  Anyone writing hooks today has `guard`, `list`, and
  `polecat-stop-check`.
- **Beads-exempt matters for correctness, not just speed.** A
  PreToolUse hook that wrote to `bd` while the user was mid-tool-call
  could cause reentrancy — the tool the user is running might itself
  emit `bd` writes that the tap hook would try to audit. The exemption
  is the cheap fix.
- **Where does `gt tap guard`'s exit code go?** Claude Code treats
  exit 2 as a veto from PreToolUse and surfaces the hook's stderr to
  the model. The `guard` subcommand (in a sibling file) is what
  actually implements the policy lookup and exit-2 path.
- **`~/gt/docs/HOOKS.md`** is referenced on `tap.go:30` — worth
  ingesting as a doc source when the wiki expands to cover docs.
