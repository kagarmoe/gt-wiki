---
title: internal/convoy
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
sources:
  - /home/kimberly/repos/gastown/internal/convoy/operations.go
  - /home/kimberly/repos/gastown/internal/convoy/multi_store.go
tags: [package, data, convoy, cross-rig, tracking, feeding, multi-store]
---

# internal/convoy

Cross-rig convoy tracking operations: finding the convoys that
track a given issue, checking convoy completion, and reactively
feeding the next ready issue into `gt sling` when a tracked issue
closes. The package is the event-driven continuation path for
convoy work — it replaces polling-based patrol cycles with direct
close-event reactions, and it handles the multi-store resolution
needed when convoys live in the town's `hq` store but track issues
scattered across per-rig stores.

**Go package path:** `github.com/steveyegge/gastown/internal/convoy`
**File count:** 2 non-test `.go` files — `operations.go`,
`multi_store.go`.
**Role:** NONE — convoys are not agents. See
[`convoy` concept](../concepts/convoy.md) for the domain story.
**CLI command:** [`gt convoy`](../commands/convoy.md) (12
subcommands; the parent command and its helpers live in
`internal/cmd/convoy.go`, not here).
**Imports (notable):** `github.com/steveyegge/beads` (SDK),
[`internal/beads`](beads.md) for rig/prefix routing and
`ParseConvoyFields`, [`internal/util`](util.md) for subprocess
process-group helpers, stdlib `os/exec` for calling `gt sling` and
`gt convoy check`.
**Imported by (notable):** `internal/cmd/close.go` (line 273 calls
`convoy.CheckConvoysForIssue` on every issue close, per the
[`gt close`](../commands/close.md) close-hook path), the daemon's
close-event ingest, and any path that needs cross-store issue
resolution via `StoreResolver`.

## What it actually does

Two files, one purpose: translate "an issue closed" into "check
whether a convoy just completed, and if not, feed the next ready
issue in that convoy to a polecat". The feed is event-driven
rather than polled: closing an issue fires the continuation
without waiting for the next Deacon patrol tick.

### Entry point — `CheckConvoysForIssue`

- `CheckConvoysForIssue(ctx, store, townRoot, issueID, caller,
  logger, gtPath, isRigParked, resolver...) []string`
  (`operations.go:38-90`) — the public entry called from
  `internal/cmd/close.go` after a successful close. Parameters:
  - `store` — beads storage (nil skips convoy checks entirely).
  - `townRoot` — town root path for subprocess working directory.
  - `issueID` — the issue that just closed.
  - `caller` — identifier for logging (e.g., `"Convoy"`).
  - `logger` — optional log function; nil becomes a no-op.
  - `gtPath` — resolved path to the `gt` binary (callers use
    `exec.LookPath` or read from daemon config).
  - `isRigParked` — predicate for skipping parked/docked rigs.
  - `resolver` — variadic optional `*StoreResolver` for
    cross-database resolution.

  The flow:

  1. `getTrackingConvoys` finds all convoys with a `tracks`
     relation to the closed issue.
  2. For each convoy:
     - Skip if already closed (`isConvoyClosed`).
     - Skip if still staged (`isConvoyStaged` — staged convoys
       aren't launched yet and shouldn't be fed).
     - Call `gt convoy check <convoy-id>` via `runConvoyCheck`
       to drive the completion check.
     - If still open after the check, call `feedNextReadyIssue`
       to dispatch the next ready issue to a polecat via
       `gt sling`.

  Returns the convoy IDs that were checked.

- `getTrackingConvoys(ctx, store, issueID, logger) []string`
  (`operations.go:94-110`) — uses
  `store.GetDependentsWithMetadata` filtered by
  `DependencyType == "tracks"`. The dependents of the closed issue
  ARE the convoys tracking it.
- `isConvoyClosed(ctx, store, convoyID) bool`
  (`operations.go:113-119`) — fresh read of convoy status.
- `isConvoyStaged(ctx, store, convoyID) bool`
  (`operations.go:124-130`) — `strings.HasPrefix(status,
  "staged_")`. Fail-open on read error.
- `runConvoyCheck(ctx, townRoot, convoyID, gtPath) error`
  (`operations.go:136-148`) — shells out to `gt convoy check
  <convoy-id>` in the town root. Uses
  `util.SetProcessGroup` so SIGKILL can clean up children. On
  error, returns the error with stderr concatenated.

### Reactive feeding — `feedNextReadyIssue`

- `trackedIssue` struct (`operations.go:151-157`) — the projection
  used for scoring: `ID`, `Status`, `Assignee`, `Priority`,
  `IssueType`.
- `slingableTypes` set (`operations.go:163-169`) — only
  `task`, `bug`, `feature`, `chore`, and empty-type (defaults to
  task) are slingable. Epics, convoys, decisions, messages, and
  events are NOT dispatchable by themselves.
- `IsSlingableType(issueType) bool` (`operations.go:173-175`) —
  exported so the CLI `gt convoy stranded` path can reuse it.
