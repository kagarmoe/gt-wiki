---
title: Wiki validation round 3
type: drift
status: active
topic: gastown
created: 2026-04-17
updated: 2026-04-17
phase: 7
---

# Wiki validation round 3

Five open gastown issues scored against the wiki to measure how well it
supports issue investigation without reading gastown source.

## Summary

| Issue | Title | Score | Gap type |
|---|---|---|---|
| #3604 | Refinery stalls during branch consolidation — leaves dirty worktree, no PR | partial | conflict-recovery flow documented but silent-stall failure mode not flagged |
| #3570 | gt daemon doesn't clean up legacy tmux sockets on upgrade | partial | legacy socket cleanup documented on `gt down`; gap that daemon startup lacks it is implicit |
| #3565 | tmux keybindings use stale prefix pattern — miss rig-specific sessions | full | tmux package page documents `SetRigMenuBinding`, `isGTBindingCurrent`, and `sessionPrefixPattern` |
| #3563 | gt nudge fails with stale tmux pane ID on cross-rig crew targets | partial | `NudgePane`, `GT_PANE_ID`, and pane-ID caching documented; stale-pane-across-restart scenario not flagged |
| #3562 | Rig init can create duplicate Dolt databases causing beads to be invisible to mayor | full | rig package documents `InitBeads`, orphan `beads_<prefix>` cleanup, `verifyRigIdentity`, and `EnsureMetadata` |

**Scores: 2 full, 3 partial, 0 miss.**

---

### #3604 — Refinery stalls during branch consolidation — leaves dirty worktree, no PR

**Issue summary:** When multiple polecats complete work on the same rig and
the Refinery initiates branch consolidation, a merge conflict during
cherry-pick causes the process to terminate silently. No PR is created, the
worktree is left dirty with conflict markers, and no error or escalation is
generated. The documented behavior of "spawns FRESH polecat to re-implement"
fails to trigger.

**Wiki pages checked:**
- [internal/refinery](../packages/refinery.md) -- Engineer merge pipeline,
  conflict handling, `createConflictResolutionTaskForMR`
- [refinery role](../roles/refinery.md) -- behavioral overview, conflict
  recovery narrative
- [gt refinery](../commands/refinery.md) -- CLI surface

**What the wiki knows:** The wiki thoroughly documents the Refinery's
conflict-recovery mechanism. The `refinery` package page describes
`createConflictResolutionTaskForMR` (`engineer.go:1342+`) as the path that
"spawns a synthesis task bead to handle a conflicting MR." It documents the
`FailureType` enum including `conflict`, the `HandleMRInfoFailure` dispatch
(`engineer.go:1225+`), the `doMerge` pipeline, and the `CloseReason` of
`conflict`. The role page explicitly states: "On conflict, create a synthesis
task bead that spawns a fresh polecat to re-implement the work with the
conflict as context." The wiki also documents the `MRPhase` state machine
with terminal states `merged` and `rejected`.

**What the wiki doesn't know:** The wiki presents the conflict-recovery path
as working. It does not flag the failure mode described in #3604: that the
process can stall silently, leaving a dirty worktree without reaching the
conflict-resolution dispatch. The wiki documents what the code *intends* to
do but not this particular gap between the merge engine's exception handling
and the conflict-synthesis dispatch. There is no note about edge cases where
the cherry-pick operation itself fails in a way that doesn't trigger the
`FailureType.conflict` classification, causing the Engineer to exit without
escalation.

**Score:** partial

**Remediation:** Add a note to the `internal/refinery` package page's
"Notes / open questions" section flagging that the conflict-recovery dispatch
relies on the merge engine correctly classifying the failure as a `conflict`
`FailureType`. If the cherry-pick fails in an unclassified way (e.g., dirty
worktree from a prior interrupted operation), the Engineer may exit without
triggering synthesis or escalation. Reference #3604.

---

### #3570 — gt daemon doesn't clean up legacy tmux sockets on upgrade

**Issue summary:** Upgrading from v0.12.1 to v1.0.0 introduces a split-brain
condition. The version change modified tmux socket naming from the default
socket (`/tmp/tmux-<UID>/default`) to a hashed scheme
(`/tmp/tmux-<UID>/gt-<hash>`). The daemon's `ensureDeaconRunning` only checks
the new socket via `tmux.HasSession`, remaining unaware of legacy sessions.
Existing cleanup routines (`cleanupLegacyDefaultSocket`,
`cleanupLegacyBaseSocket`) exist in `gt down` but are unreachable during
typical upgrade scenarios where users restart the daemon without a full
shutdown cycle.

