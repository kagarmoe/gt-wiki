---
title: Drift index
type: drift
status: complete
topic: gastown
created: 2026-04-15
updated: 2026-04-17
phase: 3-7
---

# Drift Index

Consolidated index of findings from Phase 3 (Drift audit, Batches 1-14), Phase 4 (Coverage/Completeness audit, Batches 1-3), Phase 5 (Audience classification of 111 commands), Phase 6 (Implementation), and Phase 7 (Correctness validation + wiki lint). Phase 3 audited 271 units (211 entity pages in Sweep 1 + 60 docs files in Sweep 2) and surfaced ~83 findings. Phase 4 audited 94 `status:partial` pages: 89 upgraded to `status:verified`, 5 confirmed incomplete. Phase 5 classified all 111 command pages by primary audience (user/agent/dev/internal) for Phase 6 prioritization. Phase 6 executed: 6 new package pages, 5 expanded, 4 subcommand gaps filled, 61 upstream correction drafts. Phase 7 validated 18/61 corrections (all valid), found 12 ambiguity findings, and ran a full wiki lint pass.

**How to read this index:**
- **Section 1** is the Phase 6 work-list: upstream corrections to the gastown source repo (code edits, docs PRs). Phase 5 (Audience classification) refines it; Phase 6 (Implementation) executes it.
- **Section 2** is the wiki self-maintenance log: our own synthesis errors, all fixed during Sweep 1. Preserved as an audit trail of what was wrong, not what is wrong.
- **Section 3** identifies cross-cutting meta-patterns that Phase 6 should batch as single PRs.
- **Section 4** lists missing entity pages (code entities with no corresponding wiki page).
- **Section 5** records subcommand coverage decisions (384 subcommands systematically classified).
- **Section 6** lists coverage gaps from Phase 4: pages whose wiki content is incomplete relative to their source code surface.
- **Section 7** summarizes the Phase 5 audience classification across all 111 command pages.
- **Section 8** summarizes Phase 6 implementation results: wiki-side fixes, upstream correction drafts, and remaining work.
- **Section 9** records Phase 7 correctness validation, ambiguity findings, and wiki lint results.

**Plan:** [../../.claude/plans/2026-04-14-phase3-drift.md](../../.claude/plans/2026-04-14-phase3-drift.md) (gitignored).

---

## Section 1: Gastown corrections list (Phase 6 work-list)

Rows requiring upstream fixes to gastown source (code edits, docs PRs). Grouped by fix tier.

### Fix tier: code (cobra-drift — edit Long text / in-code docstrings)

