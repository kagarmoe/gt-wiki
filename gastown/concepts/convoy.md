---
title: convoy (concept)
type: concept
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
audit_log: "Sweep 1 (2026-04-14), Sweep 2 Batch 5a (2026-04-15), Sweep 2 Batch 7a (2026-04-15)"
sources:
  - /home/kimberly/repos/gastown/internal/convoy/operations.go
  - /home/kimberly/repos/gastown/internal/convoy/multi_store.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_stage.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_launch.go
tags: [concept, convoy, cross-rig, tracking, work-unit, multi-stage, launch]
phase3_audited: 2026-04-14
phase3_findings: [drift, drift]
phase3_severities: [wrong, wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
---

# convoy

A **convoy** is a persistent, cross-rig, multi-stage work-
tracking unit. It exists above the rig level — one convoy can
track issues in several rigs at once — and it has its own
lifecycle of staging, launching, watching, feeding, and
landing. Where an individual bead represents one unit of work
and a rig contains one repo's workers, a convoy represents a
coherent batch of work that should complete together: a
protocol bump across services, a migration that spans
multiple codebases, or an epic whose children live in
different rigs.

Convoys are polecat-safe — that is, a polecat session can run
`gt convoy status` / `gt convoy check` without corrupting its
own sandbox — which makes them suitable for being surfaced to
working polecats as context.

## Origin and purpose

The rig-scoped model covers most Gas Town work: a polecat
picks up a bead, writes code, calls `gt done`, the Refinery
merges, and the work is over. Individual beads with
parent-child relationships give you nested tasks. Epics give
you a grouping of beads that share a theme.

Three problems this doesn't solve:

1. **Cross-rig work.** An epic's children may live in
   different rigs (dashboard, gastown, beads). No bead can
   "parent" a child in a different rig's database safely.
   The relationship has to live somewhere that knows about
   all the rigs.
2. **Tracking vs. blocking.** Sometimes you want a bead to
   *observe* the completion of other beads without *blocking*
   anything on them. Epics use parent-child, which is a
   strong-hierarchical relation; "wait for these to close"
   is a weaker relation — one that shouldn't propagate
   blocking semantics.
3. **Coordinated feed + land.** Work that spans multiple
   rigs needs coordinated dispatching: as one issue closes,
   the next ready one should go to a polecat without waiting
   for the next Deacon patrol tick. And when the whole batch
   finishes, something needs to close the convoy, land the
   swarm, and notify subscribers.

Convoys solve all three:

- They use a `tracks` dependency type — a non-blocking
  relation. A convoy tracks beads rather than parenting
  them.
- They live in the town's `hq` store, above any rig, so
  cross-rig tracking is natural.
- They have their own launch → feed → land state machine
  with explicit stages.

## What distinguishes a convoy from a rig or a molecule

Explicit answer for future readers:

A **rig** is a workspace: one codebase, one set of polecats,
one refinery, one witness, one beads database. Rigs are
physical locations where work happens. See `concepts/rig.md`
(pending Sub C).

A **molecule** is a multi-step formula that composes multiple
gt commands into a higher-level operation (e.g.
`mol-witness-patrol`, `mol-dog-reaper`, `mol-convoy-feed`).
Molecules are behavior compositions. See
`concepts/molecule.md` (pending Sub C).

A **convoy** is a batch of related work that spans rigs and
has a coordinated lifecycle. Convoys are work collections,
not workspaces or behaviors.

Concretely:

- A rig contains multiple polecats; a convoy contains zero
  polecats (it tracks issues, not workers).
- A rig is bound to one beads database; a convoy lives in
  `hq` and references beads across many databases via
  prefix routing.
- A molecule is a TOML formula that composes commands; a
  convoy is a bead with tracked-issue dependencies and a
  status state machine.
- A molecule is how you say "do this work"; a convoy is how
  you say "these things belong together and track their
  completion as a group."
- A rig outlives its convoys; a convoy outlives any
  individual polecat or molecule that runs within it.

A convoy USES molecules (`mol-convoy-feed` for dispatch) and
spans rigs, but it is neither.

## Origin and naming

The metaphor: a convoy of trucks travelling together across
multiple checkpoints. The individual trucks (issues) can be
inspected separately, but they're managed as a single
caravan. When one gets stuck, the convoy can feed another
truck forward. When the last one passes the final
checkpoint, the convoy "lands" and the journey is over.

## How it's realized in the code

- **CLI command** — `gt convoy` is implemented in
  `/home/kimberly/repos/gastown/internal/cmd/convoy.go` (2739
  lines), with stage/launch/watch logic in sibling files
  (`convoy_stage.go`, `convoy_launch.go`, `convoy_watch.go`).
  Twelve subcommands total: `create`, `status`, `list`,
  `add`, `check`, `stranded`, `close`, `land`, `stage`,
  `launch`, `watch`, `unwatch`. See
  [`gt convoy`](../commands/convoy.md).
- **Status state machine** —
  `/home/kimberly/repos/gastown/internal/cmd/convoy.go:97-102`
  defines four statuses: `open`, `closed`, `staged_ready`,
  `staged_warnings`. Transitions are validated by
  `validateConvoyStatusTransition` at `convoy.go:129-163`.
  `staged_*` → `open` is launch; `staged_*` → `closed` is
  cancel; `open ↔ closed` is round-trippable.
- **Event-driven feeding** —
  [`internal/convoy`](../packages/convoy.md)
  (`operations.go` and `multi_store.go`) is the reactive
  continuation path. `CheckConvoysForIssue`
  (`operations.go:38-90`) is called by
  `internal/cmd/close.go:273` on every close event.
- **Cross-store resolution** —
  `/home/kimberly/repos/gastown/internal/convoy/multi_store.go`
  defines `StoreResolver`, which routes issue reads to the
  correct rig store based on prefix. Without this, cross-rig
  dependency reads see stale snapshots (gh#2624).
- **Refinery integration** —
  `internal/refinery/engineer.go:1804+` (`postMergeConvoyCheck`,
  `notifyDeaconConvoyFeeding`, `checkAndCloseCompletedConvoys`,
  `notifyConvoyCompletion`, `landConvoySwarm`). When the
  Refinery merges the last tracked issue of a convoy, it
  closes the convoy and lands the swarm.
- **Deacon safety net** — the Deacon's `feed-stranded` patrol
  walks convoys with ready-but-unassigned work and dispatches
  them via `gt sling`. This is the polling fallback for cases
  where the close-event continuation path was missed.
- **Tracking relation** — convoys use a `tracks` dependency
  type on bead relationships. Non-blocking — the convoy's
  close state is independent of whether its tracked issues
  would block normal dispatch.

## Lifecycle / state

A convoy passes through the following states during a
complete launch:

1. **Staged.** `gt convoy stage <epic|issues>` creates the
   convoy with status `staged_ready` or `staged_warnings`
   based on pre-launch validation. Tracked issues are
   identified; no work is dispatched yet. An operator can
   inspect the staged convoy and decide to launch or cancel.
2. **Launched (open).** `gt convoy launch <epic|issues>` is
   literally `gt convoy stage --launch`
   (`convoy_launch.go:328`). The convoy transitions from
   `staged_*` to `open` and initial dispatches fire via
   `mol-convoy-feed`.
3. **Feeding (open, reactive).** As tracked issues close
   (polecats complete work, the Refinery merges MRs),
   `convoy.CheckConvoysForIssue` fires the continuation and
   dispatches the next ready issue via `gt sling`. Only one
   issue per close event — the thundering-herd prevention
   property.
4. **Watching.** Subscribers registered via `gt convoy watch`
   get notifications on major state changes and completion.
5. **Landing.** When all tracked issues close, the Refinery's
   `postMergeConvoyCheck` path (or the Deacon's periodic
   `convoy check` sweep) closes the convoy and fires
   `landConvoySwarm` if the convoy is `gt:owned`.
6. **Closed.** The convoy is done. Its dependency snapshot
   and completion event persist in the bead.

Branches and escape hatches:

- **Stranded.** `gt convoy stranded` detects convoys with
  ready-but-unassigned work, empty convoys, or convoys where
  all tracked issues are blocked. Consumed by the Deacon's
  patrol as a "needs attention" signal.
- **Force-close.** `gt convoy close --force` closes a convoy
  even when tracked issues aren't done, for abort scenarios.
- **Mountain-Eater.** If the convoy is labelled `mountain`,
  the Witness's `trackConvoyFailures` logic auto-skips
  individual tracked issues after 3 polecat failures,
  unblocking the convoy rather than looping forever.

## How a convoy relates to a rig and a molecule

Explicit answer for future readers:

- **Convoy ↔ Rig.** A convoy lives ABOVE rigs. Its ID has
  the `hq-` prefix and it lives in the town's `hq` store.
  Its tracked issues can be in any number of rigs; the
  cross-store resolver routes reads to the correct rig
  database. When the convoy dispatches an issue, `gt sling`
  sends it to the rig the issue's prefix maps to (via
  `beads.GetRigNameForPrefix`). A convoy is never bound to a
  single rig; that's the whole point.
- **Convoy ↔ Molecule.** A convoy USES molecules. The
  `mol-convoy-feed` molecule is the standard dispatch path:
  when the convoy needs to feed the next issue, the Deacon
  runs `mol-convoy-feed` via a Dog. The `stage` / `launch`
  commands may also be driven via molecule compositions in
  some setups. Molecules are the "how" of convoy work; the
  convoy is the "what we're tracking".
- **Convoy ↔ Epic.** An epic is a parent-child container
  within one rig's beads database. A convoy can be built
  from an epic via `--from-epic <id>`, which auto-discovers
  tracked issues from the epic's slingable children. The
  convoy is the cross-rig projection; the epic is the
  in-rig hierarchy. They can coexist.

## Relationships with other concepts

- **wisp** — stranded-convoy feed markers and
  `mol-convoy-feed` molecule steps are wisps. See
  [`wisp` concept](wisp.md).
- **rig** — the workspaces a convoy's tracked issues live
  in. See `concepts/rig.md` (pending Sub C).
- **molecule** — the formulas convoys use for dispatch
  (`mol-convoy-feed`). See `concepts/molecule.md` (pending
  Sub C).
- **swarm** — the ephemeral set of workers currently
  assigned to a convoy's issues. Reuses the convoy ID; no
  separate identity. Per the CLI Long help at
  `internal/cmd/convoy.go:176-206`: "A swarm is ephemeral —
  the workers currently assigned to a convoy's issues."
- **merge request** — a wisp type. Convoy completion events
  can trigger when the refinery merges the LAST MR for the
  last tracked issue.

## Related wiki pages

- [internal/convoy](../packages/convoy.md) — the reactive
  continuation Go package.
- [gt convoy](../commands/convoy.md) — the CLI with 12
  subcommands.
- [gt close](../commands/close.md) — calls
  `CheckConvoysForIssue` on every close.
- [gt sling](../commands/sling.md) — the dispatch primitive
  convoys use for feeding.
- [convoy-launch workflow](../workflows/convoy-launch.md) —
  the multi-step launch → feed → land flow.
- [wisp concept](wisp.md) — ephemeral bead story.
- [refinery role](../roles/refinery.md) — closes the convoy
  on last-merge.
- [refinery package](../packages/refinery.md) —
  `postMergeConvoyCheck`, `landConvoySwarm`.
- [witness role](../roles/witness.md) — Mountain-Eater
  auto-skip for mountain convoys.
- [witness package](../packages/witness.md) —
  `trackConvoyFailures`, `mountain.go`.
- [deacon role](../roles/deacon.md) — safety-net polling via
  `feed-stranded`.
- [deacon package](../packages/deacon.md) —
  `feed_stranded.go`.
- [internal/beads](../packages/beads.md) —
  `ParseConvoyFields`, `ExtractPrefix`,
  `GetRigNameForPrefix`, `GetRigPathForPrefix`.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/docs/concepts/convoy.md` (lines 63-76)
- `/home/kimberly/repos/gastown/docs/skills/convoy/SKILL.md` (lines 38-60)

### Verbatim (docs/concepts/convoy.md)
> "OPEN ──(all issues close)──► LANDED/CLOSED"
> States listed: `open` ("Active tracking, work in progress") and `closed` ("All tracked issues closed, notification sent"). No mention of `staged_ready` or `staged_warnings`.

### Verbatim (docs/skills/convoy/SKILL.md)
> ```
> += EVENT-DRIVEN FEEDER (5s) =+
> |                              |
> |   GetAllEventsSince (SDK)    |
> ```
> "Event-driven (`operations.go`): Polls beads stores every ~5s for close events."

## Drift

### Event-driven feeder misattributed to operations.go
- **Claim source:** `docs/skills/convoy/SKILL.md` (lines 38-60, architecture diagram and paragraph at line 59)
- **Docs claim:** The 5s event-driven feeder lives in `operations.go`.
- **Code does:** The 5s event poll loop is `ConvoyManager.runEventPoll()` at `/home/kimberly/repos/gastown/internal/daemon/convoy_manager.go:186-247`. It calls `GetAllEventsSince` from the beads SDK. `operations.go` provides `CheckConvoysForIssue` (the synchronous close-event handler called from `close.go:273`), not the polling loop. The SKILL.md's own "Key source files" table (lines 380-391) correctly lists `convoy_manager.go` as owning `runEventPoll`, contradicting the architecture section.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — the architecture diagram and paragraph should attribute the 5s event poll to `daemon/convoy_manager.go`, not `operations.go`.
- **Release position:** `in-release`

**Note:** The wiki pages for [internal/convoy](../packages/convoy.md) and this concept page already describe the architecture correctly — the wiki is not affected. The drift is docs-internal (the SKILL.md's architecture section contradicts its own source-files table).

### docs/concepts/convoy.md omits staged_ready and staged_warnings states
- **Claim source:** `docs/concepts/convoy.md` (lines 66-76, lifecycle diagram and state table)
- **Docs claim:** "OPEN ──(all issues close)──► LANDED/CLOSED" with only two states listed: `open` and `closed`.
- **Code does:** `/home/kimberly/repos/gastown/internal/cmd/convoy.go:97-102` defines four statuses: `open`, `closed`, `staged_ready`, `staged_warnings`. The staged statuses are the pre-launch states created by `gt convoy stage`. Transitions are validated by `validateConvoyStatusTransition` at `convoy.go:129-163`: `staged_* → open` is launch, `staged_* → closed` is cancel. The full lifecycle is stage → launch → feed → land, not just open → closed.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — the concepts doc should include the staged states and the full lifecycle.
- **Release position:** `in-release`

## Notes / open questions

- **Convoys are polecat-safe.** The `gt convoy` command has
  `AnnotationPolecatSafe: "true"` (`convoy.go:168`). Most
  commands aren't polecat-safe — convoys are an exception
  because polecats sometimes need to query their own convoy
  context.
- **Non-blocking `tracks` relation is the key technical
  choice.** Using `blocks` would have made the convoy's
  close state propagate into normal dispatch filters (stopping
  unrelated work). `tracks` keeps the relationship visible
  without entangling it with dispatch policy.
- **Two feeding paths exist.** Event-driven (close-hook fires
  `CheckConvoysForIssue`) and polling (Deacon's
  `feed-stranded`). The event-driven path is primary; the
  polling path is a safety net for missed events. Worth a
  future drift check: how often does the safety net actually
  fire in a healthy town?
- **`gt:owned` convoys.** Convoys created with `--owned` have
  caller-managed lifecycle — no automatic witness / refinery
  registration. `gt convoy land` is for this case. Non-owned
  convoys land automatically via the Refinery's
  `landConvoySwarm`.
- **Short IDs.** `generateShortID` in `convoy.go:30-46` uses a
  5-char base36 suffix with ~1% birthday-paradox collision
  risk at ~1,100 IDs. Entropy source is swappable via
  `convoyIDEntropy` for tests.
- **The convoy state lives in the bead's description.**
  `beads.ParseConvoyFields` reads a structured block inside
  the description. This is how `base_branch`, staging
  metadata, and warnings travel with the convoy without
  needing a dedicated table.
- **Dashboard rigs were the motivating use case.** Per
  `multi_store.go:14-16` and gh#2624: "Convoys live in the HQ
  store but may track issues in rig stores (e.g., `ds-*` in
  dashboard). Without cross-store resolution, convoy tracking
  sees 0/0 for cross-database dependencies."
