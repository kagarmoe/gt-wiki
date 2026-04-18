---
title: "Investigating: message delivery failures"
type: investigation
status: verified
topic: gastown
created: 2026-04-17
updated: 2026-04-17
sources:
  - gastown/packages/nudge.md
  - gastown/packages/mail.md
  - gastown/packages/tmux.md
  - gastown/packages/session.md
  - gastown/commands/nudge.md
  - gastown/commands/nudge-poller.md
  - /home/kimberly/repos/gastown/internal/cmd/nudge.go
  - /home/kimberly/repos/gastown/internal/cmd/nudge_poller.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_send.go
  - /home/kimberly/repos/gastown/internal/cmd/mail_check.go
  - /home/kimberly/repos/gastown/internal/mail/router.go
  - /home/kimberly/repos/gastown/internal/nudge/queue.go
  - /home/kimberly/repos/gastown/internal/nudge/poller.go
  - /home/kimberly/repos/gastown/internal/tmux/tmux.go
tags: [investigation, message-delivery, diagnostic, nudge, mail]
---

# Investigating: message delivery failures

**Symptom:** `gt nudge` returns success but the target agent never
receives the message, OR `gt mail send` succeeds but the recipient
never sees a notification, OR queued nudges accumulate without being
drained.

Gas Town has two messaging systems with different failure profiles:

- **Nudge** (ephemeral): filesystem queue at
  `<townRoot>/.runtime/nudge_queue/<session>/`, drained by Claude
  Code's `UserPromptSubmit` hook or a background
  [nudge-poller](../../commands/nudge-poller.md). No Dolt dependency.
- **Mail** (durable): bead with `gt:message` label in Dolt, drained
  by `gt mail check`. Depends on Dolt being reachable. After writing
  the bead, mail's `Router.notifyRecipient` drops a nudge as the
  arrival notification.

Both systems ultimately deliver via tmux `send-keys` (the
`NudgeSession` protocol in [tmux](../../packages/tmux.md)).

## Decision tree

### 1. Is the delivery system nudge or mail?

**Check:** What command was used?

