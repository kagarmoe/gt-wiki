---
title: gt mail
type: command
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/mail.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_send.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_inbox.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_check.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_thread.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_search.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_channel.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_directory.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_drain.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_group.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_announce.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_hook.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_identity.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_queue.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, communication, mail, durable, polecat-safe, beads, messaging]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift, wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
---

# gt mail

The durable agent messaging system — a multi-subcommand family
whose parent command alone is over 540 lines and whose sibling
`mail_*.go` files implement ~15 subcommand runners. Mail is the
permanent audit trail; [nudge](nudge.md) is its ephemeral twin.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupComm` ("Communication") (`mail.go:57`)
**Polecat-safe:** yes (`Annotations: map[string]string{
AnnotationPolecatSafe: "true"}` at `mail.go:58`)
**Beads-exempt:** yes — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:53`
**Branch-check-exempt:** no

## What it actually does

Source (cobra surface): `/home/kimberly/repos/gastown/internal/cmd/mail.go:1-544`.
The parent `mailCmd`'s `RunE` is `requireSubcommand` (`mail.go:60`),
meaning `gt mail` with no subcommand errors out — every path goes
through one of the children.

### Mail routing (from the Long help)

`mail.go:66-82` draws an ASCII diagram of inbox layout:

```
Town (.beads/)
  Mayor Inbox
    mayor/
  gastown/ (rig mailboxes)
    witness      ← greenplace/witness
    refinery     ← greenplace/refinery
    Toast        ← greenplace/Toast
    crew/max     ← greenplace/crew/max
```

### Address formats

`mail.go:83-89`:

| address              | meaning                                |
|----------------------|-----------------------------------------|
| `mayor/`             | Mayor inbox                            |
| `<rig>/witness`      | Rig's Witness                          |
| `<rig>/refinery`     | Rig's Refinery                         |
| `<rig>/<polecat>`    | Polecat (e.g. `greenplace/Toast`)      |
| `<rig>/crew/<name>`  | Crew worker (e.g. `greenplace/crew/max`) |
| `--human`            | Special: human overseer                |
| `list:<name>`        | Mailing list (fans out per member)     |

Mailing lists live in `~/gt/config/messaging.json` and expand so
each recipient gets their own copy
(`mail.go:107-111`). Compare [nudge](nudge.md)'s
`channel:<name>` — different config section, similar idea.

### Storage model

From `mail.go:64-65`: "Messages are stored in beads as issues
with `type=message`." Mail is not a distinct data store; it's a
conventional use of beads plus a folder-of-labels dispatching
layer. That's why `mail` is beads-exempt at the preflight level
but still calls into the [`internal/mail`](../packages/mail.md) package
(which in turn uses [`internal/beads`](../packages/beads.md)) — it needs
bd installed to actually do anything.

### Subcommands

22 subcommands total. 17 registered in `mail.go:524-541`; 5 more
registered in sibling files via their own `init()` functions.

**Registered in `mail.go:524-541`:**

| subcommand | short | sibling file | notes |
|------------|-------|--------------|-------|
| `send` | Send a message | `mail_send.go` | Positional `<address>` or `--to`; many flags |
| `inbox` | Check inbox | `mail_inbox.go` | Default view; `--unread` / `--all` |
| `read` (alias `show`) | Read a specific message | `mail_thread.go` | By ID or 1-based index |
| `peek` | Show preview of first unread | — | Status-bar preview (exit 1 if none) |
| `delete` | Delete (ack) messages | — | Closes messages in beads |
| `archive` | Archive messages | — | `--stale` / `--dry-run` |
| `mark-read` (alias `ack`) | Mark as read without archiving | — | `--all` for silence on all unread |
| `mark-unread` | Mark as unread | — | — |
| `check` | Check for new mail (for hooks) | `mail_check.go` | `--inject` for Claude Code hooks; `--identity` |
| `thread` | View a message thread | `mail_thread.go` | `--json` |
| `reply` | Reply to a message | — | Auto reply-to / Re: subject |
| `claim` | Claim a message from a queue | — | Work-queue primitive (runner in `mail_queue.go`) |
| `release` | Release a claimed queue message | — | Undo `claim` (runner in `mail_queue.go`) |
| `clear` | Clear all messages from an inbox | — | Quiescence reset |
| `search` | Search messages by content | `mail_search.go` | Regex; `--from`/`--subject`/`--body`/`--archive` |
| `announces` | List or read announce channels | `mail_announce.go` | Bulletin boards |
| `drain` | Batch processing | `mail_drain.go` | Batch drain of mail queue |