| Entity page | Category | Audience | Summary | Severity | Release position | PR reference |
|---|---|---|---|---|---|---|
| [account.md](../commands/account.md) | cobra-drift | user | Parent `Long` text enumerates 4 subcommands; 5 are registered (`switch` omitted) | wrong | in-release | |
| [activity.md](../commands/activity.md) | cobra-drift | dev | Refinery event type names in Cobra `Long` don't match event constants (`merged` not `merge_complete`, `merge_skipped` not `queue_processed`) | wrong | in-release | |
| [assign.md](../commands/assign.md) | cobra-drift | agent | `--force` flag is defined but never consumed by `runAssign` | wrong | in-release | |
| [boot.md](../commands/boot.md) | cobra-drift | agent | Boot is "a special dog" per Cobra `Long`, but is explicitly NOT a dog in the kennel (`manager.go:300`) | wrong | in-release | |
| [callbacks.md](../commands/callbacks.md) | cobra-drift | agent | `SLING_REQUEST` handler does not "spawn polecat for the work" — it only logs | wrong | in-release | |
| [changelog.md](../commands/changelog.md) | cobra-drift | dev | `--week` flag is defined but never consulted by `changelogSinceTime` | wrong | in-release | |
| [config.md](../commands/config.md) | cobra-drift | user | `config get` `Long` text omits `dolt.port` from its Supported keys list | wrong | in-release | |
| [crew.md](../commands/crew.md) | cobra-drift | user | `crewCmd.Long` "Commands:" lists 8; 11 visible subcommands registered (omits `status`, `rename`, `pristine`) | wrong | in-release | |
| [doctor.md](../commands/doctor.md) | cobra-drift | dev | Cobra `Long` text enumerates a selective subset of the ~80 checks `runDoctor` registers | wrong | in-release | |
| [down.md](../commands/down.md) | cobra-drift | user | `downCmd.Long` recommends `gt start` as complement; `gt up` is the actual complement | wrong | in-release | |
| [git-init.md](../commands/git-init.md) | cobra-drift | user | `gitInitCmd.Long` lists 3 steps; code performs 4 operations (omits branch-protection hook installation) | wrong | in-release | |
| [hooks.md](../commands/hooks.md) | cobra-drift | user | Parent `Long` text enumerates 8 subcommands; `hooks init` is wired but unlisted | wrong | in-release | |
| [init.md](../commands/init.md) | cobra-drift | user | `initCmd.Long` lists 4 directories; code creates 5 (and with different paths: `crew` omitted, `refinery/rig/` not `refinery/`, `mayor/rig/` not `mayor/`) | wrong | in-release | |
| [install.md](../commands/install.md) | cobra-drift | user | `installCmd.Long` HQ structure lists 3 items; install creates at least 5 directories (omits `deacon/`, `plugins/`) | wrong | in-release | |
| [install.md](../commands/install.md) | cobra-drift | user | `installCmd.Long` references non-existent `docs/hq.md` | wrong | in-release | |
| [mail.md](../commands/mail.md) | cobra-drift | agent | `mailCmd.Long` COMMANDS section lists 4 subcommands; 22 are registered (82% incomplete) | wrong | in-release | |
| [molecule.md](../commands/molecule.md) | cobra-drift | agent | Parent `Long` hand-maintained subcommand categories omit four step-group members and the `await-signal` shortcut | wrong | in-release | |
| [namepool.md](../commands/namepool.md) | cobra-drift | user | `namepoolCmd.Long` "Examples:" shows 6 operations; 8 subcommands registered (omits `create`, `delete`) | wrong | in-release | |
| [plugin.md](../commands/plugin.md) | cobra-drift | user | `plugin run` gate check is cooldown-only; other gate types silently fall through as open | wrong | in-release | |
| [quota.md](../commands/quota.md) | cobra-drift | user | `quotaCmd.Long` COMMANDS block lists 4 subcommands; 5 registered (omits `watch`) | wrong | in-release | |
| [reaper.md](../commands/reaper.md) | cobra-drift | dev | `reaperCmd.Long` "When run by a Dog" block lists 4 subcommands; 6 registered (omits `databases`, `run`) | wrong | in-release | |
| [repair.md](../commands/repair.md) | cobra-drift | dev | Cobra `Long` text lists six repair targets; `runRepair` registers two checks | wrong | in-release | |
| [shell.md](../commands/shell.md) | cobra-drift | user | `shell install` silently re-enables Gas Town via `state.Enable(Version)` without mentioning it in `Long` text | wrong | in-release | |
| [status.md](../commands/status.md) | cobra-drift | user | `--fast` help text understates what it skips (omits hooks, MQ, per-agent mail — not just mail) | wrong | in-release | |
| [tap.md](../commands/tap.md) | cobra-drift | agent | Long text omits two implemented subcommands (`list`, `polecat-stop-check`) | wrong | in-release | |
| [tap.md](../commands/tap.md) | cobra-drift | agent | Long text advertises three unimplemented subcommands (`audit`, `inject`, `check` — all `[planned]`) | wrong | in-release | |
| [uninstall.md](../commands/uninstall.md) | cobra-drift | user | `--workspace` is a hardcoded two-location scan (`~/gt`, `~/gastown`); silently no-ops for any other workspace path | wrong | in-release | |
| [warrant.md](../commands/warrant.md) | cobra-drift | dev | Long text hardcodes `~/gt/warrants/` but code uses `<townRoot>/warrants/` | wrong | in-release | |
| [witness.md](../commands/witness.md) | cobra-drift | agent | `witness start --foreground` is advertised as functional in `Long` + flag description, but is a no-op notice | wrong | in-release | |
| [wrappers.md](../packages/wrappers.md) | cobra-drift | — | ABOUTME header omits `gt-gemini` wrapper (lists 2, code installs 3) | wrong | in-release | |

### Fix tier: docs (upstream docs PR)

