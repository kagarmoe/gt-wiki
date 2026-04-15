---
title: gt callbacks
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/callbacks.go
tags: [command, agents, mail, callbacks, mayor, routing, escalation]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# gt callbacks

Process pending callback messages in the Mayor's inbox — the routing
point for polecat completions, merge events, help requests,
escalations, and sling requests sent from Witnesses, Refineries, and
polecats.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`callbacks.go:71`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/callbacks.go`
(508 lines). Registration: `callbacks.go:112-118` attaches the single
`process` subcommand and registers the parent on `rootCmd`.

The parent `callbacksCmd` (`callbacks.go:69-83`) has no `RunE`, so
`gt callbacks` with no arguments prints help. All real behavior is
under `gt callbacks process`.

### Invocation

```
gt callbacks process [--dry-run] [-v | --verbose]
```

### Mailbox

`runCallbacksProcess` (`callbacks.go:120-201`):

1. Requires a Gas Town workspace via
   `workspace.FindFromCwdOrError`.
2. Opens the Mayor's mailbox: `router.GetMailbox("mayor/")` — that
   is, callbacks are always processed from the Mayor's inbox, not
   the caller's. The Mayor is the fixed routing hub.
3. Calls `mailbox.ListUnread()` to enumerate pending messages. If
   empty, prints `○ No pending callbacks` and exits.
4. Iterates each message, classifying and dispatching via
   `processCallback`.
5. Prints per-message results, then a summary line: `Processed N/M
   callbacks` (or `Dry run: would process N/M callbacks`), with an
   errors count in parentheses if any.

### Callback types

Six known patterns plus "unknown" (`callbacks.go:19-56`):

| CallbackType | subject regex | source |
|---|---|---|
| `polecat_done`    | `^POLECAT_DONE\s+(\S+)`          | `callbacks.go:21`    |
| `merge_rejected`  | `^Merge Request Rejected:\s+(.+)` | `callbacks.go:24`    |
| `merge_completed` | `^Merge Request Completed:\s+(.+)`| `callbacks.go:27`    |
| `help`            | `^HELP:\s+(.+)`                  | `callbacks.go:30`    |
| `escalation`      | `^ESCALATION:\s+(.+)`            | `callbacks.go:33`    |
| `sling`           | `^SLING_REQUEST:\s+(\S+)`        | `callbacks.go:36`    |

The `classifyCallback` switch at `callbacks.go:257-274` tries each
pattern in order and returns `CallbackUnknown` when none match.

**Removed by design** (`callbacks.go:38-41, 54-56`):
`WITNESS_REPORT` and `REFINERY_REPORT`. The inline note says
Witnesses and Refineries handle their duties autonomously and only
escalate genuine problems — not routine status updates.

### Per-type handlers

All handlers take `(townRoot, msg, dryRun)` and return
`(action string, err error)`:

- **`handlePolecatDone`** (`callbacks.go:278-307`). Parses polecat
  name from subject regex; parses `Exit:` and `Issue:` from the body
  line-by-line. Logs the completion via `logCallback` (which uses
  `townlog.EventCallback` at `callbacks.go:507`). No external
  dispatch — just bookkeeping.

- **`handleMergeCompleted`** (`callbacks.go:310-354`). Parses branch
  from subject; parses `MR:`, `Source:`, `Commit:` from the body.
  Logs the merge and — if a source issue is present — calls
  `beads.New(cwd).Close(sourceIssue, reason)` to close it
  automatically, using the merge commit as the close reason. Close
  failures are non-fatal: the message is still marked handled.

- **`handleMergeRejected`** (`callbacks.go:357-385`). Parses branch
  from subject and `Reason:` from the body (first line after the
  keyword). Logs the rejection. Does not re-queue or notify the
  source worker beyond the log entry.

- **`handleHelp`** (`callbacks.go:389-436`). The most structured
  handler. Parses the help payload via `witness.ParseHelp`, then
  classifies it with `witness.AssessHelp` to produce a category,
  severity, and suggested routing. Maps severity →
  `mail.Priority` (`critical`→`Urgent`, `high`→`High`, else
  `Normal`), then forwards to `overseer` with an `[FWD][<SEV>] HELP:
  <topic>` subject and an assessment-rationale body. Waits on
  pending notifications via `router.WaitPendingNotifications()` so
  the forward completes before returning.

- **`handleEscalation`** (`callbacks.go:439-468`). Parses topic
  from subject; forwards to `overseer` with
  `[ESCALATION] <topic>` subject and `mail.PriorityUrgent`.

- **`handleSling`** (`callbacks.go:471-502`). Parses bead ID from
  subject and target `Rig:` from body. Errors if no rig is
  specified. Logs the request but does *not* actually execute the
  sling — the note at `callbacks.go:498-499` explains: "We don't
  actually spawn here — that would be done by the Deacon executing
  the sling command based on this request." The returned action
  string includes the exact `gt sling <bead> <rig>` invocation.

### Message archival

After handling, `processCallback` (`callbacks.go:204-254`) archives
handled messages by calling `mailbox.Delete(msg.ID)` — unless
`--dry-run` is set or the message was *not* handled (unknown type
or error). Dry-run shows what *would* be processed without touching
the mailbox. Unknown-type messages stay in the inbox.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--dry-run`   | bool | `false` | `process` | `callbacks.go:113` |
| `-v, --verbose` | bool | `false` | `process` | `callbacks.go:114` |