**Registered in sibling-file `init()` blocks** (Phase 2 missed
these — see wiki-stale note below):

| subcommand | short | sibling file | children | notes |
|------------|-------|--------------|----------|-------|
| `channel` | Manage beads-native channels | `mail_channel.go:149` | `list`, `show`, `create`, `delete`, `subscribe`, `unsubscribe`, `subscribers` (7) | Pub/sub broadcast channels with retention policies |
| `directory` (aliases: `dir`, `addresses`) | List all valid mail addresses | `mail_directory.go:42` | none | Shows agent, group, queue, channel, and special addresses |
| `group` | Manage mail groups | `mail_group.go:115` | `list`, `show`, `create`, `add`, `remove`, `delete` (6) | Distribution groups with pattern/nested-group members |
| `hook` | Attach mail to your hook | `mail_hook.go:45` | none | Alias for `gt hook attach <mail-id>` |
| `queue` | Manage mail queues | `mail_queue.go:548` | `create`, `show`, `list`, `delete` (4) | Queue CRUD (distinct from `claim`/`release` which operate on messages) |

Not all runners were read for this page; they are future targets.

### Message types

Per `mail.go:114-118`:

- `task` — required processing
- `scavenge` — optional first-come work
- `notification` — informational (default)
- `reply` — response to a message

### Priority levels

Per `mail.go:120-125`:

| value | meaning |
|-------|---------|
| 0 | urgent / critical |
| 1 | high |
| 2 | normal (default) |
| 3 | low |
| 4 | backlog |

`--urgent` is a shortcut for `--priority 0`.

### Wisp vs permanent

`mail.go:473-475` — two mutually-compatible flags on send:

- `--wisp` (bool, default `true`) — send as wisp (ephemeral;
  local only).
- `--permanent` (bool, default `false`) — send as permanent
  (synced to remote).

The default is wisp, meaning most mail never makes it to remote
Dolt. This is a newer-sounding knob worth drift-checking against
docs.

### Auto-notification

`mail.go:470-472`:

- `--notify` / `-n` — bump priority to high
  (auto-nudge-notification is on by default for send; this
  only changes priority).
- `--no-notify` — **suppress** the auto-nudge-notification
  entirely. Mutually exclusive with `--notify`
  (`mail.go:472`).

The interesting behavior here is "auto-nudge": when you send
mail, the recipient is also nudged via the [nudge](nudge.md)
machinery so they can't sit on an unread inbox forever unless
sender explicitly passes `--no-notify`. That's how mail-and-
nudge work together: mail is the durable record, nudge is the
in-band "new mail" poke.

### `mail send` — flags

Defined at `mail.go:462-480`:

| flag | short | type | default | description |
|------|-------|------|---------|-------------|
| `--subject` | `-s` | string | `""` | Message subject (**required**, marked at `mail.go:480`) |
| `--message` | `-m` | string | `""` | Message body |
| `--body` | — | string | `""` | Alias for `--message` |
| `--stdin` | — | bool | `false` | Read body from stdin (quoting-safe) |
| `--priority` | — | int | `2` | Priority 0-4 |
| `--urgent` | — | bool | `false` | Set priority=0 |
| `--type` | — | string | `notification` | `task`/`scavenge`/`notification`/`reply` |
| `--reply-to` | — | string | `""` | Message ID this is replying to |
| `--notify` | `-n` | bool | `false` | Bump priority to high |
| `--no-notify` | — | bool | `false` | Suppress auto-nudge (mutually exclusive with `--notify`) |
| `--pinned` | — | bool | `false` | Pin message (handoff context that persists) |
| `--wisp` | — | bool | `true` | Send as wisp (ephemeral, default) |
| `--permanent` | — | bool | `false` | Send as permanent (synced to remote) |
| `--to` | — | string | `""` | Recipient (alt to positional) |
| `--from` | — | string | `""` | Override sender (relay/bridge use) |
| `--self` | — | bool | `false` | Send to self (auto-detect from cwd) |
| `--cc` | — | []string | `nil` | CC recipients (repeatable) |