- `gt nudge` -> [Step 2: Nudge delivery path](#2-which-nudge-delivery-mode-was-used)
- `gt mail send` -> [Step 7: Mail send path](#7-did-the-mail-bead-get-created)

### 2. Which nudge delivery mode was used?

The `deliverNudge` function at `nudge.go:157-279` routes based on the
`--mode` flag (default: `wait-idle`).

| Mode | Mechanism | Failure signature |
|---|---|---|
| `wait-idle` | Wait for idle, then direct deliver; fall back to queue on timeout | Agent busy forever: nudge sits in queue |
| `queue` | Write JSON file to `<townRoot>/.runtime/nudge_queue/<session>/` | Queue written but never drained |
| `immediate` | `tmux send-keys` directly via `NudgeSession` | Text garbled, interleaved, or swallowed |

- **immediate** -> go to [Step 5: tmux delivery issues](#5-did-tmux-send-keys-actually-deliver)
- **queue or wait-idle** -> go to [Step 3: Is the queue writable?](#3-is-the-nudge-queue-writable)

### 3. Is the nudge queue writable?

**Check:** Verify the queue directory exists and is writable.

```bash
ls -la ~/gt/.runtime/nudge_queue/<session>/
```

The queue path is `<townRoot>/.runtime/nudge_queue/<session>/` where
`<session>` has `/` replaced with `_` (`queue.go:76-79`).

- **Directory missing** -> the queue was never initialized. This is
  normal for the first nudge; `Enqueue` at `queue.go:92-138` creates
  it with `os.MkdirAll`. If creation fails, check directory
  permissions on `<townRoot>/.runtime/`.
- **Directory exists, no `.json` files** -> the queue was drained
  already, or max depth was hit. `MaxQueueDepthV()` from operational
  config defaults to 50 (`queue.go:38-51`). Check for `.claimed`
  files — orphaned claims indicate a drainer crashed mid-delivery.
  See [nudge](../../packages/nudge.md) ## Failure modes -> Partial
  completion.
- **`.json` files present but not being drained** -> go to
  [Step 4: Is the drain mechanism running?](#4-is-the-drain-mechanism-running)
- **Permission denied** -> file ownership mismatch. Common with Docker
  bind mounts where the host UID differs from the container UID.

### 4. Is the drain mechanism running?

Queued nudges must be drained by one of two mechanisms:

**For Claude Code agents:**
The `UserPromptSubmit` hook calls `nudge.Drain` inline at
`mail_check.go:107-111`. This fires at every turn boundary when the
user submits a prompt.

**Check:** Verify the hook is installed:

```bash
# Inside the agent's session:
cat .claude/settings.json | grep -A5 UserPromptSubmit
```

If the hook is missing, `gt hooks install` reinstalls it. See
[gt nudge](../../commands/nudge.md) for the hook integration.

**For non-Claude agents (Gemini, Codex, Cursor):**
The [nudge-poller](../../commands/nudge-poller.md) daemon polls the
queue directory and injects via `tmux send-keys`.

**Check:** Is the poller running?

```bash
cat ~/gt/.runtime/nudge_poller/<session>.pid 2>/dev/null && \
  kill -0 $(cat ~/gt/.runtime/nudge_poller/<session>.pid) 2>/dev/null && \
  echo "alive" || echo "dead"
```

`nudge.StartPoller` at `poller.go:54-90` spawns a detached
`gt nudge-poller <session>` child. If the poller crashed, its PID
file persists but the process is dead. `pollerAlive` at
`poller.go:143-163` detects this and cleans up the stale PID file.

The `wait-idle` mode's fallback path at `nudge.go:216-218` starts a
poller idempotently after queuing, specifically to prevent this
failure. But if the poller was never started (manual session, no crew
manager), queued nudges sit forever.

**Fix:** Start a poller manually: `gt nudge-poller <session>` or
restart the agent via its manager which starts the poller
automatically.

**Additional drain path:** The `watchAndDeliver` function at
`nudge.go:296-340` (used by `wait-idle` mode) runs a synchronous idle
watcher for up to 60 seconds after queuing. If the agent becomes idle
within this window, it drains the queue and delivers directly. This is
a bridge for the gap between queue-write and the next hook/poller
drain.

### 5. Did tmux send-keys actually deliver?

The `NudgeSession` protocol at `tmux.go:1601-1729` is an 8-step
sequence where each step exists because of a real failure. Common
failure points:

**a. Target session doesn't exist:**
`tmux.HasSession` returns false. The session name may be wrong (role
shortcuts expand via `session.ParseAddress` at `identity.go:32-98`)
or the session died between the nudge being sent and delivery. See
[session](../../packages/session.md) ## What it actually does ->
identity.go.

**b. Pane is in copy/scroll mode:**
Step 1 of `NudgeSession` exits copy mode. If the user manually
scrolled and the exit fails, `send-keys` text goes to the copy mode
buffer instead of the shell.

**c. Rewind mode (Claude Code):**
Steps 0 and 6.5 dismiss Claude Code's double-Escape Rewind menu
(`isInRewindMode` at `tmux.go:1378`, issue gt-8el). If Rewind isn't
dismissed, the message is consumed by the Rewind UI.

**d. Escape/Enter timing race:**
Step 6 waits 600ms after sending Escape to exceed bash readline's
`keyseq-timeout` (500ms). If this timing fails (slow terminal, high
system load), ESC+Enter within 500ms becomes M-Enter, which does NOT
submit the line. See [tmux](../../packages/tmux.md) ## What it
actually does -> Nudge protocol.

**e. Nudge lock timeout:**
The in-process semaphore at `tmux.go:1248-1268` (`nudgeLockTimeout =
30s`) and cross-process flock at `flock_unix.go:16-42` serialize
concurrent nudges to the same session. If a previous nudge is stuck,
the lock times out and the current nudge fails.

**f. Message too large for single send-keys:**
`sendMessageToTarget` at `tmux.go:1509-1535` splits into 512-byte
chunks with 10ms inter-chunk delays. Very large messages (> 50KB) may
hit tmux buffer limits.

**g. Detached pane (no SIGWINCH):**
Step 8: `WakePaneIfDetached` at `tmux.go:1327` sends SIGWINCH.
Claude Code ignores typed input in detached panes until SIGWINCH
arrives. If this step fails, the text is in the pane buffer but the
agent doesn't process it.

### 6. Did the agent process the delivered text?

Even when tmux delivery succeeds (text appears in the pane), the agent
may not act on it.

- **Claude Code:** The text must be in the input area and followed by
  Enter. `sendEnterVerified` at `tmux.go:1434` polls pane content to
  confirm Enter was processed. If the agent is in a long-running tool
  call, the text sits in the input buffer until the tool returns.
- **Non-Claude agents:** Behavior depends on the runtime. Some agents
  process typed input immediately; others queue it.
- **DND enabled:** If the target has Do Not Disturb enabled,
  `deliverNudge` calls `isRecipientMuted` at the command level (for
  nudge commands) and `Router.notifyRecipient` checks `isRecipientMuted`
  at `router.go:1598-1603` (for mail notifications). Nudges are
  silently skipped unless `--force` is used.

---

## Mail-specific path

### 7. Did the mail bead get created?

**Check:** Is Dolt reachable? Mail's `runBdCommand` at `bd.go:58-118`
shells out to `bd create`, which requires a working Dolt connection.

If Dolt is unreachable, `bd create` hangs until `bdWriteTimeout = 60s`
at `bd.go:15-22`, then fails with "signal: killed".

-> **If Dolt is unreachable**, follow the
[data-plane investigation](data-plane.md) starting at Step 1.

If Dolt is reachable, verify the bead was created:

```bash
bd list --label gt:message --assignee <recipient> --json
```

- **Bead exists** -> the message was written. Go to
  [Step 8](#8-did-the-mail-notification-nudge-arrive).
- **Bead missing** -> check stderr from `gt mail send`. Common causes:
  `ErrUnknownRecipient` from `resolver.Resolve` at `mail_send.go:147`
  (address validation failure), or stale PID file causing `bd` to
  connect to the wrong server. See
  [beads](../../packages/beads.md) ## Failure modes -> Precondition
  violations.

### 8. Did the mail notification nudge arrive?

After `Router.Send` writes the bead, it calls `notifyRecipient` at
`router.go:1597-1675` in a goroutine. This function:

1. Checks DND status (`isRecipientMuted` at `router.go:1599-1603`).
2. Resolves the recipient to tmux session IDs via
   `AddressToSessionIDs` at `router.go:1788-1835`.
3. Tries wait-idle delivery first (`WaitForIdle` at
   `router.go:1638`).
4. Falls back to queue-based delivery via `nudge.Enqueue` at
   `router.go:1656-1665`.

**Notification is async and best-effort.** The goroutine's error is
logged but does not undo the send. The message IS in beads even if
notification fails. See [mail](../../packages/mail.md) ## Failure
modes -> Silent suppression.

- **Notification never arrived** -> the nudge failed (go back to
  [Step 2](#2-which-nudge-delivery-mode-was-used) to diagnose the nudge path).
- **Notification arrived but agent didn't check mail** -> the agent
  received the nudge prompt but `gt mail check` hasn't been called.
  Claude agents run `gt mail check --inject` in the
  `UserPromptSubmit` hook, which both drains nudges AND checks mail
  in one call.

### 9. Did the recipient acknowledge delivery?

Mail uses a two-phase delivery protocol (see
[mail](../../packages/mail.md) ## What it actually does ->
delivery.go). Phase 1 sets `delivery:pending`; phase 2 sets
`delivery:acked` after the recipient calls
`AcknowledgeDeliveries` at `mailbox.go:1122-1175`.

**Check:** Look at the bead's labels:

```bash
bd show <message-id> --json | grep delivery
```

- `delivery:pending` only -> recipient hasn't acknowledged yet.
- `delivery:acked` present -> recipient received and processed the
  message. If the recipient "didn't see it," the problem is in the
  agent's message handling, not delivery.

## Cycles and shared dependencies

- **Mail -> nudge -> tmux:** Mail notification failure often means
  the nudge failed to deliver. Diagnose via the nudge path
  (Steps 2-6) before concluding mail is broken.
- **Both depend on tmux:** If `tmux.ErrNoServer` appears, the entire
  tmux server is down. All nudge and mail delivery fails. Check
  `tmux list-sessions` and restart with `gt up` if needed.
- **Mail depends on Dolt; nudge does not:** If Dolt is down, mail
  fails at Step 7 but nudge still works for all three modes (the
  queue is filesystem-only). This is a key architectural distinction:
  nudge was designed as the messaging fallback when the data plane is
  broken.

## Related investigation workflows

- [Investigating: data-plane failures](data-plane.md) -- for Dolt
  reachability issues that cause mail failures (Step 7 above links
  here).
