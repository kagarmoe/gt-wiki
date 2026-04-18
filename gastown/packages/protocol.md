---
title: internal/protocol
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/protocol/types.go
  - /home/kimberly/repos/gastown/internal/protocol/messages.go
  - /home/kimberly/repos/gastown/internal/protocol/handlers.go
  - /home/kimberly/repos/gastown/internal/protocol/refinery_handlers.go
  - /home/kimberly/repos/gastown/internal/protocol/witness_handlers.go
tags: [package, protocol, mail, witness, refinery, polecat, merge]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
---

# internal/protocol

Structured Witness↔Refinery↔Polecat protocol messages that ride on top
of the mail system. Defines typed subjects (`MERGE_READY`, `MERGED`,
`MERGE_FAILED`, `FIX_NEEDED`, `REWORK_REQUEST`, `CONVOY_NEEDS_FEEDING`),
payload structs, key-value body encoding/parsing, a handler registry
that dispatches parsed messages to interface-typed callbacks, and
default `DefaultWitnessHandler`/`DefaultRefineryHandler` implementations
that wire the whole merge pipeline together.

**Go package path:** `github.com/steveyegge/gastown/internal/protocol`
**File count:** 5 go files, 1 test file
**Imports (notable):** stdlib (`errors`, `fmt`, `strings`, `time`),
plus [`internal/mail`](mail.md) for `Message`/`Router`/priority+type
constants, and [`internal/witness`](witness.md) for
`witness.AutoNukeIfClean` (used by `DefaultWitnessHandler` after a
successful merge).
**Imported by (notable):** the witness and refinery long-running
processes (they wrap the default handlers and drive them from their
poll loops), and by `gt done` code paths that construct
`POLECAT_DONE`/`MERGE_READY` messages directly.

## What it actually does

### Message types (`types.go`)

- `MessageType` is a string alias with constants `TypeMergeReady`,
  `TypeMerged`, `TypeMergeFailed`, `TypeFixNeeded`, `TypeReworkRequest`,
  `TypeConvoyNeedsFeeding` (`types.go:22-56`). Each maps one-to-one to a
  mail subject prefix — the on-wire format is
  `"<TYPE> <polecat-name>"` or `"CONVOY_NEEDS_FEEDING <convoy-id>"`.
- `ParseMessageType(subject)` (`types.go:60-81`) extracts the type from
  a subject by prefix match. Returns `""` if the subject isn't a known
  protocol type — callers use this as the "is this a protocol message?"
  predicate.
- `IsProtocolMessage(subject)` (`types.go:276-278`) and
  `ExtractPolecat(subject)` (`types.go:282-289`) are the companion
  helpers.
- `MergeReadyPayload`, `MergedPayload`, `MergeFailedPayload`,
  `FixNeededPayload`, `ReworkRequestPayload`,
  `ConvoyNeedsFeedingPayload`, `PolecatDonePayload` (`types.go:85-273`)
  are the fully-typed payload structs. Each is JSON-tagged for wire
  format and carries rig/polecat/branch/issue identity plus
  event-specific fields. `PolecatDonePayload.SkipMergeFlow()`
  (`types.go:255-257`) is the owned+direct convoy predicate used by
  the witness to decide whether to register work for merge processing.

### Message construction and parsing (`messages.go`)

