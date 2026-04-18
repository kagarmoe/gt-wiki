---
title: convoy-launch (workflow)
type: workflow
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/convoy.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_stage.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_launch.go
  - /home/kimberly/repos/gastown/internal/cmd/convoy_watch.go
  - /home/kimberly/repos/gastown/internal/convoy/operations.go
  - /home/kimberly/repos/gastown/internal/refinery/engineer.go
  - /home/kimberly/repos/gastown/internal/cmd/close.go
tags: [workflow, multi-step, convoy, launch, feed, land, cross-rig]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [cross-ref-only]
---

# convoy-launch

The end-to-end workflow for launching and completing a Gas
Town convoy: from the operator's `gt convoy launch` command
through staging, initial dispatch, event-driven feeding of
subsequent issues, and the eventual landing (convoy closure +
swarm cleanup) triggered by the Refinery when the last
tracked issue merges. A convoy launch touches every per-rig
agent (polecats, Refinery, Witness) plus the Deacon (as the
safety-net dispatcher) and the event-driven continuation
path in [`internal/convoy`](../packages/convoy.md).

This is the "happy path" for a multi-issue cross-rig batch.
Failure branches (stranded, mountain-skipped, force-closed)
are documented under "Failure modes" below.

## Trigger

The workflow is triggered by:

- **`gt convoy launch <epic|issues>`** — the normal operator
  entry point. Internally,
  `/home/kimberly/repos/gastown/internal/cmd/convoy_launch.go:328`
  delegates to `runConvoyStage` with `--launch`. A launch is
  literally "stage + immediate transition to open".
- **`gt convoy stage <epic|issues> --launch`** — the
  equivalent form. The `stage` subcommand takes a `--launch`
  flag that turns a staging op into a launch op.
- **`gt convoy add <convoy-id> <issue-id>...`** on a closed
  convoy — re-opens the convoy and effectively re-launches
  with added issues. Documented at
  `/home/kimberly/repos/gastown/internal/cmd/convoy.go:270-283`.

In molecule-driven setups, the same entry points can be
invoked from a higher-level formula; the underlying command
is the same.

## Steps

### Step 1: Stage the convoy

**Command:** `gt convoy stage <epic|issues> [--launch]`
**Source:** `convoy_stage.go:runConvoyStage` (entry at
`convoy_stage.go:204`) and helpers.

Staging does the pre-launch validation and transforms a
collection of inputs (an epic ID, a list of issue IDs, or
both) into a concrete convoy bead. The stage step:

