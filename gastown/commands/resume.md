---
title: gt resume
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/resume.go
tags: [command, work, mail, handoff, session]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt resume

Check the inbox for handoff messages and display them formatted for
continuation. A very thin wrapper over `gt mail inbox` that filters
to messages with `HANDOFF` in the subject.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`resume.go:16`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/resume.go` (full
file, 144 lines).

### Invocation

```
gt resume
```

Takes no arguments, no flags. `runResume` immediately delegates to
`checkHandoffMessages` (`resume.go:33-35`).

### Behavior (`checkHandoffMessages`, `resume.go:38-138`)

1. **Shell to inbox.** Run `gt mail inbox --json` via `exec.Command`
   (`:40-41`). Capture stdout.
2. **Fallback** if `--json` isn't supported (`:42-60`): rerun
   `gt mail inbox` without `--json`, check the plain text output
   for the case-insensitive substring `HANDOFF` via the helper
   `containsHandoff` (`:141-144`), and print the whole output as-is
   if found.
3. **JSON path** (`:62-86`): unmarshal into an anonymous struct
   slice of `{id, subject, from, date, body}`. If unmarshal fails,
   fall back to the plain-text path again.
4. **Filter** (`:95-106`): keep only messages whose subject contains
   `HANDOFF`.
5. **Empty case** (`:108-113`): print the informational "No handoff
   messages in inbox" line and hint
   `gt handoff -s "..." to create one when handing off`.
6. **Render** (`:115-128`): print `--- Handoff N: <id> ---` headers
   with subject/from/date/body for each message.
7. **Trailing hints** (`:130-135`): tell the user how to read the
   full message (`gt mail read <id>`) and clear it afterward
   (`gt mail close <id>`).

### Subcommands

None.

### Flags

None.

## Notes / open questions

- **Not a session resumer.** Despite the name, this doesn't restore
  any hook state, reattach to a molecule, or resume a killed
  polecat. It is literally a filtered inbox reader. The semantic
  "resume your work after a break" action is split across:
  - `gt resume` — read the handoff mail.
  - [hook](hook.md) / `gt mol status` — see what's on your hook.
  - [sling](sling.md) — pick up whatever the handoff says to.
- **Double fallback.** The code tries `gt mail inbox --json`, then
  `gt mail inbox` (plain), then re-runs plain if JSON parsing fails.
  Three exec-command paths for the same nominal query is a code
  smell — likely a transitional layer while `mail inbox --json` is
  being rolled out.
- **Subject-only filter.** Filtering on `HANDOFF` in the subject
  means a handoff mail sent with a lowercased subject still hits
  (`containsHandoff` upper-cases first, `:141-143`) but a handoff
  in a body without the subject marker is missed.
- **Depends on `gt handoff`'s convention.** The text at `:111`
  tells the user to run `gt handoff -s "..."` to create one — the
  subject-line convention is soft-coded on the sender side. See
  [handoff](handoff.md) for how handoff mail is generated.
- **Related commands.** [handoff](handoff.md) (producer side),
  [mail](../commands/README.md) (inbox), [hook](hook.md)
  (post-resume action).
