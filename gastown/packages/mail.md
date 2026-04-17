---
title: internal/mail
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/mail/types.go
  - /home/kimberly/repos/gastown/internal/mail/mailbox.go
  - /home/kimberly/repos/gastown/internal/mail/router.go
  - /home/kimberly/repos/gastown/internal/mail/resolve.go
  - /home/kimberly/repos/gastown/internal/mail/store.go
  - /home/kimberly/repos/gastown/internal/mail/delivery.go
  - /home/kimberly/repos/gastown/internal/mail/bd.go
tags: [package, data-layer, mail, messaging, beads-backed, delivery, routing, two-phase-delivery]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# internal/mail

**Durable** inter-agent messaging. Everything the [`gt mail`](../commands/mail.md),
[`gt broadcast`](../commands/broadcast.md), and related commands do
ultimately bottoms out in this package. Messages are **beads with
`label=gt:message`** — direct mail is a bead assigned to the recipient,
queue mail is an unassigned bead labelled with a queue name, channel
mail is a bead labelled with a channel name. A rich set of other labels
carries metadata that would normally be struct fields (sender, thread,
reply-to, message type, CC, claim state, two-phase delivery state).
This package is how gt speaks mail; the beads in Dolt are the wire
format.

**Go package path:** `github.com/steveyegge/gastown/internal/mail`
**File count:** 7 non-test go files (`types.go`, `mailbox.go`,
`router.go`, `resolve.go`, `store.go`, `delivery.go`, `bd.go`), 7 test
files. `router.go` is the largest (~62 KB, ~1880 lines); `mailbox.go`
and `types.go` are next.
**Imports (notable):** [`internal/beads`](beads.md) (the bd/store
interface), `github.com/steveyegge/beads` (the `beadsdk` in-process
store — bypass-the-subprocess fast path), [`internal/config`](config.md)
(messaging config, queue/announce/channel definitions),
[`internal/runtime`](../inventory/go-packages.md) (session ID lookup
for two-phase delivery), [`internal/telemetry`](telemetry.md) (OTel
spans per operation), [`internal/util`](util.md) (detached subprocess
helper), `github.com/gofrs/flock` (legacy JSONL mailbox file locking).
**Imported by (notable):** every mail/messaging command —
[`gt mail`](../commands/mail.md), [`gt broadcast`](../commands/broadcast.md),
and the many `gt mail` subcommands (check, send, show, reply, thread,
claim, release, ack, pin, archive, purge). Also the nudge side:
`internal/cmd/mail_check.go` mixes mail drains with
[`internal/nudge`](nudge.md) drops. Escalations go through the Router
as well.

## What it actually does

### Public API

Two main entities dominate the API: `Message` and `BeadsMessage` (two
views of the same thing); `Mailbox` (the per-identity reader/writer);
and `Router` (the sender / fan-out engine).

#### types.go — message shapes and address normalization

- **`Priority`** string type with constants `PriorityLow`,
  `PriorityNormal`, `PriorityHigh`, `PriorityUrgent`
  (`types.go:13-27`).
- **`MessageType`** with `TypeTask`, `TypeEscalation`, `TypeScavenge`,
  `TypeNotification`, `TypeReply` (`types.go:29-47`).
- **`Delivery`** with `DeliveryQueue` (default: sit in the inbox and
  wait for `gt mail check`) and `DeliveryInterrupt` (inject as a
  system-reminder directly, for URGENT/lifecycle/stuck-detection)
  (`types.go:49-60`).
- **`Message`** struct (`types.go:64-140`) — the in-memory
  representation: `ID`, `From`, `To`, `Subject`, `Body`, `Timestamp`,
  `Read`, `Priority`, `Type`, `Delivery`, `ThreadID`, `ReplyTo`,
  `Pinned`, `Wisp`, `CC`, `Queue`, `Channel`, `ClaimedBy`,
  `ClaimedAt`, `DeliveryState`, `DeliveryAckedBy`, `DeliveryAckedAt`,
  and an in-memory-only `SuppressNotify`.
- **Constructors:** `NewMessage` (direct), `NewReplyMessage` (inherits
  `ThreadID` from original), `NewQueueMessage`, `NewChannelMessage`
  (`types.go:143-207`).
- **Predicates:** `IsQueueMessage`, `IsChannelMessage`,
  `IsDirectMessage`, `IsClaimed` (`types.go:210-227`).
