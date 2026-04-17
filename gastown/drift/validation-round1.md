---
title: Wiki validation round 1
type: drift
status: active
topic: gastown
created: 2026-04-17
updated: 2026-04-17
phase: 7
---

# Wiki validation round 1

Five open gastown issues scored against the wiki to measure how well it
supports issue investigation without reading gastown source.

## Summary

| Issue | Title | Score | Gap type |
|---|---|---|---|
| #3665 | Centralize cross-platform sleep command strings | miss | no `internal/shellcmd` coverage; no cross-platform command-string concept |
| #3661 | `bd` only has `--repo` flag, not `--rig` | partial | beads package page covers the call site file but not the flag-mapping detail |
| #3658 | gt polecat remove leaves zombie tmux sessions | partial | polecat package documents `Remove` but not its tmux session cleanup gap |
| #3653 | Dead code: HandlePolecatDone / HandlePolecatDoneFromBead | partial | witness package documents both code paths but doesn't flag the dead-code relationship |
| #3652 | witness and refinery Start() never create agent beads | partial | both Start() methods documented; agent bead gap visible by comparison with polecat |

**Scores: 0 full, 4 partial, 1 miss.**

---

### #3665 — Centralize cross-platform sleep command strings (sleep vs Windows timeout)

**Issue summary:** Gas Town hardcodes `sleep N` in tmux pane commands,
hooks, tests, and mock scripts. This works on Unix but fails on Windows
where `timeout /T N` is standard. The issue proposes a new
`internal/shellcmd` package to centralize portable sleep helpers.

**Wiki pages checked:**
- [internal/shell](../packages/shell.md) — shell integration (RC files), unrelated
- [internal/tmux](../packages/tmux.md) — tmux wrapper library
- [Makefile](../files/makefile.md) — mentions Windows powershell path and sleep calls
- [.goreleaser.yml](../files/goreleaser-yml.md) — mentions Windows build mode
- [Dockerfile](../files/dockerfile.md) — mentions `sleep infinity`
- [gastown/drift/gaps.md](gaps.md) — no mention of this gap

**What the wiki knows:** The wiki documents the `internal/shell` package
(RC file integration, not command-string helpers) and mentions
Windows-specific build paths in goreleaser and Makefile pages. It knows
about `sleep infinity` in container files. The `internal/tmux` package
page covers session creation but not the embedded shell command strings.

