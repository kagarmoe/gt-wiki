---
title: gt handoff
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/handoff.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, polecat-safe, session, handoff, durability]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# gt handoff

End the current agent session and hand off to a fresh one — the
canonical way to restart any non-polecat agent. Polecats are
transparently redirected to [done](done.md).

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`handoff.go:32`)
**Polecat-safe:** yes (`AnnotationPolecatSafe: "true"` on
`handoff.go:33`)
**Beads-exempt:** yes (in `beadsExemptCommands` on
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/handoff.go:30-96`
for the cobra surface; the full `runHandoff` is 1720 lines covering
session detection, role-based dispatch, git cleanliness checks, state
collection, mail composition, tmux pane management, and respawn.

### Invocation

```
gt handoff [bead-or-role] [flags]
```

From the Long help (`handoff.go:35-65`):

- **No args** — hand off the current session.
- **Bead ID** (`gt-xxx`, `hq-xxx`) — hook that work first, then
  restart.
- **Role name** — hand off that role's session (and switch to it).

### Role-based behavior

The top-level role matrix from the Long text (`handoff.go:37-41`):

> - Mayor, Crew, Witness, Refinery, Deacon: Respawns with fresh
>   Claude instance
> - Polecats: Calls `gt done --status DEFERRED` (Witness handles
>   lifecycle)

The polecat-detection-and-redirect is implemented at
`handoff.go:131-161`. Role is determined via `GT_ROLE` first (parsed
through `parseRoleString`), with `GT_POLECAT` as a fallback. The
code comments warn about coordinators having "stale `GT_POLECAT` in
their environment from spawning polecats" — a gotcha that motivates
the `GT_ROLE`-first check. If the parsed role is
`RolePolecat`, `gt handoff` forks `gt done --status DEFERRED` and
returns its exit code.

### `--auto` vs `--cycle` vs normal handoff

Three modes, documented in the `Long` (`handoff.go:58-63`) and
dispatched early in `runHandoff`:

1. **`--auto`** (`handoff.go:116-118`): save state only, no session
   cycling. Used by PreCompact hooks to preserve state before
   compaction. Exits before the git-status warning check — auto
   mode is triggered by hooks and shouldn't spam warnings.
   `--no-git-check` has no effect in this mode.
2. **`--cycle`** (`handoff.go:127-129`): full session cycling. Runs
   `runHandoffCycle()`. Skips the polecat→`gt done` redirect
   because cycling preserves work state (the hook stays attached).
3. **Normal** — the 1700-line default path. Collects state
   optionally (`--collect`), composes handoff mail, enforces a
   cooldown, and respawns the current tmux pane.

### Normal handoff flow (first ~200 lines)

Observed at `handoff.go:98-200`:

1. **`--stdin`** — read message body from stdin to avoid shell
   quoting issues (`handoff.go:100-109`). Mutually exclusive with
   `-m/--message`.
2. **`--auto` / `--cycle` early-return** as described above.
3. **Polecat detect + redirect** (`handoff.go:131-161`).
4. **Interactive confirmation** via `promptYesNo` unless `--yes/-y`
   or `--dry-run` or stdin is not a TTY (`handoff.go:166-171`).
5. **Cooldown enforcement** via `enforceHandoffCooldown()`
   (`handoff.go:173-176`). Comment references bead `gt-058d`: a
   patrol agent that completes quickly on idle rigs can hand off
   immediately and the daemon respawns, "creating a crash loop."
6. **`--collect`** (`handoff.go:179-189`): auto-gather state
   (status, inbox, ready beads, in-progress items) into the
   handoff message via `collectHandoffState`.
7. **Tmux socket setup** (`handoff.go:194-199`): uses
   `tmux.SocketFromEnv()` to get the caller's tmux server socket,
   because "the calling process may be on a different tmux server
   than the town socket" — critical for self-handoff because pane
   operations (`clear-history`, `respawn-pane`) must target the
   caller's own server.

The remaining ~1500 lines cover git cleanliness checks (gated by
`--no-git-check`), hook attachment, mail composition, bd updates,
tmux pane ops, and the SessionStart-hook handoff that runs
`gt prime` on the successor.

### `--collect` detail

Per the Long text (`handoff.go:55-57`): "gathers current state
(hooked work, inbox, ready beads, in-progress items) and includes
it in the handoff mail. This provides context for the next session
without manual summarization."

### Subcommands

None. `grep handoffCmd.AddCommand ... handoff.go` returns no hits.

### Flags

Defined in `init()` at `handoff.go:83-96`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--watch` | `-w` | bool | `true` | Switch to new session (for remote handoff) |
| `--dry-run` | `-n` | bool | `false` | Show what would be done without executing |
| `--subject` | `-s` | string | `""` | Subject for handoff mail (optional) |
| `--message` | `-m` | string | `""` | Message body for handoff mail (optional) |
| `--collect` | `-c` | bool | `false` | Auto-collect state (status, inbox, beads) into handoff message |
| `--stdin` | — | bool | `false` | Read message body from stdin (avoids shell quoting issues) |
| `--auto` | — | bool | `false` | Save state only, no session cycling (for PreCompact hooks) |
| `--cycle` | — | bool | `false` | Auto-cycle session (for PreCompact hooks that want full session replacement) |
| `--reason` | — | string | `""` | Reason for handoff (e.g., 'compaction', 'idle') |
| `--no-git-check` | — | bool | `false` | Skip git workspace cleanliness check |
| `--yes` | `-y` | bool | `false` | Skip confirmation prompt (for automation and scripting) |

### The `getCurrentTmuxSession` export

Per the batch notes: `feed.go` references `getCurrentTmuxSession`
from `handoff.go`. This helper is one of the tmux-session primitives
the handoff machinery needs, and is shared across session-aware
commands.

### Related commands

- [done](done.md) — polecats don't use `gt handoff`; they're
  redirected to `gt done --status DEFERRED`. This is the single
  most important asymmetry in the Work group: polecats have a
  one-way exit, other roles cycle in place.
- [hook](hook.md) — the `SessionStart` hook invoked on the fresh
  successor runs `gt prime --hook` to resume hooked work. `gt
  handoff <bead-id>` first attaches the bead to the hook, then
  restarts, so the fresh session picks it up.
- [prime](prime.md) — run on session start by the SessionStart hook
  to restore context.
- [mail](../commands/) (not yet mapped) — used to compose the
  handoff mail.
- [../binaries/gt.md](../binaries/gt.md) — root.

## Notes / open questions

- **Beads-exempt** — handoff works without `bd` installed. Rationale
  is probably that handoff is a durability primitive: the system
  must be able to hand off even when beads itself is broken.
- **The polecat redirect is transparent.** A polecat running
  `gt handoff` gets `🐾 Polecat detected (<name>) - using gt done
  for handoff` printed and then the redirect happens. There's no
  way for a polecat to bypass this — the `Use` doesn't mention it,
  and there's no `--force-handoff` or similar escape hatch.
- **Tmux cooldown** (`gt-058d`) — the cooldown enforcement is a
  stabilization measure for patrol agents. Worth capturing as a
  concept.
- **Normal/auto/cycle split** is the core distinction and the three
  modes diverge early. Some callers (PreCompact hooks) only ever use
  `--auto`; interactive humans never do.