### `mail inbox` — flags

`mail.go:483-487`:

- `--json` — JSON output.
- `--unread` / `-u` — only unread.
- `--all` / `-a` — all (default shows all anyway; this is an
  explicit override over `--unread`).
- `--identity` (alias `--address`) — explicit identity for
  polecats that have ambiguous cwd context.

### `mail check` — flags

`mail.go:493-496`. This is the **hook-facing** subcommand; per
`mail.go:273-287`:

Exit codes (normal):
- 0 — new mail available
- 1 — no new mail

Exit codes (`--inject` mode):
- 0 — always (hooks must not block)
- stdout — system-reminder block if mail exists, silent otherwise

Flags: `--inject`, `--json`, `--identity` / `--address`.

### `mail claim` / `mail release` — queue primitives

`mail.go:324-373` define queue-based work claiming. A queue is a
bead with a `claim_pattern` label (e.g. `"*"`,
`"gastown/polecats/*"`). `claim`:

1. Pick a queue (specified or "any eligible").
2. Add `claimed-by` and `claimed-at` labels to the oldest
   unclaimed message.
3. Print the claimed message.

Eligibility is enforced by matching the caller's identity
against `claim_pattern`. `release` reverses it: verifies the
caller is the one who claimed, strips labels, returns message
to the queue.

### `mail search` — regex search

`mail.go:399-426`. Case-insensitive regex by default across
subject and body. Flags: `--from <sender>` (substring),
`--subject` (subject-only), `--body` (body-only),
`--archive` (include closed messages), `--json`.

### `mail announces` — bulletin boards

`mail.go:428-458`. Announce channels are defined in
`~/gt/config/messaging.json` with reader patterns and a
`retain_count`. Unlike regular mail, announce messages are
NOT removed when read. Queries beads for
`announce_channel=<channel>` and displays reverse-chronological.

### `mail clear` — quiescence reset

`mail.go:375-397`. Enumerate + delete every message in the
target inbox. `mail.go:389-390` documents the use case: "Town
quiescence - reset all inboxes across workers efficiently." The
town-wide reset loops externally; `clear` is the per-inbox
primitive.

## Docs claim

### Source
- `internal/cmd/mail.go:92-95` — Cobra `Long` text COMMANDS section

### Verbatim
> COMMANDS:
>   inbox     View your inbox
>   send      Send a message
>   read      Read a specific message
>   mark      Mark messages read/unread

## Drift

### mailCmd.Long COMMANDS section lists 4 subcommands; 22 are registered

- **Claim source:** Cobra `Long` text at `internal/cmd/mail.go:92-95`
- **Docs claim:** The COMMANDS block enumerates 4 subcommands: `inbox`, `send`, `read`, `mark`.
- **Code does:** `mail.go:524-541` registers 17 subcommands in its own `init()` (`send`, `inbox`, `read`, `peek`, `delete`, `archive`, `mark-read`, `mark-unread`, `check`, `thread`, `reply`, `claim`, `release`, `clear`, `search`, `announces`, `drain`). Five more are registered by sibling-file `init()` blocks: `channel` at `mail_channel.go:149`, `directory` at `mail_directory.go:42`, `group` at `mail_group.go:115`, `hook` at `mail_hook.go:45`, `queue` at `mail_queue.go:548`. Total: 22 subcommands. The COMMANDS list is 82% incomplete.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — update the `Long` text's COMMANDS block to enumerate the actual subcommand set, or remove the hand-maintained list and rely on cobra's auto-generated `Available Commands:` block.
- **Release position:** `in-release` — `mail.go` COMMANDS section byte-identical at `v1.0.0`; all 5 sibling files present at `v1.0.0` with their `mailCmd.AddCommand(...)` registrations intact.

See also: [gastown/drift/README.md](../drift/README.md) for the consolidated corrections list.