**Wiki pages checked:**
- [internal/daemon](../packages/daemon.md) -- daemon startup, heartbeat loop
- [gt down](../commands/down.md) -- Phase 4c legacy socket cleanup
- [internal/tmux](../packages/tmux.md) -- socket management, `SetDefaultSocket`,
  `SocketDir`, `killSplitBrainSession`
- [gt doctor](../commands/doctor.md) -- `tmux_socket_check.go` split-brain
  detection

**What the wiki knows:** The wiki documents the legacy socket cleanup in
detail on the `gt down` command page: Phase 4c describes
`cleanupLegacyDefaultSocket` and `cleanupLegacyBaseSocket` at
`down.go:348-371, 826-899`. The `tmux` package page documents
`killSplitBrainSession` (`tmux.go:771-787`) which kills stale same-named
sessions on other sockets during `NewSessionWithCommandAndEnv`. The `doctor`
page mentions `tmux_socket_check.go` for split-brain socket detection. The
daemon page documents the heartbeat loop's "ensure Deacon running" step and
"kill ghost sessions" step.

**What the wiki doesn't know:** The wiki does not explicitly flag the gap:
that legacy socket cleanup is only reachable via `gt down` (Phase 4c) and
`gt doctor`, but not via daemon startup or `gt up`. An investigator reading
the daemon page would see "ensure Deacon running" and "kill ghost sessions"
but would have to infer that these operate only on the current socket, not
legacy ones. The connection between the `gt down` cleanup helpers and the
daemon startup path is never made explicit as a gap.

**Score:** partial

**Remediation:** Add a note to the `internal/daemon` package page's
"Notes / open questions" section: "Legacy tmux socket cleanup
(`cleanupLegacyDefaultSocket`, `cleanupLegacyBaseSocket`) exists only in
the `gt down` teardown path. The daemon startup and heartbeat do not invoke
these helpers, meaning an upgrade that skips `gt down` can leave legacy
sessions running on old sockets, causing split-brain. See #3570."

---

### #3565 — tmux keybindings use stale prefix pattern — miss rig-specific sessions

**Issue summary:** Custom tmux keybindings fail in newly-added rig sessions
because `SetRigMenuBinding` does not call `isGTBindingCurrent()` to verify
the session prefix pattern is still valid. Unlike `SetAgentsBinding` and
`SetFeedBinding`, it only validates whether a binding exists via
`isGTBinding()` but not whether the pattern is current. Adding a new rig
does not refresh bindings on existing sessions.

**Wiki pages checked:**
- [internal/tmux](../packages/tmux.md) -- keybindings section,
  `SetRigMenuBinding`, `isGTBindingCurrent`, `sessionPrefixPattern`,
  `config.AllRigPrefixes`
- [internal/config](../packages/config.md) -- `AllRigPrefixes`

**What the wiki knows:** The tmux package page explicitly documents the
relevant functions: `SetRigMenuBinding` (`tmux.go:3565`), the smart binding
detection functions `isGTBinding`, `isGTBindingCurrent`, `getKeyBinding`
(`tmux.go:3307-3411`), and the `sessionPrefixPattern()` function
(`tmux.go:3426`) that builds a grep pattern from `config.AllRigPrefixes()`.
The page states that `isGTBindingCurrent` "reads existing bindings via
`list-keys` so repeated `ConfigureGasTownSession` calls are idempotent *and*
preserve the user's original fallback." The page documents both
`SetAgentsBinding` (`tmux.go:3542`) and `SetFeedBinding` (`tmux.go:3514`)
alongside `SetRigMenuBinding` in the same keybindings section. The config
package page documents `AllRigPrefixes`.

An investigator reading the wiki would see all three `Set*Binding` functions
listed and `isGTBindingCurrent` described as the staleness-check mechanism.
With that information, checking whether `SetRigMenuBinding` calls
`isGTBindingCurrent` (while the others do) is a natural next step. The wiki
provides the exact function names and line numbers needed to go straight to
the relevant code.

**Score:** full

**Remediation:** None needed. The wiki provides sufficient coverage of the
binding mechanism, the staleness-check function, and the relevant function
names to investigate this issue efficiently.

---

### #3563 — gt nudge fails with stale tmux pane ID on cross-rig crew targets

**Issue summary:** `gt nudge` fails with "can't find window: %164" when
targeting crew members on different rigs after their sessions restart. The
root cause is stale `GT_PANE_ID` caching in session environment variables:
pane IDs persist across crew stop/start cycles, and the nudge function's
global pane lookup can match a different session's pane sharing the same
numeric ID, causing the subsequent session-specific `send-keys` to fail.

**Wiki pages checked:**
- [internal/tmux](../packages/tmux.md) -- `NudgeSession`, `NudgePane`,
  `GetPaneID`, pane queries