| Entity page | Category | Audience | Summary | Severity | Release position | PR reference |
|---|---|---|---|---|---|---|
| [done.md](../commands/done.md) | drift | agent | `docs/CLEANUP.md` claims `gt done` self-nukes the worktree and kills its session, but code transitions to IDLE with sandbox preserved | wrong | in-release | |
| [install.md](../commands/install.md) | drift | user | `docs/INSTALLING.md` shows `rigs/` in the `gt install` tree, but rigs are top-level directories | wrong | in-release | |
| [convoy.md](../concepts/convoy.md) | drift | — | Event-driven feeder misattributed to `operations.go` in `docs/skills/convoy/SKILL.md`; actually `convoy_manager.go:186-247` | wrong | in-release | |
| [convoy.md](../concepts/convoy.md) | drift | — | `docs/concepts/convoy.md` omits `staged_ready` and `staged_warnings` states (shows only `open` and `closed`) | wrong | in-release | |
| [identity.md](../concepts/identity.md) | drift | — | `docs/concepts/identity.md` claims `GIT_AUTHOR_NAME="gastown/polecats/toast"` (full BD_ACTOR path); code sets it to agent name only (`toast`) | wrong | in-release | |
| [go-mod.md](../files/go-mod.md) | drift | — | Go version disagrees across three build paths: `go.mod` 1.25.8, Dockerfile 1.25.6, Dockerfile.e2e 1.26 | wrong | in-release | |
| [goreleaser-yml.md](../files/goreleaser-yml.md) | drift | — | Release header advertises `brew install gastown` but no `brews:` block exists in `.goreleaser.yml` | wrong | in-release | |
| [health.md](../packages/health.md) | drift | — | Package doc claims Doctor Dog shares this package; only `gt health` imports it | wrong | in-release | |
| [telemetry.md](../packages/telemetry.md) | drift | — | `docs/design/otel/otel-architecture.md` claims `RecordMol*`/`RecordBeadCreate`/`RecordAgentInstantiate` don't exist; they do. Says "18 metric instruments"; code has 24 | wrong | in-release | |

### Fix tier: docs (docs-only — no wiki page annotation)

These findings were surfaced during Sweep 2 (docs-file audit) and affect only upstream docs files. No wiki entity page was annotated because the wiki already has the correct information.

| Docs file | Category | Audience | Summary | Severity | Release position | Batch | PR reference |
|---|---|---|---|---|---|---|---|
| `docs/design/persistent-polecat-pool.md` | drift | — | Says `gt polecat pool init <rig>` (space-separated); code registers as `pool-init` (hyphenated) | wrong | in-release | 11c | |
| `docs/HOOKS.md` | drift | — | Known Gap #2 claims `gt tap guard dangerous-command` doesn't exist; `tap_guard_dangerous.go` exists at HEAD and v1.0.0 | wrong | in-release | 10e | |
| `docs/design/directives-and-overlays.md` | drift | — | "CLI commands are being added" label stale; `gt directive show/edit/list` and `gt formula overlay show/edit/list` all exist | wrong | in-release | 11e | |
| `docs/design/architecture.md` | drift | — | MQ implementation phases table says GatesParallel "In progress" and batch-then-bisect "Blocked by Phase 1"; both fully implemented | wrong | in-release | 11f | |
| `docs/design/architecture.md` | drift | — | Dead reference to `[Watchdog Chain](watchdog-chain.md)`; no such file exists (closest: `dog-infrastructure.md`) | wrong | in-release | 11f | |
| `docs/design/property-layers.md` | drift | — | Dead reference to `[Watchdog Chain](watchdog-chain.md)` | wrong | in-release | 11g | |
| `docs/design/scheduler.md` | drift | — | Dead reference to `[Watchdog Chain](watchdog-chain.md)` | wrong | in-release | 11h | |
| `docs/design/polecat-self-managed-completion.md` | drift | — | Header says "Status: Design proposal" but all three migration phases are shipped in code | wrong | in-release | 11i | |
| `docs/design/mail-protocol.md` | drift | — | Stale Polecat->Witness->Refinery completion flow; polecat now nudges refinery directly (self-managed completion) | wrong | in-release | 11l | |
| `docs/design/polecat-lifecycle-patrol.md` | drift | — | Stale cleanup pipeline flow (same as mail-protocol.md); witness no longer required checkpoint | wrong | in-release | 11o | |

### Fix tier: preserve-as-vision (implementation-status: unbuilt/partial)

These findings are NOT bugs. They describe aspirational or partially-implemented features. Phase 6 action: add "not yet implemented" callouts where the docs don't already have them.

