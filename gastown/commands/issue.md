---
title: gt issue
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/issue.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, tmux, status-line, session-env]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt issue

Sets, clears, or shows the active issue ID stored in the current
tmux session's environment so the tmux status line can display what
the user is working on. This is **not** a bead/issue-tracker wrapper —
it only manipulates the `GT_ISSUE` tmux session variable.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/issue.go:12-20`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`).
Despite the name, this command does not touch beads at all; the stale
binary check will still run.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/issue.go:12-151`.

### Invocation

```
gt issue set <issue-id>
gt issue clear
gt issue show
```

Parent `issueCmd` has no `RunE` set (`issue.go:12-20`) — running
`gt issue` with no subcommand falls through to cobra's default help.

### Behavior

All three subcommands share the same tmux-session-discovery shape:
first try `TMUX_PANE` env var (`issue.go:63-70`, `:82-88`, `:101-107`),
then fall back to `detectCurrentSession()` (`issue.go:124-151`) which
builds a session name from `GT_ROLE`, `GT_RIG`, `GT_POLECAT`,
`GT_CREW`. If neither resolves, returns `"not in a tmux session"`.

All three call into `internal/tmux`:

- `runIssueSet` (`issue.go:59-79`) — `t.SetEnvironment(session,
  "GT_ISSUE", issueID)` then prints `Issue set to: <id>`.
- `runIssueClear` (`issue.go:81-98`) — same, with empty string as
  value. Comment at `:91` notes: "Set to empty string to clear".
- `runIssueShow` (`issue.go:100-121`) — `t.GetEnvironment(session,
  "GT_ISSUE")`; prints `No issue set` or `Current issue: <id>`.

### `detectCurrentSession` details

`issue.go:124-151`. Logic cascade:

1. **Rig-scoped path** (`issue.go:132-142`): if `GT_RIG` is set —
   - If `GT_POLECAT` is also set, **and** `GT_ROLE` is empty or
     parses to `RolePolecat` (comment `:131` warns coordinators may
     have stale `GT_POLECAT`), use `session.PolecatSessionName`.
   - Else if `GT_CREW` is set, use `session.CrewSessionName`.
2. **Mayor path** (`issue.go:144-148`): if `GT_ROLE` parses to
   `RoleMayor`, use `getMayorSessionName()`.
3. Otherwise return empty string (caller treats as "not in a tmux
   session").

### Subcommands

- `set <issue-id>` — write `GT_ISSUE` to session env
  (`issue.go:22-31`).
- `clear` — set `GT_ISSUE=""` (`issue.go:33-40`).
- `show` — read and print `GT_ISSUE` (`issue.go:42-50`).

### Flags

None on any subcommand.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [prime.md](prime.md) — `prime` also reads the `GT_ROLE`/`GT_RIG`
  environment convention that `detectCurrentSession` relies on.
- [whoami.md](whoami.md) — peer helper that also parses `GT_ROLE` etc.
  to identify the current agent.
- **`issue` vs `bd`**: this command has no relationship to the beads
  issue tracker despite sharing the word. Gas Town uses `bd` directly
  for issue management; `gt issue` is a tmux-display cosmetic layer.
  The `bead` subcommand (if present elsewhere in `internal/cmd/`) is
  the beads-related one.

## Notes / open questions

- `GT_ISSUE` is not listed in any of the env-var documentation pages
  in the Layer-(c) file layer yet; should be captured in an
  `env-vars.md` rollup page alongside `GT_ROLE`, `GT_RIG`,
  `GT_POLECAT`, `GT_CREW`, `GT_TOWN_ROOT`, and others.
- The `TMUX_PANE` → `detectCurrentSession` cascade is used in other
  commands too (e.g., see `detectCurrentSession()` reference from
  `theme.go:237`). Worth a `packages/session.md` page enumerating the
  canonical session-name builders.
- **Misleading grouping.** `gt issue` is in the `GroupConfig` group
  even though it is purely a tmux status-line cosmetic. It is neither
  a configuration mutation nor a persistent setting — it lives only
  in the tmux session's env and evaporates when the session ends.
  Listed here because that is where `GroupID: GroupConfig`
  (`issue.go:14`) places it.
