---
title: gt unsling
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/unsling.go
  - /home/kimberly/repos/gastown/internal/beads/
tags: [command, work, sling, hook, recovery, stale-bead]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# gt unsling

Remove work from an agent's hook — the inverse of [sling](sling.md)
and [hook](hook.md). With no arguments, clears the caller's own
hook. With a bead ID, only unslings if that specific bead is
currently hooked. With a target, operates on another agent's hook.
Alias: `unhook`.

**Parent:** [gt](../binaries/gt.md) (root command), alias `unhook`
**Group:** `GroupWork` ("Work Management") (`unsling.go:20`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/unsling.go` (full
file, 385 lines).

### Invocation

```
gt unsling                               # clear my hook
gt unsling <bead-id>                     # only if this bead is hooked
gt unsling <target>                      # clear another agent's hook
gt unsling <bead-id> <target>            # both
```

`Args: cobra.MaximumNArgs(2)` (`unsling.go:40`).

### Arg parsing (`runUnslingWith`, `unsling.go:59-78`)

- 0 args: self, whatever is hooked.
- 1 arg: if `isAgentTarget(arg)` — contains `/` or is a known role
  name (`RoleMayor`, `RoleDeacon`, `RoleWitness`, `RoleRefinery`,
  `RoleCrew`) — treat as target; otherwise bead ID (`:368-385`).
- 2 args: `bead target`.

### Behavior (`runUnslingWith`, `unsling.go:59-259`)

1. **Resolve target agent.** `resolveTargetAgent(target)` or
   `resolveSelfTarget()` (`:81-93`).
2. **Find town root** (`:96-98`).
3. **Agent ID → bead ID.** `agentIDToBeadID(agentID, townRoot)`
   (`:102`). Error if it can't be converted.
4. **Pick beads database** via prefix-based routing (`:110-117`).
   `mayor`/`deacon` fall back to `townRoot` (hq-prefix beads);
   other rigs use `<townRoot>/<rigName>`. Then
   `beads.ResolveHookDir(townRoot, agentBeadID, fallbackPath)`.
5. **Find hooked bead by query, not by slot.** Per `hq-l6mm5`,
   the authoritative source is now the work bead itself
   (`:121-132`):
   ```go
   b.List(ListOptions{ Status: StatusHooked, Assignee: agentID,
                       Priority: -1 })
   ```
   — no need to read the agent bead's `hook_bead` slot.
6. **Town-level fallback** (`:138-160`) — per `gt-dtq7`, polecats
   and crew can hold `hq-*` beads that live in `townRoot/.beads`,
   not the rig DB. If the rig-scoped query returns empty and the
   agent isn't a town-level role, search town beads with both the
   exact agent ID and a normalized form (`mailNormalizedAgentID`:
   `rig/crew/name` → `rig/name`, `rig/polecats/name` → `rig/name`,
   `:360-366`).
7. **If still nothing:** call `cleanStaleHookedBeads` (`:166`,
   helper at `:267-355`) — finds stale beads with `status=hooked`
   assigned to the agent when the hook slot is already clear, and
   reopens them. Mirrors the dual-database search used above.
   Reports "Nothing on your hook" if nothing to clean.
8. **Specific-bead mismatch check** (`:178-180`): if a bead ID was
   passed and it doesn't match the currently hooked bead, error
   out.
9. **Resolve the hooked bead's database.** May be different from
   the agent bead's database (`:185-189`). Use
   `ResolveHookDir(townRoot, hookedBeadID, beadsPath)`.
10. **Get the bead to check completion** (`:190-198`). If the bead
    is missing and `--force` is set, proceed with a stub; otherwise
    error with a hint to use `--force`.
11. **Completion guard** (`:201-205`): if the bead isn't closed and
    `--force` isn't set, refuse with a clear error.
12. **Dry run** (`:213-216`) → print and return.
13. **Update the bead** (`:224-236`): set status `open`, clear
    assignee. Non-fatal on update error — print a warning and
    continue so the agent gets unblocked.
14. **Log `unhook` event** via `events.LogFeed(TypeUnhook, agentID,
    events.UnhookPayload(hookedBeadID))` (`:239`).
15. **Propulsion signal for mayor** (`:243-253`): if the target was
    `"mayor/"`, enqueue a nudge so the ACP propeller reacts
    event-driven to the hook change.
16. **Report success** (`:255-256`).

### Flags

| flag | default | notes |
|---|---|---|
| `-n`, `--dry-run` | `false` | Show what would happen. |
| `-f`, `--force` | `false` | Unsling even if incomplete or bead missing. |

### Subcommands

None.

### `cleanStaleHookedBeads` (`:267-355`)

Stale-hook cleanup is invoked when the hook query returns empty:
the function looks for beads with `status=hooked` still assigned to
the agent in both the rig DB and (for non-town-level roles) the
town DB. If a specific bead was requested, filters to just that
one. For each stale bead, it runs `b.Update(... Status: open,
Assignee: "")`.

### `mailNormalizedAgentID` (`:360-366`)

Normalizes `rig/crew/name` and `rig/polecats/name` to `rig/name` —
the canonical form used by mail bead assignees (via
`mail.AddressToIdentity` in `sendHandoffMail`).

## Failure modes

### Silent suppression (what errors are swallowed?)

- **Bead status update failure non-fatal:** `unsling.go:227-235` — if `hookedB.Update` fails to set the bead status back to "open", a warning is printed to stderr but the unsling continues. The agent's hook is cleared but the bead remains in "hooked" status with the old assignee. **Present** — warning emitted but bead status is inconsistent.
- **Unhook event log silently discarded:** `unsling.go:239` — `_ = events.LogFeed(...)` discards the error. **Absent** — activity feed gap.
- **Mayor nudge silently discarded:** `unsling.go:247` — same `_ = nudge.Enqueue(...)` pattern as sling/hook. **Absent**.

## Notes / open questions

- **Two bugs baked into the helper.** The `hq-l6mm5` and
  `gt-dtq7` comments point at prior state drift between the hook
  slot and the bead's own status. The current code now treats the
  bead as authoritative, with dual-database fallback.
- **Dry run is partial.** `--dry-run` only prints before the
  update path; the stale-bead cleanup path (`:325-329`) has its
  own dry-run handling. The event log and propulsion nudge are
  skipped on dry-run because of the early return (`:213-216`).
- **Stale cleanup is opportunistic.** If `cleanStaleHookedBeads`
  fixes stale beads, it prints per-bead but does NOT record an
  `unhook` event, on the theory that those beads weren't really
  hooked in the first place.
- **Related commands.**
  - [sling](sling.md) / [hook](hook.md) — the producers.
  - [done](done.md) — the happy path out of a hook.
  - [release](release.md) — heavier recovery for stuck
    in-progress beads.
  - [handoff](handoff.md) — fresh-context handoff alternative
    that also leaves the bead re-hookable.