| Source | Category | Audience | Summary | Severity | Release position | Batch | PR reference |
|---|---|---|---|---|---|---|---|
| [witness.md](../commands/witness.md) | implementation-status: vestigial | agent | `--foreground` flag kept as runtime notice; patrol loop moved to `mol-witness-patrol` molecule | wrong | in-release | 1d | |
| [keepalive.md](../packages/keepalive.md) | implementation-status: partial | — | Package fully implemented (Touch/Read/Age API) but zero importers in codebase | incomplete | in-release | 2b | |
| `docs/design/dog-infrastructure.md` | implementation-status: unbuilt | — | Dog Pool Architecture (ShutdownDanceState, DogPool struct, warrant queue, `gt dog dances/warrants/pool status`) — no code | wrong | in-release | 11j | |
| `docs/design/model-aware-molecules.md` | implementation-status: unbuilt | — | Entire model-aware molecules feature (per-step model constraints, subscription routing, OpenRouter pricing) — no code; `internal/models/` doesn't exist | wrong | in-release | 11m | |
| `docs/design/ledger-export-triggers.md` | implementation-status: unbuilt | — | Entire ledger export trigger system (bead closure triggers, convoy triggers, skill derivation engine, HOP economy) — no code | wrong | in-release | 11n | |
| `docs/design/mol-mall-design.md` | implementation-status: unbuilt | — | Mol Mall registry system (public/private/federated registries, HOP URI scheme, formula publishing) — no code beyond embedded formulas | wrong | in-release | 12a | |
| `docs/design/witness-at-team-lead.md` | implementation-status: unbuilt | — | Witness-as-AT-team-lead architecture (delegate mode, AT teammate spawning) — no code; depends on Claude Code Agent Teams | wrong | in-release | 12b | |
| `docs/design/sandboxed-polecat-execution.md` | implementation-status: unbuilt | — | Sandboxed execution (exitbox container backend, daytona cloud backend, SandboxAdapter interface) — no code; all execution is local tmux | wrong | in-release | 12d | |
| `docs/design/factory-worker-api.md` | implementation-status: unbuilt | — | Factory Worker API (7 structured endpoints) — no code; all agent interaction remains tmux-based (28 touch points) | wrong | in-release | 12f | |
| `docs/design/plugin-system.md` | implementation-status: partial | — | Core dispatch shipped (`gt dog dispatch --plugin`, `gt plugin` CLI); design doc status label stale ("not yet implemented"). Advanced features (versioning, marketplace) not implemented | incomplete | in-release | 12c | |
| `docs/design/federation.md` | implementation-status: partial | — | Federation infrastructure exists (Dolt remotes, wasteland commands); core features (HOP URI, cross-workspace queries, sovereignty model) not implemented | incomplete | in-release | 12e | |
| `docs/design/formula-resolution.md` | implementation-status: partial | — | Tier 2 (town/rig overrides) + Tier 3 (embedded) work; Tier 1 (project-committed formulas) + Mol Mall integration not implemented | incomplete | in-release | 12g | |

---

## Section 2: Wiki self-maintenance

Rows that we fixed ourselves (our own Phase 2 synthesis errors). All were fixed inline during Sweep 1 — this section records what WAS wrong, not what IS wrong.