**What the wiki doesn't know:** There is no page for `internal/shellcmd`
(it doesn't exist yet — the issue is a feature request). More importantly,
the wiki has no concept page or cross-cutting note about platform-specific
shell command fragments scattered across the codebase. The
`internal/session` and `internal/tmux` pages don't catalog the hardcoded
`sleep` strings in tmux send-keys calls. No wiki content helps an
investigator find or enumerate the affected call sites.

**Score:** miss

**Remediation:** This is a feature-request issue (the package doesn't
exist yet), so the wiki correctly has no page for it. However, a
cross-cutting note in the `internal/tmux` or `internal/session` package
pages about platform assumptions in embedded shell commands would help
future investigators. After the feature lands, create
`gastown/packages/shellcmd.md`.

---

### #3661 — `bd` only has `--repo` flag, not `--rig`

**Issue summary:** When `gt` invokes `bd create`, it passes `--rig=<name>`,
but `bd` only recognizes `--repo=<name>`. This causes silent MR bead
creation failures in the `gt done` flow and the daemon's `bd list` path.
The bug is at `internal/beads/beads.go:1229-1231` and
`internal/daemon/daemon.go:2632`.

**Wiki pages checked:**
- [internal/beads](../packages/beads.md) — the 10,300-line library page
- [internal/daemon](../packages/daemon.md) — daemon heartbeat loop
- [gt done](../commands/done.md) — completion command
- [polecat-lifecycle](../workflows/polecat-lifecycle.md) — end-to-end flow
- [refinery role](../roles/refinery.md) — merge queue persona
- [gastown/drift/corrections.md](corrections.md) — upstream correction drafts

**What the wiki knows:** The `internal/beads` page documents
`beads_agent.go` in detail (753 lines, `CreateOrReopenAgentBead`, agent
fields including `rig`), but at the file-summary level — it doesn't
enumerate the CLI flag strings that `beads.go` passes to the `bd`
subprocess. The `polecat-lifecycle` workflow documents the `gt done` flow
including agent bead completion writes, but doesn't trace through to the
`bd create` invocation for MR beads. The `daemon` page covers the
heartbeat loop but not the specific `bd list` call at line 2632.

**What the wiki doesn't know:** The wiki doesn't document the exact CLI
flags that `internal/beads` passes when shelling out to `bd`. The
`--rig` vs `--repo` flag mismatch is invisible at the wiki's current
level of detail. An investigator reading the wiki would find the right
files (`beads.go`, `daemon.go`) but wouldn't see the bug without reading
the source.

**Score:** partial

**Remediation:** Expand the `internal/beads` package page's
`beads.go` section to document the `bd` subprocess invocation pattern,
including which flags are passed. Add a note about the `--rig`/`--repo`
mismatch once the fix lands (or as a drift finding now). The
`polecat-lifecycle` workflow could also trace the MR bead creation path
more explicitly.

---

### #3658 — gt polecat remove leaves zombie tmux sessions

**Issue summary:** `gt polecat remove -f` deletes worktrees but does not
kill the associated tmux sessions, leaving orphaned polecats visible in
`gt polecat list` as "zombie (no worktree)". The `gt deacon zombie-scan`
doesn't catch these because it looks for orphaned processes, not sessions
without worktrees.

**Wiki pages checked:**
- [internal/polecat](../packages/polecat.md) — polecat manager, `Remove`/`RemoveWithOptions`
- [gt polecat](../commands/polecat.md) — CLI surface, `runPolecatRemove`
- [polecat-lifecycle](../workflows/polecat-lifecycle.md) — lifecycle states including zombie
- [internal/witness](../packages/witness.md) — zombie detection
- [internal/doctor](../packages/doctor.md) — `zombie_check.go`
- [internal/deacon](../packages/deacon.md) — watchdog

**What the wiki knows:** The polecat package page documents
`Remove(name, force)` and `RemoveWithOptions` at
`manager.go:1026-1199`, noting that it "resets the agent bead BEFORE
filesystem ops, unassigns work beads, refuses to nuke if cwd is inside
the worktree, removes the worktree via `repoGit.WorktreeRemove`, releases
the name back to the pool." The `polecat-lifecycle` workflow documents
zombie states and the witness detection engine. The `doctor` page
documents `zombie_check.go` (tmux session alive but Claude process dead).

**What the wiki doesn't know:** The wiki's description of `Remove` lists
the steps (reset bead, unassign work, remove worktree, release name) but
does not mention whether `Remove` kills the tmux session. The omission
is the exact gap the bug exploits. The wiki also doesn't note that
`deacon zombie-scan` only detects process-without-session zombies, not
session-without-worktree zombies. An investigator would find the right
code locations but wouldn't learn from the wiki alone that the tmux
session kill step is missing.

**Score:** partial

**Remediation:** Expand the `Remove`/`RemoveWithOptions` section in
`gastown/packages/polecat.md` to explicitly list the tmux session
cleanup step (or note its absence). Add a "known limitations" note to
the `polecat-lifecycle` workflow about session-without-worktree zombies
not being caught by existing zombie detection. Once the fix lands,
update the `Remove` documentation to include the session kill step.

---

### #3653 — Dead code: HandlePolecatDone and HandlePolecatDoneFromBead never called in production

**Issue summary:** `HandlePolecatDone` and `HandlePolecatDoneFromBead` in
`internal/witness/handlers.go` are mail-based completion handlers that
exist only in test code. The actual production path uses
`DiscoverCompletions` (bead-sweep patrol). The issue proposes removing
the dead code.

**Wiki pages checked:**
- [internal/witness](../packages/witness.md) — handlers.go documentation
- [witness role](../roles/witness.md) — domain persona
- [gt patrol](../commands/patrol.md) — patrol command
- [internal/protocol](../packages/protocol.md) — protocol handlers

**What the wiki knows:** The witness package page documents both
`HandlePolecatDone` (`handlers.go:118+`) and `HandlePolecatDoneFromBead`
(`handlers.go:182+`) as "notable entry points." It also documents
`DiscoverCompletions` (`handlers.go:1700+`) with the explicit note that
it "sweeps beads for polecats whose `exit_type` field signals completion
even though the POLECAT_DONE mail was missed or dropped — the 'bead is
the source of truth' fallback." The "Notable design choices" section
states: "Two detection engines for one problem" and "Bead is the source
of truth for completion." The witness role page lists both
`DiscoverCompletions` and `HandlePolecatDone` as functions.

**What the wiki doesn't know:** The wiki presents both code paths as
active, co-existing production mechanisms. It doesn't flag that
`HandlePolecatDone` and `HandlePolecatDoneFromBead` are never called in
production — only in tests. The "Notable design choices" section frames
the two engines as complementary (zombie vs stalled), not as
active-vs-dead. An investigator reading the wiki would assume both paths
are live, which is the opposite of the truth. The wiki is not misleading
in what it says, but it is misleading in what it omits.

**Score:** partial

**Remediation:** Add a note to the `HandlePolecatDone` /
`HandlePolecatDoneFromBead` entries in `gastown/packages/witness.md`
indicating they are test-only / vestigial — the production completion
path is `DiscoverCompletions` via the patrol sweep. Update the "Notable
design choices" section to explicitly state that the mail-based handlers
are dead code superseded by the bead-sweep approach. If the dead code is
removed upstream, remove the entries from the wiki page.

---

### #3652 — bug: witness and refinery Start() never create agent beads

**Issue summary:** The witness and refinery `Start()` methods create tmux
sessions but never call `CreateOrReopenAgentBead`. Agent beads only
exist if previously created by `gt rig add` via `initAgentBeads()`. The
polecat `Start()` path correctly uses `createAgentBeadWithRetry`.

**Wiki pages checked:**
- [internal/witness](../packages/witness.md) — `Manager.Start()` documentation
- [internal/refinery](../packages/refinery.md) — `Manager.Start()` documentation
- [internal/polecat](../packages/polecat.md) — `createAgentBeadWithRetry` documentation
- [internal/beads](../packages/beads.md) — `CreateOrReopenAgentBead` documentation
- [internal/rig](../packages/rig.md) — `initAgentBeads`
- [polecat-lifecycle](../workflows/polecat-lifecycle.md) — agent bead creation in lifecycle

**What the wiki knows:** The witness `Start()` is documented at
`manager.go:110-278` with a detailed step list: zombie detection, role
config, runtime settings, tmux session creation, env vars, theme, nudge
poller, PID tracking. The refinery `Start()` is documented at
`manager.go:113-271` with a similar step list. The polecat package
documents `createAgentBeadWithRetry` at `manager.go:308-326` which
"calls `beads.CreateOrReopenAgentBead`." The `polecat-lifecycle` workflow
explicitly documents State 1 as "name allocated → worktree + agent bead
created."

**What the wiki doesn't know:** The wiki doesn't explicitly state that
witness and refinery `Start()` do NOT create agent beads. However, an
attentive reader comparing the three `Start()` descriptions would notice
the asymmetry: polecat's path mentions agent bead creation; witness and
refinery paths don't. The wiki has the signal, but it requires
cross-page comparison to extract. No single page says "witness/refinery
Start() assumes agent beads already exist" or flags this as a gap.

**Score:** partial

**Remediation:** Add a note to both `gastown/packages/witness.md` and
`gastown/packages/refinery.md` in their `Start()` sections explicitly
stating that `Start()` does not create or reopen agent beads — it assumes
they were created by `gt rig add` / `initAgentBeads()`. Reference the
polecat's `createAgentBeadWithRetry` as the contrast pattern. Once the
fix lands, update both pages to document the new agent bead
creation/reopen step.