- `Validate() error` (`types.go:231-271`) — enforces exactly one of
  `To`/`Queue`/`Channel`, plus claim fields only on queue messages.
- `GenerateID() string` / `generateThreadID() string`
  (`types.go:277-295`) — 8-byte random hex for messages, 6-byte for
  threads. NOTE the comment at `types.go:275-276`: the generated
  `msg.ID` is **not** passed to `bd create`; bd auto-generates IDs with
  the correct database prefix. This ID is only for in-memory tracking.
- **`BeadsMessage`** (`types.go:299-325`) — the bd-side view returned by
  `bd list --json`. Title → Subject, Description → Body, Assignee → To,
  Labels → every other metadata field. Has private cached parse
  state and a `ParseLabels()` method (`types.go:329-369`) that extracts
  `from:`, `thread:`, `reply-to:`, `msg-type:`, `cc:`, `queue:`,
  `channel:`, `claimed-by:`, `claimed-at:`, plus delivery labels.
- **`ToMessage()` and helpers** (`types.go:387-530`) — converts
  `BeadsMessage` → `Message`, including beads integer priority
  (0=urgent, 1=high, 2=normal, 3=low) ↔ `Priority`.
- **Address normalization** (`types.go:553-624`) — `normalizeAddress`,
  `AddressToIdentity`, `identityToAddress`. Canonical forms:
  - `overseer` (the human operator, no trailing slash) stays as-is.
  - `mayor`, `mayor/`, `gastown/mayor` → `mayor/` (town-level).
  - `deacon`, `deacon/`, `gastown/deacon` → `deacon/`.
  - `gastown/polecats/Toast`, `gastown/crew/max`, `gastown/Toast` →
    `gastown/Toast` (crew and polecat subfolders collapse).
  Postel's Law is explicit in the comment: accept many inputs,
  produce one canonical form.

#### mailbox.go — per-identity inbox

- `Mailbox` struct (`mailbox.go:37-48`) — has two modes: **beads-mode**
  (the default; reads/writes beads via bd subprocess or in-process
  store) and **legacy JSONL-mode** (for crew workers with local
  `inbox.jsonl` files). The `legacy bool` flag discriminates.
- Constructors: `NewMailbox` (JSONL), `NewMailboxBeads`,
  `NewMailboxFromAddress` (follows `.beads/redirect` for shared-beads
  workers), `NewMailboxWithBeadsDir` (explicit dir)
  (`mailbox.go:52-88`). Plus store variants in `store.go`:
  `NewMailboxBeadsWithStore`, `NewMailboxWithBeadsDirAndStore`.
- `List` / `listBeads` / `listLegacy` / `listFromDir`
  (`mailbox.go:112-260`) — open messages for this identity, ordered by
  priority (urgent first) then timestamp (newest first). In beads mode,
  queries `bd list --label gt:message --assignee <id> --json` for each
  identity variant (there can be several, e.g. canonical vs. crew-
  prefixed), deduplicates across queries, then runs a second CC query.
  Includes both `open` and `hooked` (auto-assigned handoff mail).
- `identityVariants() []string` (`mailbox.go:388-...`) — returns all
  canonical forms of the mailbox's identity so assignee queries
  catch beads written under the old non-normalized form.
- `Get(id) / getBeads / getFromDir / getLegacy` (`mailbox.go:461-528`).
- `MarkRead(id)` / `markReadBeads(id)` / `closeInDir(id, beadsDir)`
  (`mailbox.go:529-608`) — marking read closes the underlying bead.
  This is the natural "read is terminal" model: reading mail is
  completing the issue.
- `MarkReadOnly` / `MarkUnreadOnly` (`mailbox.go:609-703`) — a lighter
  operation that adds/removes a `read` label **without** closing the
  bead. Used where the caller wants a read indicator but not terminal
  closure.
- `MarkUnread` (`mailbox.go:704-772`) — reopens a closed bead.
- `Delete`, `Archive`, `ArchivePath`, `appendToArchive`,
  `ListArchived`, `PurgeArchive`, `rewriteArchive`
  (`mailbox.go:773-1022`) — legacy-mode only, moving messages between
  `inbox.jsonl` and `archive.jsonl`.
- `SearchOptions` struct and `Search` (`mailbox.go:1023-1099`) —
  filter by sender, subject pattern, type, priority, thread, date.