| Entity page | Category | Summary | Phase 2 root cause | Status |
|---|---|---|---|---|
| [agents.md](../commands/agents.md) | wiki-stale | Phase 2 said 4 subcommands; missed `agent_state.go` sibling wiring a 5th (`state`) | phase-2-incomplete | fixed in Sweep 1 |
| [directive.md](../commands/directive.md) | wiki-stale | Phase 2 claimed subcommands were unwired; sibling files `directive_show.go`, `directive_edit.go`, `directive_list.go` wire them | phase-2-incomplete | fixed in Sweep 1 |
| [dolt.md](../commands/dolt.md) | wiki-stale | Phase 2 listed 19 subcommands from `dolt.go` only; 2 more in sibling files (`dolt_flatten.go`, etc.) | phase-2-incomplete | fixed in Sweep 1 |
| [hooks.md](../commands/hooks.md) | wiki-stale | Phase 2 claimed subcommand wiring was missing; all 9 subcommands wired by sibling `hooks_*.go` files | phase-2-incomplete | fixed in Sweep 1 |
| [init.md](../commands/init.md) | wiki-stale | Phase 2 repeated Long text's incorrect 4-directory list; trusted Long instead of reading `rig.AgentDirs` | phase-2-incomplete | fixed in Sweep 1 |
| [mail.md](../commands/mail.md) | wiki-stale | Phase 2 listed 17 subcommands from `mail.go` only; missed 5 sibling-file groups (`channel`, `directory`, `group`, `hook`, `queue`) | phase-2-incomplete | fixed in Sweep 1 |
| [molecule.md](../commands/molecule.md) | wiki-stale | Phase 2 under-counted subcommand set; missed sibling-file registrations | phase-2-incomplete | fixed in Sweep 1 |
| [mq.md](../commands/mq.md) | wiki-stale | Phase 2 missed `mq next` subcommand wired in `mq_next.go` sibling file | phase-2-incomplete | fixed in Sweep 1 |
| [polecat.md](../commands/polecat.md) | wiki-stale | Phase 2 incorrectly claimed `polecat_spawn.go` and `polecat_cycle.go` register additional subcommands; they don't | phase-2-incomplete | fixed in Sweep 1 |
| [session.md](../commands/session.md) | wiki-stale | Phase 2 characterized `session inject` as "explicitly deprecated in favor of nudge"; neither Long text nor code says this | phase-2-incomplete | fixed in Sweep 1 |
| [tap.md](../commands/tap.md) | wiki-stale | Phase 2 missed sibling files `tap_list.go`, `tap_polecat_stop.go` | phase-2-incomplete | fixed in Sweep 1 |
| [wl.md](../commands/wl.md) | wiki-stale | Phase 2 claimed only `join` wired; 10 sibling files wire 10 more subcommands | phase-2-incomplete | fixed in Sweep 1 |
| [directive.md](../concepts/directive.md) | wiki-stale | Phase 2 concept page described CLI as "parent-only stub" with no wired subcommands | phase-2-incomplete | fixed in Sweep 1 |
| [docker-entrypoint.md](../files/docker-entrypoint.md) | wiki-stale | Line count 23 -> 22 | phase-2-incomplete | fixed in Sweep 1 |
| [go-mod.md](../files/go-mod.md) | wiki-stale | Require count 31 -> 30 | phase-2-incomplete | fixed in Sweep 1 |
| [docs-tree.md](../inventory/docs-tree.md) | wiki-stale | Subdirectory count 10 -> 11 (missed `skills/convoy/`) | phase-2-incomplete | fixed in Sweep 1 |
| [beads.md](../packages/beads.md) | wiki-stale | Wiki body incomplete | phase-2-incomplete | fixed in Sweep 1 |
| [events.md](../packages/events.md) | wiki-stale | Phase 2 event-type count wrong (said "21 event types + 21"; actual count differs) | phase-2-incomplete | fixed in Sweep 1 |
| [keepalive.md](../packages/keepalive.md) | wiki-stale | Phase 2 coverage incomplete | phase-2-incomplete | fixed in Sweep 1 |
| [util.md](../packages/util.md) | wiki-stale | Wiki body incomplete | phase-2-incomplete | fixed in Sweep 1 |
| [polecat.md](../roles/polecat.md) | wiki-stale | Claimed witness and refinery role pages were "pending"; both pages already existed | phase-2-incomplete | fixed in Sweep 1 |

---

## Section 3: Phase 6 meta-patterns

Cross-cutting patterns that Phase 6 should batch as single PRs rather than per-finding fixes.

### 1. Hand-maintained Long text enumerations

10+ commands have hand-maintained subcommand/capability lists in their `Long` text that omit entries. The omission pattern is uniform: parent `Long` was written at one point, sibling files added subcommands later, and the `Long` text was never updated. A single PR pattern could:
- Auto-generate the "Commands:" / "Subcommands:" blocks from the registered subcommand set, or
- Remove the hand-maintained lists and rely on Cobra's auto-generated `Available Commands:` block, or
- Add a linter / test that compares Long text enumeration count against actual registered count.

**Affected commands:** account, crew, doctor, hooks, mail, molecule, namepool, quota, reaper, tap.

### 2. Stale completion flow diagrams

3+ design docs show the old Polecat -> Witness -> Refinery completion flow. Since self-managed completion shipped (gt-1qlg), the polecat now nudges refinery directly and sets `agent_state=idle` directly. The witness is kept for observability, not as a required relay.

**Affected docs:** `docs/design/mail-protocol.md`, `docs/design/polecat-lifecycle-patrol.md`, `docs/design/polecat-self-managed-completion.md` (status header only).

### 3. Dead doc references

3 design docs reference `[Watchdog Chain](watchdog-chain.md)` but no `watchdog-chain.md` exists. The closest equivalent is `dog-infrastructure.md`. A single find-and-replace PR fixes all three.

**Affected docs:** `docs/design/architecture.md`, `docs/design/property-layers.md`, `docs/design/scheduler.md`.

### 4. Semantic cross-command references

The `downCmd.Long` recommends `gt start` as complement when `gt up` is the actual inverse. Similar "see also" or "use X instead" patterns in Long text should be audited as a group to verify that cross-command recommendations match actual behavior.

---

## Section 4: Missing entity pages (severity: missing)

Code entities with no corresponding wiki page. Surfaced by Batch 14 Sweep G1 (systematic code-to-wiki gap enumeration). Full detail in [gaps.md](gaps.md).

### Fix tier: wiki (we write the page)

