---
title: internal/beads
type: package
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/beads/beads.go
  - /home/kimberly/repos/gastown/internal/beads/store.go
  - /home/kimberly/repos/gastown/internal/beads/database.go
  - /home/kimberly/repos/gastown/internal/beads/beads_redirect.go
  - /home/kimberly/repos/gastown/internal/beads/beads_types.go
  - /home/kimberly/repos/gastown/internal/beads/status.go
  - /home/kimberly/repos/gastown/internal/beads/force.go
  - /home/kimberly/repos/gastown/internal/beads/audit.go
  - /home/kimberly/repos/gastown/internal/beads/beads_agent.go
  - /home/kimberly/repos/gastown/internal/beads/routes.go
  - /home/kimberly/repos/gastown/internal/beads/integration.go
  - /home/kimberly/repos/gastown/internal/beads/agent_ids.go
  - /home/kimberly/repos/gastown/internal/beads/beads_channel.go
  - /home/kimberly/repos/gastown/internal/beads/beads_delegation.go
  - /home/kimberly/repos/gastown/internal/beads/beads_dog.go
  - /home/kimberly/repos/gastown/internal/beads/beads_escalation.go
  - /home/kimberly/repos/gastown/internal/beads/beads_group.go
  - /home/kimberly/repos/gastown/internal/beads/beads_merge_slot.go
  - /home/kimberly/repos/gastown/internal/beads/beads_mr.go
  - /home/kimberly/repos/gastown/internal/beads/beads_queue.go
  - /home/kimberly/repos/gastown/internal/beads/beads_rig.go
  - /home/kimberly/repos/gastown/internal/beads/beads_sling_context.go
  - /home/kimberly/repos/gastown/internal/beads/catalog.go
  - /home/kimberly/repos/gastown/internal/beads/config_yaml.go
  - /home/kimberly/repos/gastown/internal/beads/fields.go
  - /home/kimberly/repos/gastown/internal/beads/handoff.go
  - /home/kimberly/repos/gastown/internal/beads/molecule.go
  - /home/kimberly/repos/gastown/internal/beads/stale_pid.go
tags: [package, data-layer, beads, bd-wrapper, subprocess, sdk, hybrid, issue-tracker, dolt, routing, largest-package]
phase3_audited: 2026-04-15
phase3_findings: [wiki-stale]
phase3_severities: [incomplete]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [incomplete]
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression, precondition]
detail_depth: {params: 1, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/beads

Gas Town's **Go library interface to beads**. It owns everything gt does
against a beads database: issue CRUD, ready/blocked work queries, agent
beads, molecules, channels, escalations, delegation, merge slots, merge
requests, routing between per-rig databases, and the full menagerie of
domain-specific bead "shapes" that gt code manipulates. Every gt command
that reads or writes a bead goes through this package. It is a hybrid of
`bd` subprocess shell-outs and direct `github.com/steveyegge/beads` SDK
calls — if a `beadsdk.Storage` is attached, operations with in-process
implementations bypass the ~600 ms-per-call subprocess spawn; otherwise
they shell out to `bd`.

**This is the largest package in the gastown codebase** — 28 production
`.go` files totaling ~10,300 lines, plus 21 test files. The surface area
reflects the central role of beads in Gas Town: they are the data plane
for issues, agents, channels, mail routing, molecules, escalations,
delegations, merge requests, and rig identity. (Files with more than ~500
production lines: `beads.go` 1,689; `fields.go` 1,057; `beads_agent.go`
753; `store.go` 599; `molecule.go` 579; `beads_channel.go` 529;
`beads_types.go` 481; `agent_ids.go` 472; `beads_escalation.go` 442;
`beads_queue.go` 398; `beads_redirect.go` 381; `beads_group.go` 373;
`routes.go` 360.)

**Go package path:** `github.com/steveyegge/gastown/internal/beads`
**File count:** 28 go files, 21 test files
**Imports (notable):**

- `github.com/steveyegge/beads` (aliased as `beadsdk`, v0.63.3 per
  `/home/kimberly/repos/gastown/go.mod:18`) — the upstream beads SDK that
  provides in-process `Storage`, `Issue`, `WorkFilter`, etc.
- `github.com/gofrs/flock` — cross-process file locks on per-bead `.lock`
  files (e.g., agent bead lifecycle mutations).
- [`internal/telemetry`](telemetry.md) — every `run()` and
  `runWithRouting()` call is bracketed by `telemetry.RecordBDCall(...)`,
  timing the subprocess and attaching stdout/stderr (when opted in).
- [`internal/util`](util.md) — `util.SetDetachedProcessGroup(cmd)` on every
  subprocess.
- [`internal/config`](config.md) — pulls role config for agent bead
  construction (`beads_types.go` imports `internal/config`).
- [`internal/constants`](cli.md) — `constants.BeadsCustomTypesList()`
  drives the `bd config set types.custom` sentinel dance.
- [`internal/runtime`](cli.md) — runtime feature flags.

**Imported by (notable):** virtually every high-level package — witness
handlers, polecat manager, refinery engineer/manager, rig config/manager,
mail router, doctor checks (rig beads, agent beads, stale agent beads,
hooks), molecule engine, and the command layer's bd-wrapper family (see
`commands/bead.md`, `commands/close.md`, `commands/done.md`,
`commands/ready.md`, `commands/cat.md`, `commands/assign.md`,
`commands/compact.md`, `commands/cleanup.md`).

## What `internal/beads` is **not**

