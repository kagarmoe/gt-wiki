---
title: gt hook
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/hook.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, polecat-safe, hook, durability, session]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt hook

Show or attach work on the current agent's (or another agent's)
hook — the "durability primitive" for work that survives session
restarts, context compaction, and handoffs.

**Also known as:** `gt work` (alias, `hook.go:24`).
**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`hook.go:25`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`hook.go:26`)
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

**Note:** `hook` (singular, this command) is distinct from `hooks`
(plural, Config group) at [hooks.md](hooks.md). `hooks` manages
Claude Code's SessionStart / PreCompact / etc. hook system;
`hook` is the agent work-durability primitive.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/hook.go` (653
lines).

### Invocation

```
gt hook                                 # show status (alias for 'gt mol status')
gt hook <bead-id>                       # attach bead to your hook
gt hook <bead-id> <target-agent>        # attach bead to another agent's hook
gt hook status [target]                 # show hook status
gt hook show [agent]                    # show hook in compact one-line format
gt hook attach <bead-id> [target]       # explicit attach
gt hook detach <bead-id> [target]       # detach a specific bead
gt hook clear [bead-id] [target]        # clear hook (alias for 'gt unhook')
```

### The "hook" concept

From the Long help at `hook.go:28-49`:

> The hook is the "durability primitive" — work on your hook
> survives session restarts, context compaction, and handoffs. When
> you restart (via gt handoff), your SessionStart hook finds the
> attached work and you continue from where you left off.

Related commands listed in the help:

- `gt sling <bead>` — Hook + start now (keep context)
- `gt handoff <bead>` — Hook + restart (fresh context)
- `gt unsling` — Remove work from hook

### Top-level `runHookOrStatus` dispatch (`hook.go:191-203`)

With `Args: cobra.MaximumNArgs(2)`, the parent command has four
dispatch paths:

1. `--clear` flag set → delegate to `runUnslingWith(cmd, args,
   hookDryRun, hookForce)`.
2. No args → `runMoleculeStatus(cmd, args)` — the same function
   `gt mol status` uses. This makes bare `gt hook` a status
   display.
3. One or two args → `runHook(cmd, args)` — attach work.

### Subcommands

Five, registered in `init()` at `hook.go:182-186`:

1. **`gt hook status [target]`** (`hookStatusCmd`, `hook.go:55-68`)

   Alias for `gt mol status`. Delegates to `runMoleculeStatus`.
   Optional positional is the agent whose hook to inspect.

2. **`gt hook show [agent]`** (`hookShowCmd`, `hook.go:71-94`)

   One-line compact format for any agent's hook. Uses cases
   documented in the Long text include mayor checking polecat work,
   witness debugging coordination, and quick status overviews.
   Output format (`hook.go:90-91`):

   ```
   gastown/polecats/nux: gt-abc123 'Fix the widget bug' [in_progress]
   ```

3. **`gt hook attach <bead-id> [target]`** (`hookAttachCmd`,
   `hook.go:97-112`)

   Explicit form of `gt hook <bead-id>`. Delegates to `runHook`.

4. **`gt hook detach <bead-id> [target]`** (`hookDetachCmd`,
   `hook.go:115-127`)

   Remove a specific bead from a hook. Delegates to `runUnslingWith`
   — so this is an alias for `gt unsling` with a specific bead.

5. **`gt hook clear [bead-id] [target]`** (`hookClearCmd`,
   `hook.go:130-149`)

   Documented as "alias for `gt unhook`" (`hook.go:132-133`).
   Delegates to `runHookClear` → `runUnslingWith`.

### Core `runHook` attachment (`hook.go:210-250+`)

1. **Parse args** — `beadID` and optional `targetAgent`
   (`hook.go:211-217`).
2. **Block polecats from hooking** (`hook.go:219-231`): polecats
   cannot hook work because their session lifecycle is driven by
   [done](done.md). Returns an error: `"polecats cannot hook work
   (use gt done for handoff)"`. As in [handoff.go](handoff.md), the
   check uses `GT_ROLE` first via `parseRoleString` then falls back
   to `GT_POLECAT`.
3. **Verify bead exists** via `verifyBeadExists(beadID)`
   (`hook.go:233-236`).
4. **Resolve agent identity** — `resolveTargetAgent(targetAgent)`
   when a target was given, else `resolveSelfTarget()`
   (`hook.go:239-250`).

The rest of the file continues the attach flow (creating the
durability record, notifying target, etc.) — not chased here.

### Flags

Per the `init()` at `hook.go:159-186`:

**Parent `hook` flags:**

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--subject` | `-s` | string | `""` | Subject for handoff mail (optional) |
| `--message` | `-m` | string | `""` | Message for handoff mail (optional) |
| `--dry-run` | `-n` | bool | `false` | Show what would be done |
| `--force` | `-f` | bool | `false` | Replace existing incomplete hooked bead |
| `--clear` | — | bool | `false` | Clear your hook (alias for 'gt unhook') |
| `--json` | — | bool | `false` | Output as JSON (for status; shares `moleculeJSON` with the molecule family) |

**`status` / `show` subcommands:** share `--json` binding to the
same `moleculeJSON` global (`hook.go:168-170`).

**`attach` subcommand:** `-f, --force`.

**`detach` subcommand:** `-f, --force`.

**`clear` subcommand:** `-n, --dry-run`, `-f, --force`.

### Related commands

- [handoff](handoff.md) — `gt handoff <bead>` attaches to hook, then
  restarts. The hook is what makes handoff durable: after the fresh
  session starts, the SessionStart hook reads the attached work and
  the agent resumes.
- [done](done.md) — polecats use `gt done` instead of hooking. The
  hook concept is for roles that persist across tasks.
- [prime](prime.md) — the SessionStart hook runs `gt prime --hook`
  to resume attached work.
- [assign](assign.md) — cross-agent form of `gt hook <bead>
  <target>`, plus bead creation.
- [hooks](hooks.md) — the plural command, which manages Claude Code
  hook scripts — unrelated to this command's concept but easily
  confused by name.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **`moleculeJSON` is shared.** The `--json` flag on `hook`,
  `status`, and `show` all bind to the same global
  (`hook.go:168-170`), which is defined in the `molecule` command
  family. This is a subtle coupling — changing `moleculeJSON`'s
  default elsewhere would silently affect hook.
- **"gt mol status" equivalence** — the dispatch at
  `hook.go:192-200` makes `gt hook`, `gt hook status`, and `gt mol
  status` all equivalent for the no-args show case. Worth flagging
  that `hook` is a thin alias layer on top of the molecule family.
- **Five subcommands, one parent-as-shortcut.** `gt hook <bead>` is
  equivalent to `gt hook attach <bead>`, and `gt hook` (no args) is
  equivalent to `gt hook status`. This is idiomatic for
  frequently-used primitives but obscures the command tree.
- **`runUnslingWith` is the shared detach/clear backend.** Both
  detach and clear funnel into it, with different default args and
  arity. The unsling semantics live outside `hook.go`.