| Code path | Description | Severity | Fix tier | Related bead | Notes |
|---|---|---|---|---|---|
| `internal/agent` | Shared types and StateManager generic for all agents (witness, refinery, deacon); 62 lines | missing | wiki | | Own API surface, imported by multiple agents |
| `internal/agent/provider` | JSON-RPC provider types for LLM agent communication (roles, messages, tool definitions); 799 lines | missing | wiki | | Core agent protocol layer |
| `internal/boot` | Boot watchdog package — daemon entry point for Deacon triage decisions; 237 lines | missing | wiki | | Distinct from `gt boot` command page; package has its own daemon-tick logic |
| `internal/checkpoint` | Session checkpointing for crash recovery (save/load polecat state); 350 lines | missing | wiki | | Own file format (.polecat-checkpoint.json), recovery API |
| `internal/connection` | Address parsing for agent/rig/polecat addresses (`[machine:]rig[/polecat]`); 689 lines | missing | wiki | wiki-7u4 | Networking/routing abstraction (bitbucket bead also covers related gap) |
| `internal/proxy` | mTLS CA management and proxy server for sandboxed polecat execution; 1,363 lines | missing | wiki | | Distinct from the `gt-proxy-*` binary pages |

### Docs file coverage gaps (from bead wiki-w71)

Six new docs files identified during Sweep 2 (Batch 11) that also lack wiki entity page coverage. These were filed under bead wiki-w71 and are listed here for completeness:

| Docs file | Status | Bead |
|---|---|---|
| 6 new `docs/` files from Sweep 2 | Coverage pending | wiki-w71 |

*Note: Individual docs file paths are tracked in the wiki-w71 bead. They are upstream documentation files that need wiki synthesis pages, not code packages.*

---

## Section 5: Subcommand coverage decisions

Systematic classification of all 384 non-root subcommands across 62 parent commands. Surfaced by Batch 14 Sweep G2. Full per-parent tables in [gaps.md](gaps.md).

### Coverage summary

| Metric | Count |
|---|---|
| Total non-root subcommands | 384 |
| Covered in parent wiki page | 380 |
| Gap: needs parent expansion | 4 |
| Gap: needs own page | 0 |
| Deliberately excluded | 0 |

**Coverage rate: 98.9%** (380 of 384).

### Subcommand gaps (4 findings)

| Parent command | Missing subcommand(s) | Count | Classification | Action |
|---|---|---|---|---|
| `gt formula overlay` | `show`, `edit`, `list` | 3 | gap: needs parent expansion | Expand formula.md with Overlay section |
| `gt patrol` | `scan` | 1 | gap: needs parent expansion | Expand patrol.md with scan subcommand |

### Deliberately excluded entities (9 total)

Entities systematically evaluated and excluded from wiki coverage with rationale:

| Entity | Rationale |
|---|---|
| `internal/cmd` | CLI wiring layer (239 files, 96,500 lines); every command already has its own wiki page under `gastown/commands/` |
| `internal/scheduler/capacity` | Sub-package; parent `scheduler.md` already covers the scheduler |
| `internal/templates/commands` | Helper sub-package; parent `templates.md` covers it |
| `internal/tui/convoy` | UI sub-package; parent `tui.md` covers both TUI sub-packages |
| `internal/tui/feed` | UI sub-package; parent `tui.md` covers both TUI sub-packages |
| `codecov.yml` | CI config (108 lines); no runtime behavior, self-documenting YAML |
| `renovate.json` | Dependency bot config (74 lines); no runtime behavior |
| `go.sum` | Auto-generated dependency checksums (322 lines); never manually edited |
| `.gitignore` | Standard gitignore (86 lines); self-documenting |

---

## Section 6: Coverage gaps (severity: incomplete)