**Wiki-stale inline fix (Phase 3):** Phase 2's Subcommands table listed 17 entries (all from `mail.go`) but missed 5 subcommand groups registered by sibling-file `init()` functions (`channel`, `directory`, `group`, `hook`, `queue`). Phase 2 listed all 13 sibling files in `sources:` frontmatter but did not read their `init()` blocks for AddCommand registrations. Fixed inline above: Subcommands section now enumerates all 22 subcommands in two tables (17 from `mail.go` + 5 from siblings). **Phase 2 root cause: `phase-2-incomplete` (heuristic)** — all 5 sibling files were present at `v1.0.0` with the same registrations; Phase 2 had access to them on 2026-04-11 and listed them in `sources:` but did not verify their command registrations. Same pattern as Batch 1b (`directive`, `hooks`), Batch 1c (`molecule`, `mq`, `wl`), and Batch 1d (`agents`). See also: [gastown/drift/README.md](../drift/README.md).

## Notes / open questions

- **`mail` is in the beads-exempt preflight** so it can start
  up even if bd is unhealthy — but every subcommand
  (`send`, `inbox`, `search`, …) actually needs a working bd
  to do its job. The exemption is about the root-command
  preflight, not about being bd-free.
- **Polecat-safe is real.** Unlike [broadcast](broadcast.md),
  [escalate](escalate.md), [dnd](dnd.md), polecats can use
  `gt mail` directly. Combined with `nudge`, this gives
  polecats a full comm surface (receive, send, durable,
  ephemeral).
- **Auto-nudge coupling.** `--no-notify` is the escape hatch
  for "send mail without poking". When absent, every `gt mail
  send` also triggers a `gt nudge`-style delivery. This is
  worth knowing: mail is never silent by default.
- **Wisp is default.** `--wisp` defaults `true` and
  `--permanent` defaults `false`. Unless the caller opts in,
  most mail is local-only and will not survive Dolt sync.
  This contradicts the intuition that "mail = durable" — the
  permanence tier is opt-in.
- **Parent RunE is `requireSubcommand`.** Running bare
  `gt mail` with no args errors. There's no default inbox
  view; you must say `gt mail inbox`.
- **22 subcommands across 14 source files.** → Phase 3 promoted
  the sibling-file subcommand discovery to the Subcommands table
  and filed the incomplete COMMANDS enumeration as cobra drift.
  Each sibling runner file likely deserves its own page down the
  road — especially `mail_drain.go`, which probably runs the
  queue-draining behavior that connects mail to the hook system.
- **Relationship to `handoff --collect`.** [handoff](handoff.md)
  composes a handoff mail; the `--collect` flag gathers
  state into that mail. Handoff mail is the canonical example
  of "must survive session death" → use mail, not nudge.
- **Peek-subcommand name collision.** `gt mail peek`
  (`mail.go:189-197`) shows a preview of the first unread
  message. This is a totally different thing from
  [peek](peek.md), which captures a session's tmux pane.
  Both live in `GroupComm` and both are named "peek" —
  potential user confusion.

### Related commands

- [nudge](nudge.md) — ephemeral twin. Mail auto-nudges on
  send unless `--no-notify`. Project CLAUDE.md: "Default to
  nudge for routine agent-to-agent communication. Only use
  mail when the message MUST survive the recipient's session
  death."
- [handoff](handoff.md) — composes handoff mail
  (`handoff.go:79-...`). The `--collect` flag auto-gathers
  state into the handoff message.
- [resume](resume.md) — reads mail on session start (likely;
  verify).
- [prime](prime.md) — SessionStart hook runner; consults mail
  state.
- [broadcast](broadcast.md) — fan-out via nudge (no mail).
- [escalate](escalate.md) — uses `mail:mayor` as one of its
  routing actions; an escalation becomes a mail bead.
- [dnd](dnd.md) / [notify](notify.md) — control whether the
  auto-nudge from `mail send` actually reaches the recipient.
- [feed](feed.md) — reads event logs; mail send / read events
  plausibly show up here.
- [peek](peek.md) — unrelated peek on sessions (name collision).
- [seance](seance.md) — also in the communication-adjacent
  family.