1. Parses the inputs and discovers tracked issues. If
   `--from-epic <id>` is supplied, auto-discovers slingable
   children via `bdDepListRawIDs`
   (`convoy.go:483+`). Uses raw `dependencies` SQL queries
   because `bd dep list` can't handle cross-database
   dependency joins (gh#2624).
2. Filters tracked issues to slingable types via
   `convoy.IsSlingableType` (`operations.go:173-175`) —
   only `task`, `bug`, `feature`, `chore`, or empty-type
   beads are dispatchable. Epics, decisions, messages, and
   events are excluded.
3. Creates the convoy bead with an `hq-<short-id>` ID via
   `generateShortID` (`convoy.go:30-46`).
4. Writes `ConvoyFields` into the bead description:
   `base_branch`, `from_epic`, the merge strategy
   (`direct` / `mr` / `local`), owner, and any notify
   subscribers.
5. For each tracked issue, creates a `tracks` dependency
   from the convoy to the issue. `tracks` is non-blocking
   so it doesn't entangle with normal dispatch filtering.
6. Runs validation passes: unassigned-to-rig beads, blocked
   beads, wrong-type beads. Warnings are collected and
   stored on the bead.
7. Sets the convoy status to `staged_ready` (no warnings)
   or `staged_warnings` (some validation findings). Neither
   is "open" yet — staged convoys do not feed.

### Step 2: Transition staged → open (launch)

**Command:** `gt convoy launch <epic|issues>` or `stage
--launch`
**Source:** `convoy_launch.go:39` (`convoyLaunchCmd`), delegates
to `runConvoyStage` with `--launch` at
`convoy_launch.go:328`.

The launch step transitions the convoy from `staged_ready` /
`staged_warnings` to `open`. The transition is validated by
`validateConvoyStatusTransition` at `convoy.go:129-163` —
`staged_* → open` is allowed; `open → staged_*` is NOT
(staging is a pre-launch state only).

Once the convoy is `open`, the Deacon's `mol-convoy-feed`
formula becomes eligible to run against it, and the
event-driven continuation path becomes active.

### Step 3: Initial dispatch

**Source:** the Deacon's `mol-convoy-feed` molecule
(dispatched via the Dog chain) and/or the launch path
directly.

The first batch of issues is dispatched via `gt sling`. For
each issue in the convoy that is:

- open and unassigned,
- slingable type,
- not blocked by a non-closed dependency,
- whose rig is not parked/docked,

the launch path calls `gt sling <issue> <rig> --no-boot
[--base-branch=<b>]` to dispatch the work to a polecat in
the target rig. The rig is resolved via
`convoy.rigForIssue` (`operations.go:459-465`) which uses
prefix routing.

Depending on the configured `capacity_dispatch` policy, the
launch path may dispatch all ready issues at once or batch
the dispatches. The default is one dispatch per ready slot
per rig.

### Step 4: Watch subscription (optional)

**Command:** `gt convoy watch <convoy-id>` (or the
`--notify` flag on create)
**Source:** `convoy_watch.go:34` (`convoyWatchCmd`).

Subscribers registered at create time (via `--owner` or
`--notify`) are recorded on the convoy bead. Additional
subscribers can be added later via
`gt convoy watch <convoy-id>`. Subscribers receive
notifications via mail/nudge when the convoy:

- Makes significant progress (issues close).
- Becomes stranded.
- Completes (lands).

### Step 5: Reactive continuation — event-driven feed

**Trigger:** every `gt close` on a tracked issue.
**Source:**
`/home/kimberly/repos/gastown/internal/cmd/close.go:273`
invokes `convoy.CheckConvoysForIssue`
(`operations.go:38-90`).

Each time a tracked issue closes (polecat completes, Refinery
merges, or direct `bd close`), the close path fires the
continuation:

1. `getTrackingConvoys(ctx, store, issueID)` finds the
   convoy(s) tracking this issue via
   `GetDependentsWithMetadata` filtered by
   `DependencyType == "tracks"` (`operations.go:94-110`).
2. For each tracking convoy:
   - Skip if already closed (`isConvoyClosed`,
     `operations.go:113-119`).
   - Skip if still staged (`isConvoyStaged`,
     `operations.go:124-130`) — staged convoys aren't
     launched yet and must not be fed.
   - Call `gt convoy check <convoy-id>` via
     `runConvoyCheck` (`operations.go:136-148`). This
     updates the convoy's internal state and is idempotent.
   - If the convoy is still open after the check, call
     `feedNextReadyIssue` (`operations.go:285-350`) to
     dispatch the NEXT ready issue.

`feedNextReadyIssue`:

1. Fetches fresh tracked-issue data via
   `getConvoyTrackedIssues` (`operations.go:356-444`),
   using the `StoreResolver` for cross-rig accuracy if
   supplied.
2. Parses `base_branch` from the convoy's ConvoyFields so
   `gt sling` can target the right branch.
3. Sorts tracked issues by `(Priority asc, ID asc)`.
4. Iterates and dispatches the FIRST issue that is open,
   unassigned, slingable, unblocked, and in a non-parked
   rig. Only ONE issue per close event is dispatched —
   prevents thundering-herd.
5. The dispatch uses `gt sling <id> <rig> --no-boot
   [--base-branch=<b>]` in the town root.

This is the primary feed mechanism for a launched convoy.
Subsequent closes drive subsequent dispatches, one at a
time, until no ready issues remain.

### Step 6: Safety-net polling (Deacon)

**Command:** driven by the Deacon's patrol loop, invokes
`gt convoy stranded` and `mol-convoy-feed` via Dog dispatch.
**Source:** `internal/deacon/feed_stranded.go` and the
Deacon's patrol cycle.

In case a close event was missed (daemon crash, bead close
without the close-hook, mail loss), the Deacon polls for
stranded convoys:

1. `gt convoy stranded` (`convoy.go:306-326`) walks open
   convoys and classifies them: ready-but-unassigned work,
   empty convoys, or all-blocked convoys.
2. For each stranded convoy with ready work, the Deacon
   dispatches a Dog running `mol-convoy-feed`.
3. `mol-convoy-feed` calls the same feed path as the
   event-driven continuation, dispatching the next ready
   issue.

This is a fallback — the primary feed is event-driven. The
safety net runs even if every close hook is missed.

### Step 7: Refinery merges tracked issues

**Source:** `internal/refinery/engineer.go` — the
Refinery's normal merge pipeline.

Each tracked issue's polecat eventually calls `gt done`,
which submits a merge-request wisp to the Refinery's queue.
The Refinery claims the MR, runs gates, squash-merges, and
post-merges:

- Closes the MR bead with `reason = "merged"`.
- Force-closes the source issue with
  `"Merged in <MR-ID>"`.
- Fires the close-event chain — which, in turn, invokes
  `CheckConvoysForIssue` from `internal/cmd/close.go:273`
  and dispatches the next convoy-tracked issue.

So each Refinery merge on a convoy-tracked issue:

1. Closes that tracked issue.
2. Triggers the convoy continuation.
3. Either dispatches the next ready issue (Step 5) or, if
   no ready issues remain, lets the convoy's completion
   path fire (Step 8).

### Step 8: Last-merge landing

**Source:**
`internal/refinery/engineer.go:1804+` —
`postMergeConvoyCheck`, `checkAndCloseCompletedConvoys`,
`notifyConvoyCompletion`, `landConvoySwarm`.

When the Refinery merges an issue AND that merge closed the
last remaining tracked issue in a convoy, the Refinery's
post-merge path:

1. `postMergeConvoyCheck(mr)` detects the convoy tracks
   this issue.
2. `checkAndCloseCompletedConvoys(townRoot, townBeads)`
   walks the convoy's tracked set, confirms they're all
   closed, and closes the convoy with a completion reason.
3. `notifyConvoyCompletion(townRoot, convoyID, title,
   description)` fires nudge/mail to subscribers.
4. `landConvoySwarm(townRoot, convoy)` is called for
   `gt:owned` convoys — it closes out the swarm of worktrees
   and releases per-polecat state. Non-owned convoys have
   their landing handled by the caller's workflow.

The convoy's status transitions from `open` to `closed`.
The workflow is complete.

## Actors involved

- **Operator / Mayor** — triggers the workflow via
  `gt convoy launch` (or its equivalents). Usually the
  [`mayor` role](../roles/mayor.md) in cross-rig work, or a
  human operator directly.
- **Convoy** — the bead itself; see
  [`convoy` concept](../concepts/convoy.md).
- **`internal/convoy` package** — the event-driven
  continuation. See
  [`internal/convoy` package](../packages/convoy.md).
- **Deacon** — the [`deacon` role](../roles/deacon.md)
  runs the `feed-stranded` polling safety net and dispatches
  dogs for `mol-convoy-feed`.
- **Dog** — the [`dog` role](../roles/dog.md) executes
  `mol-convoy-feed` when dispatched by the Deacon.
- **Polecats** — the [`polecat` role](../roles/polecat.md)
  does the actual work on each tracked issue. One polecat
  per dispatched issue, per rig.
- **Witness** — per-rig [`witness` role](../roles/witness.md)
  watches polecat health during the workflow. If a polecat
  fails on a mountain-labelled convoy, the Witness's
  Mountain-Eater auto-skips after 3 failures.
- **Refinery** — per-rig [`refinery` role](../roles/refinery.md)
  merges work and drives the landing phase on last-merge.
- **Crew** — the [`crew` role](../roles/crew.md) workspaces
  are synced after each Refinery merge via
  `syncCrewWorkspaces`.

## Failure modes

- **Staging validation warnings.** If the stage step finds
  issues with missing rig prefixes, blocked deps, or wrong
  types, the convoy lands in `staged_warnings`. An operator
  can still launch it (`--launch` overrides), but the
  warnings are visible on the bead.
- **No ready issues at launch.** If every tracked issue is
  blocked or already assigned, the launch fires but no
  dispatches happen. The convoy sits in `open` state with
  zero activity. The Deacon's `feed-stranded` will eventually
  detect this as stranded.
- **Stranded mid-flight.** A tracked issue gets stuck (no
  progress, blocker won't close), all other tracked issues
  are either closed or also blocked, and no feed-event can
  fire. `gt convoy stranded` detects this and surfaces it to
  the Deacon. The resolution is typically human intervention
  or Mountain-Eater auto-skip for mountain convoys.
- **Mountain-Eater auto-skip.** For convoys labelled
  `mountain`, the Witness's `trackConvoyFailures` (in
  `internal/witness/mountain.go`) counts polecat failures per
  tracked issue. After `MountainMaxFailures = 3` failures,
  the Witness auto-skips the issue (marks blocked +
  `mountain:skipped` label). This unblocks the convoy at the
  cost of leaving one issue un-done.
- **Cross-rig resolution failure.** If the
  `StoreResolver` isn't available (e.g. CLI path without
  daemon context) and a tracked issue lives in a rig store
  the local `hq` store doesn't index, the convoy sees
  incomplete tracking. The fallback `fetchCrossRigBeadStatus`
  shells out to `bd show --json` per rig as a last resort.