Wiki pages whose content is incomplete relative to their source code surface. Surfaced by Phase 4 (Coverage/Completeness audit, Batches 1-2). These are not drift findings (the wiki doesn't contradict the code); they are coverage gaps where significant source code subsystems are acknowledged but not grounded.

**Phase 4 overview:** 94 `status:partial` pages audited (50 commands + 22 packages + 8 roles + 7 concepts + 2 workflows + 3 binaries + 1 inventory + 1 plugins). 89 upgraded to `status:verified` (Phase 2's coverage was adequate). 5 confirmed incomplete (1 command, 4 packages).

### Fix tier: wiki (we write the page content)

| Entity page | Category | Gap description | Severity | Fix tier |
|---|---|---|---|---|
| [formula.md](../commands/formula.md) | incomplete | Missing entire `overlay` subcommand tree (4 source files, 347 lines): `formula_overlay.go`, `formula_overlay_edit.go`, `formula_overlay_list.go`, `formula_overlay_show.go` | incomplete | wiki |
| [beads.md](../packages/beads.md) | incomplete | 28 files / 10,300 lines; ~10 files read in full. Merge-slot internals, molecule detach audit schema, delegation metadata encoding, channel/queue/group bead lifecycles not grounded | incomplete | wiki |
| [daemon.md](../packages/daemon.md) | incomplete | 33 files / 23,025 lines; 4 large files (~130KB total: `dolt.go`, `compactor_dog.go`, `convoy_manager.go`, `jsonl_git_backup.go`) acknowledged but not grounded | incomplete | wiki |
| [doltserver.md](../packages/doltserver.md) | incomplete | WLCommons subsystem (wanted-list, stamps, badges query surfaces) + DoltHub remote management API not covered | incomplete | wiki |
| [polecat.md](../packages/polecat.md) | incomplete | `session_manager.go` startup builder, pane monitoring loop, session recovery flow, and `AddWithOptions` sandbox container-creation flow not grounded | incomplete | wiki |

**Pattern:** The 5 incomplete pages share a common root cause: Phase 2 read the primary files and described the architecture correctly, but acknowledged large subsystem files it didn't fully ground. These are content-writing work items, not corrections.

---

## Summary statistics

### Phase 3 (Drift audit)

| Metric | Count |
|---|---|
| **Total units audited** | 271 (211 entity pages + 60 docs files) |
| **Section 1 rows (upstream corrections)** | 52 |
| **Section 2 rows (wiki self-maintenance)** | 21 |
| **Section 4 rows (missing entity pages)** | 6 (+6 docs files via wiki-w71) |
| **Section 5 (subcommand gaps)** | 4 |
| **Total drift findings (Sections 1-3)** | 73 |
| **Total gap findings (Sections 4-5)** | 10 |
| **Grand total findings** | ~83 |
| **Deliberately excluded entities** | 9 |

### Phase 4 (Coverage/Completeness audit)

| Metric | Count |
|---|---|
| **Pages audited** | 94 (all `status:partial`) |
| **Upgraded to verified** | 89 |
| **Confirmed incomplete (Section 6)** | 5 |
| **Coverage rate** | 94.7% adequate (89 of 94) |

### Section 1 breakdown by category

| Category | Fix tier | Count |
|---|---|---|
| cobra-drift | code | 30 |
| drift (wiki-annotated) | docs | 9 |
| drift (docs-only) | docs | 10 |
| implementation-status: unbuilt | preserve-as-vision | 7 |
| implementation-status: partial | preserve-as-vision | 3 |
| implementation-status: vestigial | code or docs | 1 |
| **Section 1 total** | | **60** |

*Note: Some entity pages carry multiple findings (e.g., install.md has 3, tap.md has 2, witness.md has 2). The row count (52) differs from the finding count (60) because 8 entity pages carry 2+ findings, but finding-level detail is preserved in the per-row summaries above.*

**Correction: 52 rows across the four Section 1 sub-tables. 60 findings when counting multi-finding pages at full granularity.**

### Section 2 breakdown

| Sub-type | Count |
|---|---|
| wiki-stale (phase-2-incomplete) | 21 |
| wiki-stale (churn) | 0 |
| **Section 2 total** | **21** |

All 21 wiki-stale findings were `phase-2-incomplete` — Phase 2's mapping methodology had blind spots (primarily: reading parent `.go` files without checking sibling files for `AddCommand` registrations). Zero churn-induced drift was found, meaning gastown's codebase has been stable since Phase 2.

### Release position

All findings are `in-release` (present at v1.0.0). Zero `post-release` findings.

---

## Section 7: Audience classification (Phase 5)

Phase 5 classified all 111 command pages by primary audience. The Audience column in Section 1 enables Phase 6 to prioritize: user-facing drift is fixed first, then agent, then dev, then internal.

| Audience | Count | Examples |
|---|---|---|
| user | 33 | install, up, down, config, status, crew, rig, init, daemon, shell, version, ... |
| agent | 46 | sling, done, nudge, mail, polecat, mayor, convoy, molecule, hook, broadcast, ... |
| dev | 28 | doctor, repair, audit, stale, health, vitals, costs, metrics, changelog, ... |
| internal | 4 | agent-log, nudge-poller, proxy-subcmds, status-line |

### Section 1 audience breakdown

Drift findings by audience (for Phase 6 prioritization):

| Audience | Findings in Section 1 | Priority |
|---|---|---|
| user | 18 | Highest — operators encounter these directly |
| agent | 11 | High — agents see stale help text |
| dev | 7 | Medium — maintainers can work around inaccuracies |
| internal | 0 | Low — hidden commands, rarely viewed |
| — (non-command) | 24 | Varies by entity type |

### Edge-case decisions

| Command | Competing signals | Primary audience | Rationale |
|---|---|---|---|
| status | user + agent (PolecatSafe) + dev (GroupDiag) | user | Main status command operators use |
| prime | agent (PolecatSafe) + dev (GroupDiag) | agent | Context-building command agents run at startup |
| version | user + agent (PolecatSafe) + dev (GroupDiag) | user | Operators check version |
| crew, rig | user (GroupWorkspace) + hidden subcommands | user | Hidden subs are implementation detail |
| heartbeat | dev (GroupDiag) + agent infra | agent | Polecats invoke this, not humans |
| info | dev (GroupDiag) | user | `--whats-new` is user-facing |
| thanks | dev (GroupDiag) | user | Credits page for humans |
| dashboard | dev (GroupDiag) | dev | Convoy tracking TUI for maintainers |
| scheduler | agent (GroupAgents) | agent | Daemon-facing capacity control |
| tap | agent (PolecatSafe) + hook handlers | agent | Hook handlers agents invoke |

---

## Section 8: Phase 6 Implementation summary

### Wiki-side fixes (completed)
- 6 new package pages written (agent, agent/provider, boot, checkpoint, connection, proxy)
- 5 incomplete pages expanded (formula, beads, daemon, doltserver, polecat)
- 4 subcommand gaps filled (formula overlay tree, patrol scan)

### Upstream correction drafts (pending Kimberly's PR submission)
- 61 corrections drafted in [corrections.md](corrections.md)
- Grouped by: enumerations (11), stale claims (19), docs (19), status callouts (12)

### Remaining wiki work
- beads.md, daemon.md, doltserver.md, polecat.md still status:partial (large packages)
- bitbucket package page (bead wiki-7u4) — deferred to release sync
- 6 new docs file coverage (bead wiki-w71) — deferred to release sync

---

## Section 9: Phase 7 Correctness summary

### Correction validation
- 18/61 corrections spot-checked against gastown HEAD (`9f962c4a`), all valid
- Verbatim quotes match, code citations accurate, suggested replacements correct
- No line-number drift from recent reaper commits
- Full validation deferred to PR submission time

### Ambiguity findings
- 12 severity:ambiguous findings (see [corrections-audit.md](corrections-audit.md))
- 4 are corrections.md gaps (persistent polecat model + reference.md not updated)
- 8 are docs-wording ambiguity (vague claims, dead links)
- Notable: reference.md lifecycle section (lines 258-259) and env table (line 284) have stale polecat claims not in corrections.md

### Wiki lint
- **Orphan pages:** 0 — all 6 new Phase 6 package pages have inbound links (index.md + cross-references)
- **Broken cross-links:** 0 — all relative links in Phase 6 pages resolve correctly
- **Stale frontmatter dates:** 82 pages had `updated:` dates older than their last git commit; all fixed
- **Phase 6 page quality:** all 6 new package pages have proper frontmatter, file:line citations, cross-links, and `## What it actually does` sections
- **Command page index coverage:** 73 command pages not individually listed in index.md, but all are linked from gastown/commands/README.md (by design — index.md delegates to the commands sub-index)

---

## Section 10: Phase 8 Failure Mode Analysis

### Coverage
- 212 entity pages audited for failure modes
- 83 pages with failure mode findings (39%)
- 129 pages with [none] (61%)
- All pages stamped with phase8_audited frontmatter

### Finding categories
- Silent suppression: most common (code swallows errors)
- Partial completion: second most common (no rollback on failure)
- Precondition violations: third (unchecked assumptions)
- Cross-platform: rare (platform shims are mostly stubs)

### Validation retest
- Original (pre-Phase 8): 7 full (35%), 12 partial (60%), 1 miss (5%)
- After Phase 8: 9 full (45%), 10 partial (50%), 1 miss (5%)
- Target of 60% full NOT met
- Gap analysis: 4 failure-mode-gap, 4 detail-gap, 2 cross-page-inference

### Why the target was missed
Phase 8 added failure-mode sections to the RIGHT pages, but the validation
issues often involve SPECIFIC parameter details (flag names, config keys,
connection timeouts) that failure-mode analysis doesn't naturally surface.
Failure modes are about error paths; many remaining partials are about
implementation-detail depth on the happy path.