- `blockingDepTypes` set (`operations.go:181-186`) — `blocks`,
  `conditional-blocks`, `waits-for`, `merge-blocks`. `parent-child`
  is intentionally EXCLUDED: a child task is dispatchable even if
  its parent epic is open. Consistent with molecule step behaviour
  in `internal/cmd/molecule_step.go`.
- `isIssueBlocked(ctx, store, issueID, resolver) bool`
  (`operations.go:201-275`) — the dependency gate. For each
  dependency of the issue:
  - Skip non-blocking types.
  - `tombstone` is always unblocked.
  - For `merge-blocks`: `closed` alone is NOT enough — the
    `CloseReason` must start with `"Merged in "` to confirm
    integration. Prevents dispatching against un-merged code
    (gh#1893).
  - For other types: `closed` = unblocked, else blocked.
  - **Cross-rig caveat**: the metadata snapshot may be stale for
    cross-rig blockers. When a `StoreResolver` is provided, stale
    candidates are collected and re-resolved via the rig store
    for fresh status. Without a resolver, falls back to the
    snapshot and returns `true` (assume blocked) on any
    non-closed status. See the function comment at
    `operations.go:189-200` and gh#2624.
- `feedNextReadyIssue(ctx, store, townRoot, convoyID, caller,
  logger, gtPath, isRigParked, resolver)`
  (`operations.go:285-350`) — the reactive feeder:
  1. `getConvoyTrackedIssues` fetches the tracked-issue list
     with fresh status.
  2. Parses `base_branch` from the convoy's description via
     `beads.ParseConvoyFields` (so `gt sling` can target the
     right branch).
  3. Sorts by `(Priority asc, ID asc)` for deterministic
     tie-breaking.
  4. Iterates in order. Skip issues that aren't `open` +
     unassigned. Skip non-slingable types. Skip blocked issues.
     Resolve the target rig via `rigForIssue`. Skip parked rigs.
     Then `dispatchIssue` — on success, RETURN (only one issue
     per call); on failure, try the next.

  Only one issue is dispatched per call. When that issue
  completes, the next close event triggers another feed cycle.
  This is the event-driven continuation property.

- `getConvoyTrackedIssues(ctx, store, convoyID, townRoot,
  resolver) []trackedIssue` (`operations.go:356-444`) — the
  multi-store-aware read:
  1. `store.GetDependenciesWithMetadata(convoyID)` gives the
     tracked set with stale metadata.
  2. Filter by `tracks` dependency type.
  3. `store.GetIssuesByIDs` refreshes status for issues in the
     local (hq) store.
  4. For issues NOT in the local store (e.g., `ds-*` beads when
     convoys live in `hq`), use the `StoreResolver` to query the
     appropriate rig store directly.
  5. If no resolver, fall back to
     `fetchCrossRigBeadStatus` which shells out to `bd show
     --json` in each rig's directory.
  6. Build the result from the freshest data per issue.

- `extractIssueID(id) string` (`operations.go:447-455`) — strips
  `external:<prefix>:<id>` wrapper.
- `rigForIssue(townRoot, issueID) string` (`operations.go:459-465`)
  — uses `beads.ExtractPrefix` + `beads.GetRigNameForPrefix`.
- `fetchCrossRigBeadStatus(townRoot, ids)` (`operations.go:471-
  523`) — the legacy shell-out fallback. Groups IDs by prefix,
  resolves each prefix to its rig directory, and runs `bd show
  --json` per rig. Pattern borrowed from
  `batchFetchBeadInfoByIDs` in
  `internal/cmd/capacity_dispatch.go`.
- `dispatchIssue(ctx, townRoot, issueID, rig, gtPath, baseBranch)
  error` (`operations.go:528-544`) — the final dispatch step.
  Runs `gt sling <issueID> <rig> --no-boot [--base-branch=<b>]`
  in the town root. `--no-boot` because the daemon is already
  running.

### Multi-store resolution — `multi_store.go`

- `StoreResolver` struct (`multi_store.go:17-23`) — wraps a map
  of store name → SDK storage, plus the town root for
  prefix → rig routing. The package doc comment at
  `multi_store.go:1-16` captures the motivation: in multi-rig Gas
  Town setups, each rig has its own Dolt database; convoys live
  in HQ but may track issues in rig stores (e.g., `ds-*` in
  dashboard); without cross-store resolution, convoy tracking
  sees `0/0` for cross-database dependencies (gh#2624).
- `NewStoreResolver(townRoot, stores) *StoreResolver`
  (`multi_store.go:27-32`) — constructor. If `stores` is nil or
  empty, all resolution methods fall through gracefully (returning
  nil / empty maps).
- `StoreResolver.ResolveIssues(ctx, ids) map[string]*beadsdk.Issue`
  (`multi_store.go:37-71`) — groups IDs by target store via
  prefix routing, then batches `GetIssuesByIDs` per store.
  Issues not found in any store are omitted from the result.
- `StoreResolver.ResolveDepsWithMetadata(ctx, issueID)
  []*beadsdk.IssueWithDependencyMetadata` (`multi_store.go:75-94`)
  — per-issue dependency fetch using the issue's home store.
  Returns nil on any error (fail-soft).
- `StoreResolver.storeForID(id) string` (`multi_store.go:98-118`)
  — strips any `external:` wrapper, extracts the prefix, and maps
  via `beads.GetRigNameForPrefix`:
  - Empty prefix → `""` (unknown, skip).
  - Town-level prefix or unknown-rig → `"hq"`.
  - Rig prefix → rig name.

### Notable design choices

- **Event-driven, not polling.** The whole point of this package
  is that convoys progress in response to close events rather
  than waiting for the next Deacon patrol cycle. The close-hook
  in `internal/cmd/close.go:273` is what makes the Deacon's
  `feed-stranded` command a safety net rather than the primary
  mechanism.
- **One dispatch per close event.** `feedNextReadyIssue` returns
  after the first successful dispatch. Prevents thundering-herd:
  closing one issue dispatches one new issue, not all of them.
  The next close event fires the next dispatch.
- **Cross-store resolution is opt-in via resolver.** Callers
  without a `StoreResolver` still work — they just can't
  accurately read cross-rig dependency status. The daemon has a
  resolver; most one-shot CLI paths don't.
- **Two fallback strategies for cross-rig reads.** `StoreResolver`
  is the fast path (direct SDK queries). `fetchCrossRigBeadStatus`
  is the legacy path (shells out to `bd show --json`). Both are
  kept because the resolver isn't always available and the
  subprocess is always available.
- **`parent-child` is NOT a blocking dependency.** Child tasks
  are dispatchable even when their parent epic is open. This is
  consistent with molecule step behaviour and is documented
  inline.
- **`merge-blocks` requires a real merge confirmation.** Closing
  a bead with `Merged in <MR-ID>` as the close reason is the
  signal that the code was actually integrated — just "closed"
  could mean "rejected" or "abandoned", which would be dangerous
  to dispatch against.
- **Slingable type set is closed.** Only leaf work items are
  dispatchable. Containers and non-work beads are explicitly
  excluded. Empty type defaults to task so un-typed beads still
  dispatch.
- **`IsSlingableType` is exported** specifically for
  `gt convoy stranded` to detect non-dispatchable beads as
  stranded-convoy indicators.

## Related wiki pages

- [`convoy` concept](../concepts/convoy.md) — domain explanation
  of what a convoy IS. Required reading before this package page.
- [`convoy-launch` workflow](../workflows/convoy-launch.md) —
  the multi-step launch → feed → land flow this package powers.
- [`gt convoy`](../commands/convoy.md) — CLI entry; 12
  subcommands. The parent command and helpers live in
  `internal/cmd/convoy.go`, NOT here.
- [`gt close`](../commands/close.md) — calls
  `CheckConvoysForIssue` on every close.
- [`gt sling`](../commands/sling.md) — the dispatch primitive
  `dispatchIssue` invokes.
- [`internal/beads`](beads.md) — `ExtractPrefix`,
  `GetRigNameForPrefix`, `GetRigPathForPrefix`, `ParseConvoyFields`,
  `ParseMRFields`.
- [`internal/util`](util.md) — `FirstLine`, `SetProcessGroup`,
  `SetDetachedProcessGroup`.
- [`deacon` package](deacon.md) — the Deacon runs the polling
  safety-net `feed-stranded` pass.
- [`deacon` role](../roles/deacon.md) — orchestrates the polling
  fallback.
- [`refinery` package](refinery.md) — the Engineer's
  `postMergeConvoyCheck` path closes completed convoys and lands
  the swarm.
- [`wisp` concept](../concepts/wisp.md) — stranded-convoy wisps
  are the primary artifact the safety-net path generates.

## Notes / open questions

- **Only `operations.go` and `multi_store.go` in this package.**
  The convoy CLI, state transitions, and staging logic are in
  `internal/cmd/convoy.go` + siblings (2739 lines) — not here.
  This split is a candidate for future refactor: much of
  `internal/cmd/convoy.go` could migrate into this package.
- **Variadic `resolver` parameter is a compatibility hack.** The
  function signature at `operations.go:38` uses
  `resolver ...*StoreResolver` so older call sites can compile
  without passing a resolver. Newer daemon paths always pass one.
  A future cleanup could make the parameter required and add a
  constructor helper for nil-resolver callers.
- **Fail-open vs fail-closed inconsistencies.** `isIssueBlocked`
  fails open (assume not blocked) on store nil / snapshot errors,
  but fails closed (assume blocked) on non-resolved cross-rig
  status. The asymmetry is intentional (better to dispatch than
  to wedge) but documenting it in the function comment would
  help future auditors.
- **`rigForIssue` can return `""`.** When it does,
  `feedNextReadyIssue` logs and skips. An issue that can't be
  routed to a rig is silently stranded — a future enhancement
  could file a wisp so this is visible to operators.
- **`fetchCrossRigBeadStatus` uses `bd` as a subprocess.** If
  `bd` is missing or incompatible, the fallback fails silently.
  The resolver path is preferred; the daemon always has a
  resolver, so this mostly matters for one-shot CLI use.
- **Package doc comment is on `operations.go:1-3`**, not on a
  dedicated `doc.go`. Fine today; worth a dedicated `doc.go` if
  the package grows.