- **Force-close.** `gt convoy close --force` aborts the
  workflow and closes the convoy regardless of tracked-issue
  state. Used for abort scenarios or when a convoy has been
  superseded by another.
- **Rig parked.** If a tracked issue's target rig is parked
  or docked, the feed path skips it with a log line and moves
  to the next issue. If ALL issues are in parked rigs, the
  convoy stalls until a rig is un-parked.

## Related wiki pages

- [internal/convoy package](../packages/convoy.md) — the
  event-driven continuation implementation.
- [convoy concept](../concepts/convoy.md) — the domain
  concept.
- [wisp concept](../concepts/wisp.md) — stranded-convoy feed
  markers and molecule steps are wisps.
- [gt convoy](../commands/convoy.md) — the 12-subcommand CLI.
- [gt close](../commands/close.md) — the close-event trigger.
- [gt sling](../commands/sling.md) — the dispatch primitive.
- [refinery role](../roles/refinery.md) — drives Step 7 and
  Step 8.
- [refinery package](../packages/refinery.md) —
  `postMergeConvoyCheck`, `checkAndCloseCompletedConvoys`,
  `landConvoySwarm`.
- [deacon role](../roles/deacon.md) — Step 6 safety net.
- [deacon package](../packages/deacon.md) — the
  `feed_stranded.go` implementation.