Three things share the word "beads"; readers routinely confuse them.

1. **`bd` the binary** — a standalone CLI written in Go and shipped as
   its own upstream project (`github.com/steveyegge/beads`, sometimes
   called "the beads CLI"). It's a full issue tracker with its own
   commands: `bd create`, `bd show`, `bd list`, `bd ready`, `bd close`,
   `bd sql`, `bd query`, etc. gt users rarely invoke `bd` directly; gt's
   command layer wraps most of it. `bd` lives outside this package.
2. **The `.beads/` directory** — the on-disk database directory that
   `bd` and this package read/write. It contains `metadata.json`
   (including `dolt_database` and `dolt_server_port`),
   `dolt-server.port`, a `redirect` file for worktree-to-rig redirection,
   per-bead `.locks/`, and the Dolt data itself (served by a local Dolt
   MySQL server on port 3307 by default). This package does not *own* the
   directory — it *locates*, *routes*, and *reads/writes* it, including
   following `redirect` files and resolving cross-rig prefixes via
   `routes.jsonl`.
3. **`internal/beads` — this package.** A Go library that gt uses to
   talk to beads databases. It shells out to `bd` for most operations,
   but also integrates the upstream `beadsdk.Storage` SDK for in-process
   calls that skip the subprocess cost. It's the glue layer, and it's
   where Gas Town-specific semantics (agent beads, molecules, merge
   slots, delegation metadata, rig identity beads, channel beads, escalation
   beads, delegation tracking, merge-request routing) live.

If something says "beads," the question is which of the three. The `bd`
binary is the tool; `.beads/` is the data; `internal/beads` is the
Go library.

## What it actually does

### The central type: `Beads`

`type Beads struct` (`beads.go:315-335`) wraps bd CLI operations for a
working directory plus optional overrides and an optional in-process
store:

```go
type Beads struct {
    workDir    string
    beadsDir   string         // Optional BEADS_DIR override
    isolated   bool           // Test isolation: strip inherited env
    serverPort int            // Test-only: point at alt Dolt port
    store      beadsdk.Storage // Optional in-process SDK store
    townRoot     string
    townRootOnce sync.Once
}
```

Constructors (`beads.go:337-361`, `store.go:25-45`):

- `New(workDir string) *Beads` — normal.
- `NewIsolated(workDir string) *Beads` — test isolation, strips
  `BD_ACTOR` / `BEADS_*` / `GT_ROOT` / `HOME` (`beads.go:610-634`).
- `NewIsolatedWithPort(workDir, port) *Beads` — isolation plus a custom
  Dolt server port for tests against a sandbox Dolt instance.
- `NewWithBeadsDir(workDir, beadsDir) *Beads` — explicit BEADS_DIR
  override; needed when running from a polecat worktree but targeting the
  rig-level beads.
- `NewWithStore(workDir, store) *Beads` — in-process SDK mode.
- `NewWithBeadsDirAndStore(workDir, beadsDir, store) *Beads` — both.
- `(*Beads).SetStore(store)` / `(*Beads).Store()` — attach or retrieve
  the store.
- `(*Beads).OpenStore(ctx) (beadsdk.Storage, func(), error)` —
  convenience that opens a store for the resolved beads directory via
  `beadsdk.OpenFromConfig(ctx, beadsDir)` and returns a cleanup function
  (`store.go:57-75`). Used by short-lived gt commands that want SDK
  performance for one run.

### Subprocess vs native SDK: the hybrid

Every read/write method has the same shape:

```go
func (b *Beads) Show(id string) (*Issue, error) {
    // route cross-rig queries
    if targetDir != b.getResolvedBeadsDir() { return target.Show(id) }
    if b.store != nil { return b.storeShow(id) }        // SDK path
    out, err := b.run("show", id, "--json")             // subprocess path
    ...
}
```

The SDK path (`store.go`) converts between the gastown `Issue` struct
(`beads.go:167-208`) and the SDK's `beadsdk.Issue` via `sdkIssueToIssue`
(`store.go:89-138`) and `sdkIssuesToIssues` (`store.go:141-149`), which
handles time → string conversions, label population, and dependency-
derived fields. Implementations of the SDK methods live next to each
other in `store.go`: `storeList`, `storeShow`, `storeShowMultiple`,
`storeCreate`, `storeUpdate`, `storeClose`, `storeReady`,
`storeReadyWithFilter`, `storeBlocked`, `storeSearch`,
`storeAddDependency`, `storeRemoveDependency`, `storeAddLabel`,
`storeRemoveLabel`, `storeGetLabels`, `storeDelegationSet`,
`storeDelegationClear`.

The subprocess path (`beads.go:411-525`) runs `exec.Command("bd", ...)`
with a carefully constructed environment:

- `BEADS_DIR` is explicitly set to the resolved beads directory to
  prevent inherited values from causing prefix mismatches
  (`beads.go:426-430`, fixed GH#803 / `gt-uygpe`).
- `BEADS_DOLT_*` port/host mirrors are computed from the target beads
  directory's `metadata.json` and `dolt-server.port` file
  (`beads.go:687-714`), so that cross-rig operations reach the correct
  Dolt server.
- `--allow-stale` is prepended when the installed `bd` binary supports it
  (probed once and cached by binary path — see
  `BdSupportsAllowStaleWithEnv` at `beads.go:58-94`).
- `--flat` is injected for `bd list --json` commands because bd v0.59+
  ignores `--json` unless `--flat` is also present
  (`InjectFlatForListJSON` at `beads.go:117-137`). If the installed bd
  doesn't recognize `--flat`, the call auto-retries without it
  (`beads.go:451-468`).
- Telemetry's `OTELEnvForSubprocess()` env is appended so bd subprocess
  telemetry joins gt's waterfall (`beads.go:440-441`).
- Every call records a `bd.call` event via
  `telemetry.RecordBDCall(...)` with timing, error, and (opt-in) stdout
  / stderr (`beads.go:413-417`).
- Error wrapping (`wrapError` at `beads.go:531-550`) maps "not found"
  stderr substrings to `ErrNotFound` and `exec.ErrNotFound` to
  `ErrNotInstalled`; otherwise returns `bd <args>: <stderr>`. Per the
  "ZFC" comments (`beads.go:25-26`), decision-making on stderr strings
  is deliberately minimized.
- `isSubprocessCrash` (`beads.go:555-565`) detects Dolt nil-pointer
  SIGSEGV panics so recoverable paths (like GH#1769) can fall back to a
  retry strategy.
- `stripStdoutWarnings` (`beads.go:853-872`) removes "warning:" lines
  that bd sometimes prints to stdout, which would otherwise corrupt JSON
  parsing.
- `isJSONBytes` (`beads.go:878-890`) checks whether stdout looks like
  JSON before unmarshalling, because `bd list --json` can return plain
  text like `"No issues found."` for empty results.

The practical upshot: if a gt command wants fast issue lookups and can
open a store (e.g., the daemon, long-lived services), it uses the SDK
path. If it wants one-off CLI semantics or needs features the SDK doesn't
expose (SQL passthrough, some bd-specific formatting), it uses
`b.run(...)`. Most methods check `b.store != nil` and dispatch.

### The core API — shared types

`beads.go:167-314` defines the shared data types:

- `Issue` — the gastown issue struct: 30 fields including all the
  bd columns plus gt-specific ones like `HookBead`, `AgentState`,
  `Metadata` (`json.RawMessage`, used for merged metadata blobs like
  `delegated_from` and merge-slot `holder`/`waiters`).
- `IssueDep` — one edge of a dependency graph as returned by `bd show`.
- `ListOptions` — filters for `bd list` / `storeList`: Status, Label
  (preferred), deprecated Type, Priority, Parent, Assignee, NoAssignee,
  Limit, Ephemeral (for searching the `wisps` table instead of the
  `issues` table on beads v0.59+).
- `CreateOptions` — Title, Labels (preferred), deprecated Type/Label,
  Priority, Description, Parent, Actor, Ephemeral.
- `UpdateOptions` — pointer-typed field mutators so `nil` means "don't
  change" and distinguishes from "set to empty", plus AddLabels,
  RemoveLabels, SetLabels (replacement).
- `SearchOptions` — Query, Status, Label, Limit, DescContains.
- Sentinel errors: `ErrNotInstalled`, `ErrNotFound`, `ErrFlagTitle`
  (`beads.go:28-31`).
- Label inspection helpers: `HasLabel`, `HasUncheckedCriteria`,
  `IsAgentBead` (checks `gt:agent` label or legacy `Type == "agent"`),
  `IsProtectedBead` (checks `gt:standing-orders`, `gt:keep`, `gt:role`,
  `gt:rig` — used to prevent automation from mutating protected beads).

### The core API — CRUD, queries, dependencies

From `beads.go` — this is the stable surface most gt code calls:

- `Init(prefix)` — `bd init` with optional prefix and test-server
  port.
- `List(opts ListOptions)` — open issues, optionally filtered.
  Dispatches to `storeList` when SDK is attached, otherwise `bd list
  --json` with all the `--flat` / `--allow-stale` / empty-result
  handling. Wisp-table search (`listEphemeral`) is a separate path using
  `bd query --json` because bd v0.59+ split ephemeral issues into a
  `wisps` table.
- `ListMergeRequests(opts)` — merges results from the issues table (via
  `List`) with results from the wisps table via a hand-rolled `bd sql
  --json` query, because MRs are created as wisps but `bd list` only
  queries the main table. Dedupes by ID.
- `ListByAssignee(assignee)` — convenience wrapper.
- `GetAssignedIssue(assignee)` — returns the first issue in open,
  in_progress, or the gt-specific `"hooked"` state that matches the
  assignee.
- `Ready()` — `bd ready --json`, or `storeReady()` under SDK mode.
- `ReadyForMol(moleculeID)` — `bd ready --mol <id>` delegating to bd's
  canonical `blocked_issues_cache` semantics. Under SDK mode uses
  `WorkFilter{ParentID: ...}`.
- `ReadyWithType(issueType)` — label-filtered ready query.
- `Show(id)` — routes via `ResolveRoutingTarget` for cross-rig IDs
  (`beads.go:1078-1108`); handles `bd show --json`'s array-wrapping
  output shape.
- `FindLatestIssueByTitleAndAssignee(title, assignee)` — exact-match
  scanner used by daemon-like flows that need to find "the latest one I
  created".
- `ShowMultiple(ids)` — bulk fetch returning `map[string]*Issue`.
- `Blocked()` — `bd blocked --json`, or `storeBlocked`.
- `Create(opts)` / `CreateWithID(id, opts)` — both guard against
  flag-like titles (`gt-e0kx5`: garbage beads from `--title --help`) via
  `IsFlagLikeTitle` (`beads.go:157-164`). `CreateWithID` applies
  `NeedsForceForID` (`force.go`): when a bead ID contains more than one
  hyphen (e.g. `st-stockdrop-polecat-nux`), bd's prefix inference can
  misfire, so `--force` is added to honor the explicit ID.
- `Search(opts SearchOptions)` — text search across title/description/id.
- `FindOpenBugsByTitle` / `CreateIfNoDuplicate` — duplicate detection
  for test-failure auto-filing.
- `Run(args...)` — public wrapper around `run()` for callers who need
  to run arbitrary bd commands not yet covered by typed methods.

Additional methods across the package cover: updating (title, status,
priority, description, assignee, labels), closing (with reason and
session), dependency add/remove, label add/remove, issue state
transitions, agent bead lifecycle, molecule attach/detach, merge-slot
acquisition, delegation metadata, channel and group beads, escalation
beads, handoff beads, dog beads, merge-request beads, and rig identity
beads.

### The domain files — what each `beads_*.go` covers

Reading strategy: each `beads_<thing>.go` file owns one Gas Town-specific
domain. Most expose `(*Beads).Create<Thing>Bead`, `Get<Thing>Bead`,
`Update<Thing>`, and closure/state transition methods. Sizes are noted
where they indicate complexity.

- **`beads.go` (1,689 lines)** — the entry file: struct, constructors,
  shared types, subprocess core (`run`, `runWithRouting`, env plumbing,
  `--allow-stale` probing, `--flat` injection), CRUD (`List`, `Show`,
  `Create`, `Create*`, `Search`, `Ready*`, `Blocked`), duplicate
  detection, and a large number of update/close/dependency helpers.
- **`store.go` (599 lines)** — the SDK integration layer. Every `store*`
  helper mirrors a subprocess method. Conversion between gastown `Issue`
  and `beadsdk.Issue`. Context management via `storeCtx()` (30-second
  timeout).
- **`database.go` (33 lines)** — reads `dolt_database` from
  `.beads/metadata.json`. Produces the
  `BEADS_DOLT_SERVER_DATABASE=<name>` env var that bd subprocesses need.
- **`force.go` (11 lines)** — `NeedsForceForID`: multi-hyphen IDs need
  `--force` to bypass bd's prefix-inference.
- **`status.go` (105 lines)** — typed enums `AgentState` and
  `IssueStatus` with semantic methods: `ProtectsFromCleanup`,
  `IsActive`, `BlocksRemoval`, `IsTerminal`, `IsAssigned`.
  `ResolveAgentState` prefers the description's `agent_state:` field over
  the legacy structured column (bd >= 0.62.0 dropped `bd agent state` as
  a writer).
- **`beads_redirect.go` (381 lines)** — `ResolveBeadsDir(workDir)`: if
  `<workDir>/.beads/redirect` exists, follow it (up to depth 3) to the
  real beads directory. Detects circular redirects (and removes the
  offending file). This is why a polecat worker under
  `crew/max/.beads/redirect -> ../../mayor/rig/.beads` transparently
  uses the rig-level beads.
- **`routes.go` (360 lines)** — `routes.jsonl`-based prefix routing.
  `ResolveRoutingTarget` / `GetRigPathForPrefix` / `ExtractPrefix`
  translate a bead ID prefix (e.g., `gt-`, `hq-`, `st-`) to the rig
  directory that owns it, so that a `Show("hq-cv-abc")` call from a gt
  rig automatically walks over to the HQ rig's beads directory and
  reads from there.
- **`beads_types.go` (481 lines)** — custom type management:
  `FindTownRoot`, `EnsureCustomTypes` with a two-level
  (in-memory + sentinel-file) cache, and the `bd config set
  types.custom` plumbing. Also handles auto-initializing a missing
  database via `bd init --server`.
- **`beads_agent.go` (753 lines)** — agent beads. The `AgentFields`
  struct (`beads_agent.go:38-59`) is the structured description payload
  for polecats, witnesses, refineries, deacons, mayors: role_type, rig,
  agent_state, hook_bead, cleanup_status, active_mr, notification_level,
  mode (normal or `ralph`), plus completion fields written by `gt done`
  (exit_type, mr_id, branch, mr_failed, push_failed, completion_time).
  `lockAgentBead(id)` provides a per-bead flock under `.locks/` for
  read-modify-write races. Methods: `CreateOrReopenAgentBead`,
  `ResetAgentBeadForReuse`, `UpdateAgentDescriptionFields`, hook/unhook,
  state transitions, completion handling.
- **`beads_channel.go` (529 lines)** — channel beads for beads-native
  pub/sub messaging. Separate from `internal/channelevents`: channel
  beads are persisted entities; channel events are file rendezvous.
  `ChannelFields` struct (`beads_channel.go:16-24`): `Name`,
  `Subscribers` (list), `Status` (active/closed),
  `RetentionCount`, `RetentionHours`, `CreatedBy`, `CreatedAt`.
  ID format: `hq-channel-<name>` — town-level, channels span rigs
  (`beads_channel.go:132-134`). Methods:
  `CreateChannelBead(name, subscribers, createdBy)`,
  `GetChannelBead(name)`, `GetChannelByID(id)`,
  `UpdateChannelSubscribers`, `SubscribeToChannel`,
  `UnsubscribeFromChannel`, `UpdateChannelRetention`,
  `UpdateChannelStatus`, `DeleteChannelBead`, `ListChannelBeads`,
  `LookupChannelByName` (ID-first then name-field scan).
  `EnforceChannelRetention(name)` at `beads_channel.go:386-453`
  prunes old messages by both count and time limits (on-write
  cleanup). `PruneAllChannels` at `beads_channel.go:459-529` is
  the Deacon-patrol backup cleanup with a 10% buffer for count-based
  pruning to avoid thrashing.
- **`beads_group.go` (373 lines)** — group beads for mail
  distribution. `GroupFields` struct (`beads_group.go:41-46`):
  `Name`, `Members` (addresses, patterns, or nested group names),
  `CreatedBy`, `CreatedAt`. `ValidateGroupName`
  (`beads_group.go:23-37`): lowercase alphanumeric + hyphens/
  underscores, max 64 chars. ID format: `hq-group-<name>` — town-
  level (`beads_group.go:134-136`). Methods:
  `CreateGroupBead(name, fields)`, `GetGroupByName(name)`,
  `GetGroupByID(id)`, `UpdateGroupMembers`, `AddGroupMember`,
  `RemoveGroupMember`, `DeleteGroupBead`, `ListGroupBeads`,
  `LookupGroupByName` (ID-first then name-field scan). Groups can
  nest — a member can be another group name, enabling hierarchical
  distribution lists.
- **`beads_queue.go` (398 lines)** — queue beads for work queues.
  `QueueFields` struct (`beads_queue.go:14-26`): `Name`,
  `ClaimPattern` (glob-style, e.g., `gastown/polecats/*`), `Status`
  (active/paused/closed), `MaxConcurrency`, `ProcessingOrder`
  (fifo/priority), plus count fields (`AvailableCount`,
  `ProcessingCount`, `CompletedCount`, `FailedCount`), `CreatedBy`,
  `CreatedAt`. ID format: `hq-q-<name>` (town-level) or
  `gt-q-<name>` (rig-level) — `beads_queue.go:160-165`. Methods:
  `CreateQueueBead(id, title, fields)`, `GetQueueBead(id)`,
  `UpdateQueueFields`, `UpdateQueueCounts`, `UpdateQueueStatus`,
  `ListQueueBeads`, `DeleteQueueBead`, `LookupQueueByName` (tries
  both prefixes then name-field scan). `MatchClaimPattern(pattern,
  identity)` at `beads_queue.go:337-370` implements glob matching
  (`*` = anyone, `gastown/polecats/*` = any polecat in gastown,
  `*/witness` = any witness). `FindEligibleQueues(identity)` at
  `beads_queue.go:373-398` returns all active queues the identity
  can claim from.
- **`beads_escalation.go` (442 lines)** — escalation beads (mayor gets
  pinged when a worker is stuck).
- **`beads_delegation.go` (177 lines)** — delegation tracking via
  metadata blobs: the `delegated_from` key inside
  `Issue.Metadata`. Merge/delete semantics handled by
  `mergeMetadataKey` / `deleteMetadataKey` (`store.go:572-599`).
- **`beads_merge_slot.go` (219 lines)** — merge-slot tokens for
  serialized merge execution. The merge slot is a single bead
  identified by the label `gt:merge-slot` whose `Description` stores
  a JSON blob `{"holder": "<actor>", "waiters": ["<actor1>", ...]}`
  (`beads_merge_slot.go:8-10`). `MergeSlotStatus` struct
  (`beads_merge_slot.go:20-26`) exposes `ID`, `Available`, `Holder`,
  `Waiters`, `Error`. Key methods: `MergeSlotCreate` creates the
  bead with empty holder (`beads_merge_slot.go:72-83`);
  `MergeSlotCheck` reads current status
  (`beads_merge_slot.go:87-96`); `MergeSlotAcquire(holder,
  addWaiter)` acquires the slot or adds the caller to the waiters
  list (`beads_merge_slot.go:103-163`); `MergeSlotRelease(holder)`
  clears the holder and promotes the first waiter to holder
  (`beads_merge_slot.go:167-202`); `MergeSlotEnsureExists` is the
  idempotent create-if-missing wrapper
  (`beads_merge_slot.go:206-219`). The bd `merge-slot` command was
  removed in v0.62; this implementation uses standard bead CRUD
  operations that remain available in v0.62+.
- **`beads_mr.go` (118 lines)** — merge request beads and merge-gate
  utilities (wispy MRs, usually with the `gt:merge-request` label).
- **`beads_rig.go` (281 lines)** — rig identity beads. Each rig has a
  canonical bead representing its identity and config.
- **`beads_dog.go` (115 lines)** — "dog" beads: witness-level utility
  agent beads.
- **`beads_sling_context.go` (134 lines)** — sling context used by
  `gt sling` to attach work to an agent's hook.
- **`handoff.go` (269 lines)** — handoff beads and the `pinned` /
  `hooked` status vocabulary. Exports `StatusPinned` and `StatusHooked`
  as untyped strings (re-exported typed in `status.go`).
- **`molecule.go` (579 lines)** — molecule support: composable
  workflow templates that expand into a DAG of beads. Core types:
  `MoleculeStep` (Ref, Title, Instructions, Needs, WaitsFor, Tier,
  Type, Backoff — `molecule.go:15-24`); `BackoffConfig` (Base,
  Multiplier, Max — `molecule.go:28-32`) for cost-saving wait-type
  steps. `ParseMoleculeSteps(description)` at `molecule.go:70-178`
  extracts `## Step: <ref>` headers, `Needs:`, `Tier:`, `WaitsFor:`,
  `Type:`, and `Backoff:` metadata lines from embedded markdown.
  `InstantiateMolecule(ctx, mol, parent, opts)` at
  `molecule.go:270-296` implements a **format bridge**: if the
  molecule has child issues (new format), those children are used as
  templates via `instantiateFromChildren`
  (`molecule.go:299-365`); otherwise, steps are parsed from the
  Description field via `instantiateFromMarkdown`
  (`molecule.go:368-456`). Both paths create child issues with IDs
  `{parent.ID}.{step.Ref}`, wire inter-step dependencies, and
  expand `{{variable}}` template vars via `ExpandTemplateVars`
  (`molecule.go:227-240`). `ValidateMolecule`
  (`molecule.go:467-515`) validates the old format only (checks for
  empty refs, duplicates, unknown Needs, self-dependencies, cycles
  via DFS). `detectCycles` at `molecule.go:519-574` implements
  standard DFS cycle detection on the step graph. Detach/burn/squash
  operations call `audit.go`'s `DetachMoleculeWithAudit`.
- **`catalog.go` (199 lines)** — hierarchical molecule catalog loading.
- **`audit.go` (115 lines)** — audit log entries for detach/burn/squash
  operations. `DetachAuditEntry` struct (`audit.go:12-19`):
  `Timestamp`, `Operation` ("detach"/"burn"/"squash"),
  `PinnedBeadID`, `DetachedMolecule`, `DetachedBy`, `Reason`,
  `PreviousState`. `DetachOptions` (`audit.go:22-27`) carries the
  operation type, agent, and reason. `DetachMoleculeWithAudit`
  (`audit.go:32-81`) acquires a per-bead advisory file lock
  (`lockBead`), fetches the pinned bead, parses its attachment
  fields, writes a JSONL audit entry to `<beadsDir>/audit.log`
  via `LogDetachAudit` (`audit.go:86-115`), clears the attachment
  fields in the description, and returns the updated issue. Log
  errors are non-fatal (warning to stderr). The audit log is
  append-only JSONL, stored in the resolved `.beads` directory
  (follows redirect).
- **`fields.go` (1,057 lines)** — structured-field parsing for issue
  descriptions. The `ParseAgentFields`, `ParseHandoffFields`, and
  similar helpers turn `key: value` blocks in the description into typed
  Go structs. This is the layer that lets gt code pretend a description
  is schema-structured even though bd stores it as a free-form string.
- **`agent_ids.go` (472 lines)** — agent bead ID generation and parsing
  (e.g., `gt-<rig>-polecat-<name>`), prefix handling, name extraction.
- **`integration.go` (246 lines)** — integration-branch template
  handling for epic beads. Parses/writes `integration_branch:` and
  `base_branch:` fields in descriptions via
  `GetIntegrationBranchField`, `AddIntegrationBranchField`,
  `SanitizeBranchSegment`. Defines `IssueShower` and `BranchChecker`
  interfaces to avoid circular imports.
- **`stale_pid.go` (46 lines)** — helpers for detecting stale PID
  references in bead metadata.
- **`config_yaml.go` (149 lines)** — YAML config parsing for beads
  configuration shapes.

### Env and routing — the full stack

When a gt command needs to touch beads, the lookup chain is:

1. `workDir` — typically the caller's cwd or polecat worktree.
2. `FindTownRoot(workDir)` — walks up looking for `mayor/town.json`,
   returning the outermost match (`beads_types.go:41-55`). Lazily cached
   on the `Beads` struct via `townRootOnce`.
3. `ResolveBeadsDir(workDir)` — appends `.beads`, follows up to 3
   `redirect` files, detects circular redirects, returns the final
   path (`beads_redirect.go:26-70`).
4. `ResolveRoutingTarget(townRoot, beadID, fallback)` — extracts the
   bead ID prefix, reads `routes.jsonl` from the town root, finds the
   owning rig, returns that rig's resolved beads dir (or falls back).
   Used specifically for cross-rig `Show` calls (`beads_types.go:62-88`).
5. `DatabaseNameFromMetadata(beadsDir)` — reads `metadata.json`'s
   `dolt_database` field to construct the
   `BEADS_DOLT_SERVER_DATABASE=...` env for bd subprocesses
   (`database.go:11-33`).
6. `doltConnectionFromBeadsDir(beadsDir)` — reads
   `dolt-server.port` (per-directory override) and
   `metadata.json`'s `dolt_server_port` / `dolt_server_host` for Dolt
   connectivity (`beads.go:687-714`).
7. `overrideDoltEnvFromBeadsDir` strips inherited `BEADS_DOLT_*` values
   and replaces them with the authoritative per-directory values, so a
   parent shell's stale port never routes a command to the wrong server.
8. `translateDoltPort` mirrors `GT_DOLT_PORT`/`GT_DOLT_HOST` into the
   `BEADS_DOLT_*` namespace for bd's benefit (`beads.go:641-665`).

### Telemetry integration

Every single bd subprocess invocation records a `bd.call` event via
`telemetry.RecordBDCall(ctx, args, durationMs, err, stdout, stderr)`
(`beads.go:413-417`, `beads.go:490-494`). The content fields (stdout,
stderr) are only attached to the log record if
[`GT_LOG_BD_OUTPUT=true`](telemetry.md) — see the telemetry privacy
notes. Duration is always recorded as the `gastown.bd.duration_ms`
histogram; the counter is `gastown.bd.calls.total`, labeled with
`operation` (first arg) and `status="ok"|"error"`.

This is *the* reason the beads package imports telemetry so heavily:
bd calls are the single largest source of subprocess latency in gt and
are always worth measuring. If you see a slow gt command, the first
diagnostic is usually "how many bd.call events did it emit and how long
did each take?".

### Usage pattern

Short-lived command wanting speed:

```go
b := beads.New(workDir)
store, cleanup, err := b.OpenStore(ctx)
if err == nil {
    defer cleanup()
    b.SetStore(store)
}
issues, err := b.Ready()   // store path if store attached, else subprocess
```

Daemon / long-lived service:

```go
store, _ := beadsdk.OpenFromConfig(ctx, beadsDir)
b := beads.NewWithStore(workDir, store)
// reuse b for hundreds of calls; each bypasses ~600 ms of bd startup
```

Test isolation:

```go
b := beads.NewIsolatedWithPort(t.TempDir(), 33070)
b.Init("tt")
```

Cross-rig read:

```go
b := beads.New(gtRig)
issue, err := b.Show("hq-cv-abc123")   // routes into hq rig automatically
```

## Related wiki pages

- [gt](../binaries/gt.md)
- [internal/telemetry](telemetry.md) — every `run()` records a `bd.call`
  event with timing; opt-in content logging via `GT_LOG_BD_OUTPUT`.
- [internal/config](config.md) — role config feeding agent bead
  construction.
- [internal/util](util.md) — `SetDetachedProcessGroup` on every bd
  subprocess.
- [internal/session](session.md) — sessions close beads with
  `storeClose(reason, session, ids...)`.
- [internal/events](events.md) — activity-feed events; distinct from bd
  calls but often emitted alongside them.
- [internal/lock](lock.md) — general-purpose file locking that this
  package uses indirectly via `gofrs/flock` for per-bead `.locks/`
  directories.
- [go.mod](../files/go-mod.md) — `github.com/steveyegge/beads v0.63.3`
  is the upstream SDK this package wraps.
- [go-packages inventory](../inventory/go-packages.md)
- [gt bead](../commands/bead.md) — high-level bd wrapper command family.
- [gt close](../commands/close.md) — closes beads via this package's
  closure methods.
- [gt done](../commands/done.md) — writes completion metadata into agent
  beads via `beads_agent.go`.
- [gt ready](../commands/ready.md) — calls `(*Beads).Ready` and
  `ReadyForMol`.
- [gt cat](../commands/cat.md) — prints bead descriptions; relies on
  `fields.go` parsers for structured field display.
- [gt assign](../commands/assign.md) — uses `UpdateOptions.Assignee`.
- [gt compact](../commands/compact.md) — the compaction machinery
  interacts with beads state transitions.
- [gt cleanup](../commands/cleanup.md) — uses `IsProtectedBead` to decide
  what's safe to touch.

## Failure modes

### Precondition violations (what does it assume?)
- **bd binary on PATH:** Every subprocess invocation in `beads.go`
  calls `exec.LookPath("bd")` or shells out directly to `bd`.
  `ErrNotInstalled` at `beads.go:28` is the sentinel, but many
  internal helpers call `runBdCommand` which wraps `exec.Command`
  without a pre-check. **Present** — `ErrNotInstalled` is surfaced
  to callers, and the top-level `run` function checks `LookPath`
  before executing.
- **Store 30-second timeout:** `storeCtx()` at `store.go:78-80`
  creates a `context.WithTimeout(30s)` for all store operations. If
  Dolt is slow or overloaded, the operation fails with a context
  deadline error. **Present** — the timeout is explicit and errors
  propagate.

### Silent suppression (what errors are swallowed?)
- **Store close error:** `OpenStore` at `store.go:71-73` returns a
  cleanup function that calls `_ = store.Close()`, discarding any
  close error. **Absent** — if the store close fails (e.g.,
  connection dropped), the error is silently lost. In practice
  this is cleanup code and the main operation has already completed,
  but a dangling connection or uncommitted transaction could leak.
- **BdSupportsAllowStale probe error:** `beads.go:81` —
  `_ = cmd.Run()` discards the error from the `bd --allow-stale
  version` probe. **Present** — this is intentional capability
  detection; failure means "not supported."

## Troubleshooting

For diagnostic workflows involving this entity, see
[Investigating: data-plane failures](../workflows/investigations/data-plane.md).

## Detail tables

### Subprocess environment variables

| Var | Value | Why | Source |
|---|---|---|---|
| `BEADS_DIR` | Resolved beads directory | Prevents inherited values causing prefix mismatches (GH#803) | `beads.go:426-430` |
| `BEADS_DOLT_SERVER_PORT` | From target beads dir's `metadata.json` | Cross-rig operations reach correct Dolt server | `beads.go:687-714` |
| `BEADS_DOLT_SERVER_HOST` | From target beads dir | Cross-rig host resolution | `beads.go:687-714` |
| `BEADS_DOLT_SERVER_DATABASE` | From `database.go` | Correct database name per rig | `database.go:33` |
| OTel propagation env | From `telemetry.OTELEnvForSubprocess` | Subprocess telemetry joins gt's waterfall | `beads.go:440-441` |

### Subprocess flags injected

| Flag | Condition | Purpose | Source |
|---|---|---|---|
| `--allow-stale` | bd binary supports it (probed + cached) | Skip freshness checks for read operations | `beads.go:58-94` |
| `--flat` | `bd list --json` commands on bd v0.59+ | Produce actual JSON (not nested format) | `beads.go:117-137` |
| `--force` | Bead ID has > 1 hyphen | Bypass bd's prefix inference on multi-hyphen IDs | `force.go:11` |

### Redirect chain

| Mechanism | Max depth | Cycle detection | Source |
|---|---|---|---|
| `.beads/redirect` file follow | 3 | Yes (removes offending file) | `beads_redirect.go:381` |
| `routes.jsonl` prefix routing | n/a | Prefix-based, no recursion | `routes.go:360` |

### Stale PID cleanup

| Function | Purpose | Source |
|---|---|---|
| `CleanStaleDoltServerPID` | Remove PID files pointing to dead processes in `.beads/dolt/` | `stale_pid.go:20-46` |

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| `bd` | `version` | `--allow-stale` | capability probe | `beads.go:73` |
| `bd` | (dynamic) | `fullArgs...` | caller-provided | `beads.go:436` |
| `bd` | (dynamic retry) | `retryArgs...` | retry logic | `beads.go:460` |
| `bd` | (dynamic) | `fullArgs...` | caller-provided | `beads.go:498` |
| `bd` | `config set` | `types.custom <list>` | `constants.BeadsCustomTypes` | `beads_types.go:146` |
| `bd` | `config get` | `types.custom` | hardcoded | `beads_types.go:163` |
| `bd` | `config get` | `status.custom` | hardcoded | `beads_types.go:225` |
| `bd` | `config set` | `status.custom <merged>` | runtime-computed | `beads_types.go:258` |
| `bd` | `init` | `initArgs...` (prefix, force, etc.) | runtime config | `beads_types.go:345` |
| `bd` | `config set` | `issue_prefix <prefix>` | rig config | `beads_types.go:367` |
| `bd` | `migrate` | `--yes` | hardcoded | `beads_types.go:389` |
| `bd` | `migrate` | `--yes` (retry) | hardcoded | `beads_types.go:397` |
| `git` | `config` | `user.name` | hardcoded | `integration.go:173` |

Subprocess env vars and injected flags are documented in the [Detail tables](#detail-tables) above.

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| `<beads>/audit.log` | Append audit event JSON | Operation audit trail | `audit.go:96` |
| `<beads>/prime.md` | Prime content string | bd prime output | `beads.go:1674` |
| `.beads/redirect` | Redirect path | Beads directory redirect | `beads_redirect.go:348` |
| `<beads>/.types-sentinel` | Types list | Custom type registration sentinel | `beads_types.go:175` |
| `<beads>/.statuses-sentinel` | Statuses list | Custom status registration sentinel | `beads_types.go:272` |
| `<beads>/.catalog-*.tmp` | Catalog JSON (atomic) | Issue catalog cache | `catalog.go:154` |
| `<beads>/config.yaml` | YAML config content | Beads configuration | `config_yaml.go:96` |
| `<beads>/config.yaml` | Updated YAML content | Beads config update | `config_yaml.go:148` |
| `<beads>/.routes-*.tmp` | Routes JSONL (atomic) | Prefix routing table | `routes.go:126` |

## Notes / open questions

- This page is `status: partial` — coverage was expanded in Phase 6
  Batch 2 to include full reads of `beads_merge_slot.go`,
  `molecule.go`, `audit.go`, `beads_channel.go`, `beads_queue.go`,
  and `beads_group.go`. Earlier reads covered `beads.go` (partial),
  `store.go` (full), `database.go` (full), `force.go` (full),
  `status.go` (full), `beads_redirect.go` (partial),
  `beads_types.go` (partial), `beads_agent.go` (partial),
  `routes.go` / `integration.go` (headers). Remaining unread in
  full: `beads_escalation.go`, `beads_delegation.go`, `beads_rig.go`,
  `beads_dog.go`, `beads_sling_context.go`, `beads_mr.go`,
  `handoff.go`, `fields.go`, `agent_ids.go`, `config_yaml.go`,
  `stale_pid.go`, and the bulk of `beads.go` itself.
- The boundary between `internal/beads` and the upstream
  `github.com/steveyegge/beads` SDK is not static — as the SDK adds
  first-class support for more operations, the corresponding `store*`
  wrappers get added and subprocess paths become fallbacks. Pinning this
  page to beads SDK v0.63.3 makes it a dated snapshot by design.
- `--allow-stale` probing is cached per `bd` binary path. Tests that
  swap `bd` on `PATH` within a single process must call
  `ResetBdAllowStaleCacheForTest()` (`beads.go:44-49`) between swaps.
- The [`wisps`](../concepts/wisp.md) vs `issues` table split in bd
  v0.59+ means many methods have to query both tables for complete
  results (see `ListMergeRequests`). This is a recurring source of
  "why can't I see my MR in list output?" bugs.
- Multi-hyphen ID handling (`NeedsForceForID`) is a sharp edge that
  exists entirely because of bd's prefix-inference heuristic. Any new
  ID schemes should assume `--force` is required.
- There are several comments referencing `ZFC` principles ("Zero False
  Conclusions": don't parse stderr for decisions, transport errors to
  agents instead). `wrapError` deliberately keeps only two
  stderr-parsing exceptions: `ErrNotInstalled` and `ErrNotFound`.
- The daemon (`internal/daemon/convoy_manager.go` per comments in
  `store.go:1-7`) is the reference example for holding a persistent
  `beadsdk.Storage` across many operations.
- The package has no direct dependency on `internal/dolt` or any dolt
  client — all Dolt access flows through bd (subprocess) or beadsdk
  (SDK, which in turn opens its own Dolt connection).