Verbose mode prints the message `From:` and `Subject:` under each
processed entry (`callbacks.go:169-172`).

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/callbacks.go:88-103` — Cobra `Long` text on `callbacksProcessCmd`.

### Verbatim

> Process all pending callbacks in the Mayor's inbox.
>
> Reads unread messages from the Mayor's inbox and handles each based on
> its type:
>
>   POLECAT_DONE       - Log completion, update stats
>   MERGE_COMPLETED    - Notify worker, close source issue
>   MERGE_REJECTED    - Notify worker of rejection reason
>   HELP:              - Route to human or handle if possible
>   ESCALATION:        - Log and route to human
>   SLING_REQUEST:     - Spawn polecat for the work
>
> Note: Witnesses and Refineries handle routine operations autonomously.
> They only send escalations for genuine problems, not status reports.
>
> Unknown message types are logged but left unprocessed.

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### `SLING_REQUEST` handler does not "spawn polecat for the work" — it only logs

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/callbacks.go:98`.
- **Docs claim:** the `Long`'s per-type table says `SLING_REQUEST: - Spawn polecat for the work`. Alongside the neighbouring rows (`POLECAT_DONE - Log completion, update stats`, `MERGE_COMPLETED - Notify worker, close source issue`), which accurately describe what the handlers do, this framing presents `SLING_REQUEST` as the row that causes `gt callbacks process` to dispatch work to a polecat.
- **Code does:** `handleSling` at `callbacks.go:471-502` parses the bead ID from the subject and the target rig from the body, then calls `logCallback` with `"sling_request: bead %s to rig %s"` and returns the action string `"logged sling request: %s to %s (execute with: gt sling %s %s)"`. The in-file comment at `callbacks.go:498-499` is explicit: `"Note: We don't actually spawn here - that would be done by the Deacon executing the sling command based on this request."` No `runSling` / `sling.Dispatch` / `exec.Command("gt", "sling", ...)` call exists anywhere in this file. The callback loop also archives the handled message after the "handled" return (`callbacks.go:246-251`), so the logged-but-not-spawned row disappears from the Mayor's inbox with nothing having been dispatched. A user reading the `Long` and running `gt callbacks process` on a mailbox containing `SLING_REQUEST:` rows would observe the rows disappear and reasonably conclude that polecats had been spawned for them, when in fact the spawn step is the Deacon's separate responsibility and happens asynchronously (if at all).
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — edit the `Long` row to describe what `handleSling` actually does. Candidate replacement: `"SLING_REQUEST: - Log the request (Deacon executes the actual sling)"` or `"SLING_REQUEST: - Log and archive (spawn is Deacon's responsibility via gt sling)"`. The row should make the logged-only nature clear so operators understand `gt callbacks process` does not itself drain the sling queue.
- **Release position:** `in-release` — `callbacks.go:88-103` `Long` text and `callbacks.go:471-502` `handleSling` body are both byte-identical at `v1.0.0`.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [mayor.md](mayor.md) — the fixed mailbox owner. `gt callbacks
  process` only drains `mayor/`.
- [sling.md](sling.md) — the command `handleSling` *would* run but
  explicitly defers to the Deacon for execution. Callers wanting to
  actually dispatch slung work run `gt sling` directly.
- [deacon.md](deacon.md) — per the Long help, callbacks run "during
  Deacon patrol" (`callbacks.go:73`). The Deacon is the expected
  caller of `gt callbacks process` in normal operation.
- `gt mail` — the transport layer (Sub B scope; cross-link
  pending).
- `gt witness` / `gt refinery` / `gt polecat` — the producers of
  callback messages. These are Sub B's commands; cross-links
  pending.
- `gt escalate` — adjacent escalation primitive (Sub B scope;
  cross-link pending).

## Notes / open questions

- **Fixed Mayor-only routing.** The mailbox is hard-coded to
  `mayor/`; there is no flag to process some other agent's inbox.
  Any agent running `gt callbacks process` acts on the Mayor's
  inbox. This is consistent with the Mayor being the router but
  means non-Mayor agents who accidentally run it will mutate the
  Mayor's mailbox.
- **`handleSling` doesn't sling.** → promoted to `## Drift` as
  `cobra drift`: the `Long` text at `callbacks.go:98` says
  `SLING_REQUEST: - Spawn polecat for the work`, but `handleSling` at
  `callbacks.go:471-502` only logs; the in-file comment at `:498-499`
  spells out that the actual spawn is Deacon's responsibility. See
  [../drift/README.md](../drift/README.md).
- **`handleMergeCompleted` uses `cwd` for beads.** Line
  `callbacks.go:343` calls `beads.New(cwd)` rather than
  `beads.New(townRoot)`. If the operator runs `gt callbacks
  process` from somewhere other than the town root, beads lookups
  may resolve the wrong database. Flag for investigation.
- **No priority filter.** `ListUnread` pulls everything in FIFO
  order with no way to handle urgent escalations first in a single
  pass.
- **Archival is best-effort.** `_ = mailbox.Delete(msg.ID)` at
  `callbacks.go:249` discards the error — a delete failure leaves
  an orphan "handled" message that will be reprocessed on the next
  run, potentially re-forwarding the same escalation.