- [internal/nudge](../packages/nudge.md) -- nudge queue, poller, delivery
  modes
- [internal/crew](../packages/crew.md) -- `GT_PANE_ID` storage
- [internal/deacon](../packages/deacon.md) -- `GT_PANE_ID` recording
- [internal/polecat](../packages/polecat.md) -- `GT_PANE_ID` for ZFC

**What the wiki knows:** The wiki documents the relevant mechanisms
thoroughly. The tmux package page describes `NudgePane` (`tmux.go:1735-1790`)
as "same protocol targeting a pane ID" and documents `GetPaneID`
(`tmux.go:2045`). The crew package page notes that session creation "stores
`GT_PANE_ID` for ZFC." The deacon and polecat package pages similarly
document `GT_PANE_ID` recording. The nudge package page explains the three
delivery modes (queue, wait-idle, immediate) and that immediate mode
"bypasses the queue and uses tmux `send-keys` directly." The tmux page's
nudge protocol section documents the full 8-step delivery sequence.

**What the wiki doesn't know:** The wiki does not flag the failure mode where
`GT_PANE_ID` becomes stale after a crew stop/start cycle. The pages document
that `GT_PANE_ID` is *set* during session creation but do not discuss what
happens when the pane is destroyed and recreated with a different ID while
the env var retains the old value. There is no note about cross-rig pane ID
scoping -- the wiki describes `NudgePane` as targeting "a pane ID" but does
not discuss whether the lookup is global or session-qualified. The proposed
fix (session-qualified pane verification via `display-message -t
<session>:<paneID>`) involves a concept not surfaced in the wiki.

**Score:** partial

**Remediation:** Add a note to the `internal/tmux` package page's
"Notes / open questions" section: "`GT_PANE_ID` is set once at session
creation and cached in the tmux session environment. If the session is
stopped and restarted, the pane ID changes but the cached value persists.
`NudgePane` uses global pane lookup, which can match a different session's
pane sharing the same numeric ID. Session-qualified pane targeting
(`display-message -t <session>:<paneID>`) would prevent cross-session
misdelivery. See #3563."

---

### #3562 — Rig init can create duplicate Dolt databases causing beads to be invisible to mayor

**Issue summary:** When a rig is configured with a name differing from its
prefix (e.g., rig `mobile_apps` with `prefix: "ma"`), the system may create
two separate Dolt databases. Beads routed through the rig use the
prefix-based database, while the mayor monitors the name-based one. Result:
pending work is invisible to the mayor, polecats remain idle despite queued
work, and no warnings are emitted.

**Wiki pages checked:**
- [internal/rig](../packages/rig.md) -- `InitBeads`, `verifyRigIdentity`,
  `AddRig` pipeline, `deriveBeadsPrefix`
- [internal/doltserver](../packages/doltserver.md) -- `InitRig`,
  `EnsureMetadata`, `RemoveDatabase`, `FindOrphanedDatabases`
- [rig concept](../concepts/rig.md) -- what a rig is
- [gt doctor](../commands/doctor.md) -- rig name mismatch check

**What the wiki knows:** The wiki provides comprehensive coverage of this
exact failure mode. The `internal/rig` package page documents `InitBeads`
(`manager.go:963-1098`) and explicitly notes that it "drops the orphan
`beads_<prefix>` database if it differs from `<rigName>` (gt-sv1h)." It
documents `verifyRigIdentity` (`manager.go:871-909`) as catching
`metadata.json` drift early (gas-tc4), noting that "a rig with a wrong
`dolt_database` value would silently work for read operations but fail to
spawn polecats." The `AddRig` pipeline steps 4 (derive beads prefix), 14
(`doltserver.InitRig` + `EnsureMetadata` + orphan cleanup), and 27
(`verifyRigIdentity`) are all documented in sequence. The page explicitly
warns: "Order matters: routes.jsonl before agent beads, bd init before
EnsureMetadata before issue_prefix." The doltserver page documents
`FindOrphanedDatabases`, `RemoveDatabase`, and `EnsureMetadata`. The doctor
page mentions "rig name mismatch" as a health check.

An investigator reading the wiki would find the `gt-sv1h` tag, the
`verifyRigIdentity` check, the orphan-database cleanup path, and the
explicit sequencing warning -- all directly relevant to diagnosing and
fixing #3562.

**Score:** full

**Remediation:** None needed. The wiki documents the exact code paths
involved in this issue, including the existing mitigation (`gt-sv1h` orphan
cleanup) and the identity verification check (`gas-tc4`). The issue appears
to describe a case where these guards are insufficient or not triggered,
but the wiki provides all the information needed to investigate.