- `Count(): total, unread` (`mailbox.go:1100-1120`).
- `AcknowledgeDeliveries(recipientAddress, messages)`
  (`mailbox.go:1122-1175`) — the second phase of two-phase delivery:
  the recipient confirms it saw the messages by stamping
  `delivery:acked` + `delivery-acked-by:` + `delivery-acked-at:` labels.
- `Append`, `appendLegacy`, `rewriteLegacy` (`mailbox.go:1177-1249`) —
  JSONL mailbox writers.
- `ListByThread`, `listByThreadBeads`, `listByThreadLegacy`
  (`mailbox.go:1251-1312`).
- Wisp-message support (`mailbox.go:261-380`): `listWispMessages`,
  `queryWispMessagesByAssignee`, `queryWispMessagesByCC`, `runWispSQL`.
  Wisps are **ephemeral messages stored in the same Dolt DB as regular
  messages but with the `ephemeral=true` column set, so they are not
  synced to git and get squashed on patrol**. Used for transient
  agent-to-agent notifications that would clutter the repo otherwise.
  Accessed via raw SQL because `bd list` filters them out by default.

#### store.go — in-process beadsdk integration

- `SetStore(store beadsdk.Storage)`, `Store()`,
  `NewMailboxBeadsWithStore`, `NewMailboxWithBeadsDirAndStore`
  (`store.go:24-53`).
- `storeListFromDir`, `storeGetFromDir`, `storeCloseInDir`,
  `storeMarkReadOnly`, `storeMarkUnreadOnly`, `storeMarkUnread`
  (`store.go:62-186`) — direct `beadsdk.Storage` calls, bypassing the
  `bd` subprocess. The file-level comment at `store.go:1-6` explains
  the motivation: each `bd` invocation was ~600ms of subprocess +
  Dolt-connection overhead; in-process store eliminates that entirely
  for mail queries.
- `sdkIssueToMessage(si *beadsdk.Issue) *Message` (`store.go:190-211`)
  — routes through `BeadsMessage.ToMessage()` so label parsing stays
  consistent with the subprocess path.

#### router.go — the sender and fan-out engine

Single largest file in the package (~62 KB). Owns every path from
"user passes an address to `gt mail send`" through "beads entries
created, recipients notified, telemetry recorded".

- **`Router` struct and constructors** (`router.go:41-78`): `NewRouter`,
  `NewRouterWithTownRoot`. Holds a `workDir`, `townRoot`, a lazily-
  resolved `beadsDir`, and a `sync.WaitGroup` of in-flight
  notification goroutines (`WaitPendingNotifications()` at
  `router.go:79-82` lets tests synchronize).
- **Address recognition and parsing** (`router.go:84-123`):
  `isListAddress`, `parseListName`, `isQueueAddress`, `parseQueueName`,
  `isAnnounceAddress`, `parseAnnounceName`, `isChannelAddress`,
  `parseChannelName` — prefix-driven dispatch (e.g., `list:` prefix
  means "expand from `config.MessagingConfig.Lists`").
- **Expansion from config** (`router.go:126-192`): `expandFromConfig`
  generic helper, `expandList`, `expandQueue`, `expandAnnounce` — the
  messaging config defines queue / announce / channel properties.
- **Group addresses** (`router.go:265-343`): a separate recognizer for
  "everyone in this rig", "all polecats", "overseer", etc.
  `isGroupAddress`, `parseGroupAddress` → `ParsedGroup`.
- **Agent beads cache** (`router.go:344-492`): the town maintains
  "agent beads" — one bead per known agent serving as a registry.
  Router reads them via `queryAgents` (`router.go:699-826`) to resolve
  group addresses to concrete identities. The cache has specialised
  parsers for rig/role patterns baked into bead descriptions:
  `parseRigAgentAddress`, `parseRigAgentAddressFromID`,
  `parseAgentAddressFromDescription`.
- **Resolution entry points** (`router.go:593-698`):
  `ResolveGroupAddress`, `resolveGroup`, `resolveOverseer`,
  `resolveTownAgents`, `resolveAgentsByRole`, `resolveAgentsByRig`.
- **Validation** (`router.go:930-1055`): `validateRecipient` and
  `validateAgentWorkspace` ensure an address has a real on-disk
  presence before sending. `resolveCrewShorthand` (`router.go:1056-1103`)
  rewrites `rig/alice` → `rig/crew/alice` when the short form is
  ambiguous.
