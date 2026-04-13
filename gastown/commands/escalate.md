---
title: gt escalate
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/escalate.go
  - /home/kimberly/repos/gastown/internal/cmd/escalate_impl.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, escalation, severity, routing, mayor]
---

# gt escalate

Raise a severity-routed escalation — the canonical way for an agent
to get human or mayor attention on a blocking issue.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`escalate.go:24`)
**Polecat-safe:** no (no `AnnotationPolecatSafe` on `escalateCmd`)
**Beads-exempt:** no (not in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77`)
**Branch-check-exempt:** no

## What it actually does

Source (cobra surface):
`/home/kimberly/repos/gastown/internal/cmd/escalate.go:1-173`. The
runtime implementations (`runEscalate`, `runEscalateList`,
`runEscalateAck`, `runEscalateClose`, `runEscalateStale`,
`runEscalateShow`) live in `escalate_impl.go` — not read in this
batch.

### Invocation

```
gt escalate [description] [flags]            # create an escalation
gt escalate list [--all] [--json]             # list open escalations
gt escalate ack <escalation-id>                # acknowledge
gt escalate close <escalation-id> --reason …   # close
gt escalate stale [--dry-run] [--json]         # re-escalate stale
gt escalate show <escalation-id> [--json]      # detail view
```

The top-level form creates a new escalation. Subcommands cover the
lifecycle: list → ack → close, with `stale` as an out-of-band
re-escalation pass.

### Severity model

Documented in the Long help at `escalate.go:32-36`:

| level    | priority | meaning                              |
|----------|----------|--------------------------------------|
| critical | P0       | Immediate attention required         |
| high     | P1       | Urgent, needs attention soon         |
| medium   | P2       | Standard escalation (default)        |
| low      | P3       | Informational, can wait              |

Default is `medium` (see `--severity` flag default at
`escalate.go:141`).

### Workflow

From the Long help (`escalate.go:38-43`):

1. Agent encounters a blocking issue.
2. Runs `gt escalate "Description" --severity high --reason "..."`.
3. The escalation is routed based on `settings/escalation.json`.
4. Recipient acknowledges with `gt escalate ack <id>`.
5. After resolution: `gt escalate close <id> --reason "fixed"`.

### Routing config

Read by `escalate_impl.go` (not detailed here) from
`~/gt/settings/escalation.json`. Per `escalate.go:45-50`:

- `routes` — map severity → action list (`bead`, `mail:mayor`,
  `email:human`, `sms:human`).
- `contacts` — human email / SMS for external notifications.
- `stale_threshold` — when unacked escalations get bumped
  (default 4h).
- `max_reescalations` — cap on how many times severity can be
  bumped (default 2).

### Escalations-as-beads

`escalate.go:30-31` — "Escalations are tracked as beads with
`gt:escalation` label". The escalation isn't a separate data type;
it's a bead with a specific label and the severity stored as bead
metadata. `ack` adds an `acked` label (`escalate.go:82`).

### Stale re-escalation

`escalateStaleCmd` (`escalate.go:105-125`):

1. Find escalations older than the stale threshold (default 4h).
2. Bump severity: low → medium → high → critical.
3. Re-route according to the new severity level.
4. Send mail to the new routing targets.
5. Respect `max_reescalations` (default 2) to prevent infinite
   bumping.

The threshold lives in `settings/escalation.json`.

### Subcommands

Registered at `escalate.go:164-169`:

| subcommand | help short | source |
|------------|------------|--------|
| `list`  | List open escalations                      | `escalate.go:62-75` |
| `ack`   | Acknowledge an escalation                  | `escalate.go:77-89` |
| `close` | Close a resolved escalation                | `escalate.go:91-103` |
| `stale` | Re-escalate stale unacknowledged escalations | `escalate.go:105-125` |
| `show`  | Show details of an escalation              | `escalate.go:127-137` |

### Flags

Top-level `escalateCmd` (`escalate.go:141-147`):

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--severity` | `-s` | string | `medium` | Severity level: critical, high, medium, low |
| `--reason` | `-r` | string | `""` | Detailed reason for escalation |
| `--source` | — | string | `""` | Source identifier (e.g. `plugin:rebuild-gt`, `patrol:deacon`) |
| `--related` | — | string | `""` | Related bead ID (task, bug, etc.) |
| `--json` | — | bool | `false` | Output as JSON |
| `--dry-run` | `-n` | bool | `false` | Show what would be done without executing |
| `--stdin` | — | bool | `false` | Read reason from stdin (avoids shell quoting issues) |

Subcommand flags (`escalate.go:149-163`):

- `list`: `--json`, `--all`.
- `close`: `--reason` (marked required at `escalate.go:155`).
- `stale`: `--json`, `--dry-run` (shared with top-level flag variable).
- `show`: `--json`.
- `ack`: no flags.

### Related commands

- [mail](mail.md) — the transport layer for `mail:mayor` routes.
  Escalation send-to-mayor actually writes a mail bead.
- [nudge](nudge.md) — used implicitly when mail is auto-notified
  to the mayor session.
- [mayor](mayor.md) — primary recipient; the mayor's job
  description is described in this repo's project CLAUDE.md as
  "receives all escalations".
- [patrol](patrol.md) — patrol agents can invoke `gt escalate`
  with `--source "patrol:<name>"` when they detect issues.
- The project CLAUDE.md's Dolt operational playbook shows
  `gt escalate -s HIGH "Dolt: ..."` as the canonical escalation
  path for dolt trouble (escalations are stored as beads with the
  `gt:escalation` label, regardless of the subject).

## Notes / open questions

- **`runEscalate` surface not yet mapped.** The `runEscalate*`
  implementations in `escalate_impl.go` handle routing, mail
  composition, dry-run previews, and stale re-severity bumping —
  all rich behavior that this page does not yet cover. Good
  follow-up target for a deeper dive.
- **Severity shared variable quirk.** `escalateDryRun` is reused
  across the top-level and `stale` subcommand
  (`escalate.go:146`, `159`) — both commands bind `-n / --dry-run`
  to the same package var. Harmless in practice (each invocation
  runs one command) but an anti-pattern that could trip up
  refactoring.
- **`close --reason` is required but `create --reason` is not.**
  `MarkFlagRequired("reason")` is called on `escalateCloseCmd`
  (`escalate.go:155`) but not on the top-level. That asymmetry
  (closing requires saying why, creating doesn't) is either a
  conscious policy or a miss.
- **Not beads-exempt.** Unlike [mail](mail.md), [nudge](nudge.md),
  and [dnd](dnd.md), `gt escalate` requires bd to be present and
  healthy — which is awkward because escalation is the
  "something is wrong" primitive. A broken bd means no escalations.
  The project CLAUDE.md's dolt-trouble playbook directs agents to
  `gt escalate`, implying the assumption is "bd works, dolt is
  sick" — the two are separable.