- `NewMergeReadyMessage`, `NewMergedMessage`, `NewMergeFailedMessage`,
  `NewFixNeededMessage`, `NewReworkRequestMessage`,
  `NewConvoyNeedsFeedingMessage` (`messages.go:13-320`) each build a
  `*mail.Message` with the right `From`/`To` addressing, subject, and
  a plaintext key-value body built by a sibling `format*Body` helper.
  `MERGE_READY` goes witness→refinery; `MERGED`/`MERGE_FAILED`/
  `REWORK_REQUEST` go refinery→witness; `FIX_NEEDED` goes refinery
  directly to the polecat ("event-driven lifecycle — polecat stays
  alive and fixes in-place"); `CONVOY_NEEDS_FEEDING` goes
  refinery→`deacon/`.
- `formatRebaseInstructions(targetBranch)` (`messages.go:276-285`) is
  the canonical git fetch/rebase/push-f block embedded in every
  `REWORK_REQUEST` body.
- `ParseMergeReadyPayload`, `ParseMergedPayload`,
  `ParseMergeFailedPayload`, `ParseReworkRequestPayload`,
  `ParseFixNeededPayload`, `ParseConvoyNeedsFeedingPayload`,
  `ParsePolecatDonePayload` (`messages.go:186-513`) parse the key-value
  bodies back into their structs. Each validates required fields
  (Branch, Polecat, Rig) and returns a descriptive error listing the
  missing ones. `ParsePolecatDonePayload` does **not** enforce required
  fields — POLECAT_DONE is described as "a mail convention, not a
  formal protocol message," so it returns a best-effort parse.
- `parseField(body, key)` (`messages.go:517-529`) is the shared
  line-oriented parser. It looks for `Key: value` lines and trims.

### Handler registry (`handlers.go`)

- `ErrNoHandler` (`handlers.go:14`) — sentinel distinguishing
  "subject isn't a protocol message" from "subject IS a protocol
  message but no handler is registered."
- `Handler func(*mail.Message) error` (`handlers.go:17`) — callback
  signature.
- `HandlerRegistry` (`handlers.go:20-61`) — a
  `map[MessageType]Handler`. `Register` adds, `Handle` dispatches,
  `CanHandle` tests. `ProcessProtocolMessage`
  (`handlers.go:134-145`) is the top-level entry point returning
  `(handled, err)` with three distinct cases: not a protocol message
  (`false, nil`), handled successfully (`true, nil`), handled with
  error (`true, err`), or unhandled protocol message
  (`true, ErrNoHandler`).
- `WitnessHandler` / `RefineryHandler` interfaces
  (`handlers.go:65-81`) list the methods each role must implement.
- `WrapWitnessHandlers(h)` / `WrapRefineryHandlers(h)`
  (`handlers.go:84-127`) adapt an interface implementation into a
  populated `HandlerRegistry` with the correct parsing wired in.

### Default witness handler (`witness_handlers.go`)

- `DefaultWitnessHandler` (`witness_handlers.go:14-41`) carries `Rig`,
  `WorkDir`, a `*mail.Router` for sending, and a status output writer.
- `HandleMerged` (`witness_handlers.go:48-78`) — logs the merge,
  notifies the polecat via `notifyPolecatMerged`, then calls
  `witness.AutoNukeIfClean` to tear down the polecat worktree. Honors
  the nuke result's `Nuked`/`Skipped`/`Error` states.
- `HandleMergeFailed` (`witness_handlers.go:85-101`) — legacy path
  (replaced by FIX_NEEDED for most new code). Notifies the polecat
  that rework is needed.
- `HandlePolecatDone` (`witness_handlers.go:112-144`) — branches on
  `SkipMergeFlow()`: owned+direct convoys go straight to cleanup
  because the polecat already pushed to main; standard convoys just
  log receipt because the refinery will pick the MR bead up from
  beads directly.
- `HandleReworkRequest` (`witness_handlers.go:151-169`) — logs the
  conflict and delegates to `notifyPolecatRebase` which embeds a
  human-readable "git fetch / git rebase / git push -f" instruction
  block in the notification body (`witness_handlers.go:222-259`).

### Default refinery handler (`refinery_handlers.go`)

- `DefaultRefineryHandler` (`refinery_handlers.go:14-41`) mirrors the
  witness handler's shape.
- `HandleMergeReady` (`refinery_handlers.go:52-71`) — acks receipt.
  Belt-and-suspenders: if `Verified == "owned+direct: skip merge"`,
  it returns without doing anything. Otherwise it does nothing too —
  the comment explicitly documents that the refinery now queries beads
  directly for merge requests via `ReadyWithType("merge-request")`, so
  this handler is mostly a logging shim.
- `SendMerged`, `SendMergeFailed`, `SendFixNeeded`, `SendReworkRequest`
  (`refinery_handlers.go:75-101`) — outgoing message builders.
- `MergeOutcome` (`refinery_handlers.go:105-129`) — the union type
  describing "what happened to this merge attempt": `Success`,
  `Conflict`, `FailureType`, `Error`, `MergeCommit`, `ConflictFiles`,
  `MRBeadID`, `AttemptNumber`.
- `NotifyMergeOutcome` (`refinery_handlers.go:134-146`) — dispatch
  entry point. `Success → SendMerged`; `Conflict → SendReworkRequest`;
  otherwise `SendFixNeeded` directly to the polecat (so it can fix
  the code in-place without losing context).

## Related wiki pages

- [gt](../binaries/gt.md) — the binary containing witness/refinery
  processes that consume this package.
- [internal/mail](mail.md) — message envelope and router that protocol
  payloads ride on.
- [internal/beads](beads.md) — the merge-request bead path that the
  refinery now uses instead of the legacy `mrqueue` file.
- [witness role](../roles/witness.md) and [refinery role](../roles/refinery.md)
  — conceptual descriptions of the agents this protocol connects.
- [go-packages inventory](../inventory/go-packages.md).

## Failure modes

### Silent suppression (what errors are swallowed?)
- **Protocol message delivery cleanup:** The protocol package handles
  structured agent-to-agent communication. Silent suppressions appear
  in message acknowledgment and state-tracking paths where failures
  could leave messages in limbo between sent and acknowledged.
  **Present** — most suppressions are in error-path cleanup code.

## Notes / open questions

- `TypeMergeFailed` is marked LEGACY (`types.go:35-37`); new code
  should emit `TypeFixNeeded` which sends directly to the polecat
  rather than routing through the witness. The witness still honors
  `HandleMergeFailed` for backwards compatibility.
- `DefaultRefineryHandler.HandleMergeReady` (`refinery_handlers.go:52-71`)
  is essentially a no-op — the real merge work is driven by the
  refinery's beads query loop. The MERGE_READY message remains in the
  protocol as a liveness signal but no longer triggers behavior.
- POLECAT_DONE is intentionally under-validated
  (`messages.go:496-513`). Callers must tolerate partial payloads.