- **Wisp heuristic** (`router.go:828-860`): `shouldBeWisp` — decides
  whether a given message should be marked ephemeral, e.g. daemon-to-
  daemon notifications that don't deserve git history.
- **The send path** (`router.go:861-1513`):
  - `Send(msg)` is the top-level entry: dispatches to `sendToGroup`,
    `sendToList`, `sendToQueue`, `sendToAnnounce`, `sendToChannel`, or
    `sendToSingle` based on routing fields.
  - `sendToSingle` (`router.go:1104-1205`) creates one bead per message
    via `bd create` (or the in-process store path), stamping the
    `gt:message` label, then calls `notifyRecipient`.
  - `sendToList` fans out to resolved members. `sendToQueue`
    (`router.go:1247-1319`) writes a single unassigned bead labelled
    `queue:<name>`. `sendToAnnounce` / `pruneAnnounce`
    (`router.go:1320-1570`) add retention trimming — announce channels
    keep only the last N messages. `sendToChannel` writes a
    channel-labelled broadcast.
  - `GetMailbox(address) (*Mailbox, error)` (`router.go:1577-...`).
- **Notification** (`router.go:1597-1760`): `notifyRecipient` is where
  mail becomes a [`internal/nudge`](nudge.md) drop. Converts the
  message into a nudge payload (`nudgeKindForMessage`,
  `nudgePriorityForMailPriority`, `formatNotificationMessage`,
  `prioritySeverityLabel`) and calls `nudge.Enqueue` per resolved
  session ID. `enqueueReplyReminder` additionally queues a deferred
  reminder nudge so the sender's agent will poke them later if no
  reply arrives.
- `IsRecipientMuted(address)` / `isRecipientMuted(address)`
  (`router.go:1761-1790`) — respects the recipient's DND state.
- `addressToAgentBeadID`, `AddressToSessionIDs`
  (`router.go:1788-1835`) — the last two helpers translate addresses
  to the tmux session identifiers that `notifyRecipient` feeds into
  `nudge.Enqueue`.

#### resolve.go — secondary resolver

A different resolver, used by tests and by some sender paths that want
recipient types (`agent`, `queue`, `channel`, etc.) without going
through the full `Router`:

- `ErrUnknownRecipient`, `RecipientType` string type with constants
  (`resolve.go:26-36`).
- `Recipient` struct (`resolve.go:38-43`), `Resolver` struct
  (`resolve.go:45-50`).
- `NewResolver`, `Resolve`, `resolveWithVisited` (recursive with
  cycle detection), `resolveAgentAddress`, `validateAgentAddress`,
  `resolvePattern`, `resolveAtPattern`, `resolveByName`,
  `resolveBeadsGroup`, `expandGroupMembers`, `resolveQueue`,
  `resolveChannel` (`resolve.go:51-465`).
- `AgentBeadIDToAddress` (`resolve.go:466`) — inverse of
  `addressToAgentBeadID`.
- `matchPattern` (`resolve.go:521`) — glob-style address patterns.

#### delivery.go — two-phase delivery labels

Mail uses a **two-phase commit** pattern to distinguish "durably written"
from "recipient has acknowledged receipt". This is the only place in the
whole mail system where receipts matter for correctness (e.g., so a
supervisor can tell whether a message was actually seen vs. merely
written).

- Constants (`delivery.go:13-25`): `DeliveryStatePending`,
  `DeliveryStateAcked`, `DeliveryLabelPending = "delivery:pending"`,
  `DeliveryLabelAcked = "delivery:acked"`,
  `DeliveryLabelAckedByPrefix = "delivery-acked-by:"`,
  `DeliveryLabelAckedAtPrefix = "delivery-acked-at:"`.
- `DeliverySendLabels()` (`delivery.go:28-30`) — phase-1 labels (just
  `delivery:pending`).
- `DeliveryAckLabelSequence(recipientIdentity, at)` (`delivery.go:35-42`)
  — phase-2 labels, ordered for crash safety: acked-by and acked-at go
  first, `delivery:acked` last so the state read atomically reflects
  "pending" until the final write lands.
- `DeliveryAckLabelSequenceIdempotent` (`delivery.go:54-78`) — retry-
  safe variant that reuses an existing acked-at timestamp when the
  same recipient re-acks, unless crash recovery has produced mixed
  state (multiple acked-by labels), in which case a fresh timestamp
  is generated to avoid cross-recipient leakage.