- [witness role](../roles/witness.md) — Mountain-Eater
  auto-skip for mountain convoys.
- [witness package](../packages/witness.md) — `mountain.go`.
- [polecat role](../roles/polecat.md) — executes individual
  tracked-issue work.
- [dog role](../roles/dog.md) — executes `mol-convoy-feed`
  when dispatched by the Deacon.
- [mayor role](../roles/mayor.md) — usual operator trigger
  for cross-rig work.

## Notes / open questions

- **Event-driven feeds are not instantaneous.** There's a
  short latency between `gt close` and the next dispatch
  because the close-hook runs `gt convoy check` as a
  subprocess before firing the feed. Measured empirically
  under 1 second in healthy conditions.
- **Only one dispatch per close event.** This is deliberate
  anti-thundering-herd but means a convoy with 10 ready
  issues at launch time only fires one initial dispatch per
  close event. Initial dispatch at launch itself may be
  configured differently (via `mol-convoy-feed` or
  capacity_dispatch policy).
- **`mol-convoy-feed` may fire from multiple sources.** The
  Deacon dispatches it as a safety net; the launch path may
  also invoke it during Step 3. These are not deduplicated
  at the molecule level — the close-event pathway's
  idempotency (via `gt convoy check`) is what prevents
  double-landing.
- **Landing is split across two sources.** The Refinery's
  `postMergeConvoyCheck` handles the last-merge case; the
  Deacon's periodic `convoy check` can also close convoys
  whose tracked issues all closed without going through the
  Refinery (e.g. direct `bd close`). Both paths are
  idempotent so running both is safe.
- **Subscriber notifications.** The notification channel is
  mail or nudge depending on subscriber address format.
  Completion notifications are via `notifyConvoyCompletion`
  in the Refinery. Intermediate state changes may or may not
  notify depending on the subscriber's preferences (stored
  on the bead).
- **Watch vs. subscribe distinction.** The CLI has `watch`
  and `unwatch` as explicit subscriber management, but the
  `--notify` flag on create and the `owner` field also act
  as subscribers. Three paths to subscribe, one path to
  unsubscribe. A future cleanup might unify them.