- `AcknowledgeDeliveryBead(workDir, beadsDir, beadID, recipientIdentity)`
  (`delivery.go:84-107`) — reads existing labels, computes the
  idempotent ack sequence, writes each label via `bd label add`. Uses
  `beads.ResolveBeadsDirForID` to pick the right DB when the bead ID
  prefix crosses rigs.
- `ParseDeliveryLabels(labels) (state, ackedBy, ackedAt)`
  (`delivery.go:137-164`) — order-independent reader, tolerates
  lexicographic label ordering from `bd show --json`.

#### bd.go — shared subprocess runner

All `bd` subprocess invocations go through this file so timeouts,
environment setup, telemetry, and stale-PID cleanup are consistent:

- `bdReadTimeout = 60*time.Second`, `bdWriteTimeout = 60*time.Second`
  (`bd.go:15-22`). The comment at `bd.go:18-20` records a real
  incident: 30 seconds caused `signal: killed` under concurrent agent
  load because multiple `bd` processes compete for Dolt locks.
- `bdError` struct (`bd.go:26-50`) wrapping an inner error + captured
  stderr, with `Error()`, `Unwrap()`, `ContainsError(substr)`.
- `runBdCommand(ctx, args, workDir, beadsDir, extraEnv...)`
  (`bd.go:58-118`) — the one subprocess runner. Before each call it:
  1. Records a telemetry span via `telemetry.RecordMail`.
  2. Calls `beads.CleanStaleDoltServerPID` because a stale PID file
     would point bd at the wrong Dolt server and hang on 3307
     (`bd.go:62-64`).
  3. Injects `--flat` into `list --json` args via
     `beads.InjectFlatForListJSON` (required by bd v0.59+ to produce
     actual JSON; GH#2746) and retries without `--flat` if bd is
     older and rejects the flag.
  4. Sets `BEADS_DIR=<beadsDir>`, any database env from
     `beads.DatabaseEnv`, any extra env (e.g. `BD_IDENTITY=`), and
     OTel propagation env via `telemetry.OTELEnvForSubprocess`.
  5. Runs with `util.SetDetachedProcessGroup` so the child isn't
     killed by the parent's signal group.
- `bdReadCtx()`, `bdWriteCtx()` (`bd.go:131-142`) — context factories
  using the standard timeouts.

### Internals / Notable implementation

- **Beads as wire format.** The big architectural decision this
  package embodies: messages are not their own table. They are beads
  issues with a discriminator label (`gt:message`) and metadata-
  carrying labels for sender, thread, reply-to, type, CC, queue,
  channel, claim state, and delivery state. The upside is one audit
  trail, one query engine, one sync-to-git pipeline. The downside is
  a lot of label parsing and a dependency on the bd CLI's `--flat`
  behaviour (paid back partly by the in-process store path).
- **Two paths through beads.** Every bead read/write has a subprocess
  path (`bd`) and an in-process path (`beadsdk.Storage`). The in-
  process path is conditional on `m.store != nil` (set by the
  dedicated constructors and `SetStore`). Callers that hold a
  long-lived store (e.g., a daemon) enjoy ~600ms savings per operation;
  short-lived CLI commands fall through to the subprocess path.
- **Dedupe across identity variants.** Because address normalization
  has multiple valid forms for the same identity (`gastown/Toast`,
  `gastown/polecats/Toast`, `gastown/crew/Toast`), `listFromDir`
  queries per variant and deduplicates in-memory with a `seen` map.
  Otherwise a single message assigned to a non-canonical form would
  show up twice.
- **Wisps via raw SQL.** `bd list` does not surface wisps, so
  `queryWispMessagesByAssignee` goes straight to Dolt via `runWispSQL`
  (`mailbox.go:296-380`). Wisps live in the same table as regular
  messages; only the `ephemeral` flag differs. `escapeSQLString`
  guards identity/beadsDir inputs.
- **Two-phase delivery is append-only.** The `delivery.go` comment at
  line 131 is explicit: once `delivery:acked` appears, the state is
  "acked" even if `delivery:pending` is still present. Label writes
  ordered "by, at, acked" ensure that a crash between writes leaves
  the state as pending, not a partial ack.
- **Notification is async, best-effort.** `Router.notifyRecipient`
  runs in a goroutine and is tracked by
  `WaitPendingNotifications` only for test synchronization. Failed
  notifications are logged but do not undo the send — the message is
  already durably in beads.

### Usage pattern

Producer (CLI `gt mail send`):

```go
msg := mail.NewMessage("gastown/max", "gastown/Toast", "Review PR", "...")
msg.Priority = mail.PriorityHigh
r := mail.NewRouter(workDir)
if err := r.Send(msg); err != nil { ... }
r.WaitPendingNotifications()
```

Consumer (CLI `gt mail check`):

```go
mb := mail.NewMailboxFromAddress("gastown/Toast", workDir)
msgs, _ := mb.List()
for _, m := range msgs { ... }
mb.AcknowledgeDeliveries("gastown/Toast", msgs)
```

## Related wiki pages

- [gt](../binaries/gt.md)
- [gt mail](../commands/mail.md) — user-facing front door. Every
  subcommand (`send`, `check`, `show`, `reply`, `thread`, `archive`,
  `claim`, `release`, `ack`, `pin`, `purge`) routes into functions on
  this page.
- [gt broadcast](../commands/broadcast.md) — channel-message producer;
  uses `Router.sendToChannel`.
- [internal/beads](beads.md) — the shared bead/store plumbing. Mail's
  `Router` and `Mailbox` use `beads.ResolveBeadsDir`,
  `beads.EnsureCustomTypes`, `beads.CleanStaleDoltServerPID`,
  `beads.InjectFlatForListJSON`, `beads.DatabaseEnv`.
- [internal/nudge](nudge.md) — ephemeral counterpart. Mail is durable
  and commit-logged; nudge is fire-and-forget and filesystem-only.
  `Router.notifyRecipient` drops nudges as arrival notifications, so
  mail and nudge are layered rather than parallel.
- [internal/config](config.md) — `MessagingConfig`, `QueueConfig`,
  `AnnounceConfig` define queue/announce/channel names that `Router`
  expands.
- [internal/telemetry](telemetry.md) — `RecordMail`,
  `RecordMailMessage`, `OTELEnvForSubprocess`.
- [internal/util](util.md) — `SetDetachedProcessGroup`.
- [internal/lock](lock.md) — the mail package does **not** use the
  generic lock helper; it relies on bd's own transactional guarantees
  and on legacy `github.com/gofrs/flock` for the JSONL fallback.
- [internal/events](events.md) — mail events (`TypeMail`, etc.) are
  emitted by callers, not by this package.
- [go.mod](../files/go-mod.md) — `github.com/gofrs/flock`,
  `github.com/steveyegge/beads` (in-process SDK).
- [go-packages inventory](../inventory/go-packages.md)

## Notes / open questions

- `router.go` at ~1880 lines is the single-file hotspot for
  maintenance; future subagents adding new routing modes (e.g., a
  "web" broadcast surface) will land here. A natural split would be
  one file per send mode (`router_single.go`, `router_queue.go`,
  `router_channel.go`, …) but that is a refactor, not a bug.
- The `BeadsMessage.ParseLabels` pattern rebuilds parsed state on each
  call. Functions like `IsQueueMessage` (`types.go:462-465`) invoke
  `ParseLabels` unconditionally — fine for one-shot checks, wasteful
  in tight loops. Callers that iterate many messages should call
  `ToMessage` once and work from the resulting `*Message`.
- The Wisp SQL escape helper (`mailbox.go:381-386`) is a simple
  single-quote doubler. Safe for the current call sites because input
  comes from normalized identities and beadsDir paths, but future
  callers with richer inputs should use parameterized queries instead.
- The package's mix of `runBdCommand` (subprocess) and store direct
  calls means any new mail operation must be implemented **twice** —
  once for each path — and kept in sync. Adding a feature flag to
  force subprocess mode would be helpful for regression testing.
- `delivery.go` handles idempotent retry but does not handle the
  case where a crashed recipient ack is retried by a different
  recipient who has since claimed the same queue message — the
  `recipients` slice guard at `delivery.go:70-72` explicitly forces a
  fresh timestamp in that case. The semantics of "ack by multiple
  recipients" for the same message are therefore: last acker wins,
  with a fresh timestamp.
- The routing-discriminator label `gt:message` is load-bearing. Every
  mail query filters on it. If a non-mail bead ever lands with that
  label it will show up in inboxes. This is an invariant worth
  guarding in a future lint.
