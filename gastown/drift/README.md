---
title: Phase 3 drift index
type: drift
status: complete
topic: gastown
created: 2026-04-15
updated: 2026-04-16
phase: 3
---

# Phase 3 Drift Index

Consolidated index of all Phase 3 findings from Batches 1-14. Phase 3 audited 271 units (211 entity pages in Sweep 1 + 60 docs files in Sweep 2) and surfaced ~83 findings across 9 drift taxonomy categories plus systematic gap enumeration (10 gap findings + 9 deliberate exclusions).

**How to read this index:**
- **Section 1** is the Phase 6 work-list: upstream corrections to the gastown source repo (code edits, docs PRs). Phase 5 (Audience classification) refines it; Phase 6 (Implementation) executes it.
- **Section 2** is the wiki self-maintenance log: our own synthesis errors, all fixed during Sweep 1. Preserved as an audit trail of what was wrong, not what is wrong.
- **Section 3** identifies cross-cutting meta-patterns that Phase 6 should batch as single PRs.
- **Section 4** lists missing entity pages (code entities with no corresponding wiki page).
- **Section 5** records subcommand coverage decisions (384 subcommands systematically classified).

**Plan:** [../../.claude/plans/2026-04-14-phase3-drift.md](../../.claude/plans/2026-04-14-phase3-drift.md) (gitignored).

---

## Section 1: Gastown corrections list (Phase 6 work-list)

Rows requiring upstream fixes to gastown source (code edits, docs PRs). Grouped by fix tier.

### Fix tier: code (cobra-drift — edit Long text / in-code docstrings)

| Entity page | Category | Summary | Severity | Release position | PR reference |
|---|---|---|---|---|---|
| [account.md](../commands/account.md) | cobra-drift | Parent `Long` text enumerates 4 subcommands; 5 are registered (`switch` omitted) | wrong | in-release | |
| [activity.md](../commands/activity.md) | cobra-drift | Refinery event type names in Cobra `Long` don't match event constants (`merged` not `merge_complete`, `merge_skipped` not `queue_processed`) | wrong | in-release | |
| [assign.md](../commands/assign.md) | cobra-drift | `--force` flag is defined but never consumed by `runAssign` | wrong | in-release | |
| [boot.md](../commands/boot.md) | cobra-drift | Boot is "a special dog" per Cobra `Long`, but is explicitly NOT a dog in the kennel (`manager.go:300`) | wrong | in-release | |
| [callbacks.md](../commands/callbacks.md) | cobra-drift | `SLING_REQUEST` handler does not "spawn polecat for the work" — it only logs | wrong | in-release | |
| [changelog.md](../commands/changelog.md) | cobra-drift | `--week` flag is defined but never consulted by `changelogSinceTime` | wrong | in-release | |
| [config.md](../commands/config.md) | cobra-drift | `config get` `Long` text omits `dolt.port` from its Supported keys list | wrong | in-release | |
| [crew.md](../commands/crew.md) | cobra-drift | `crewCmd.Long` "Commands:" lists 8; 11 visible subcommands registered (omits `status`, `rename`, `pristine`) | wrong | in-release | |
| [doctor.md](../commands/doctor.md) | cobra-drift | Cobra `Long` text enumerates a selective subset of the ~80 checks `runDoctor` registers | wrong | in-release | |
| [down.md](../commands/down.md) | cobra-drift | `downCmd.Long` recommends `gt start` as complement; `gt up` is the actual complement | wrong | in-release | |
| [git-init.md](../commands/git-init.md) | cobra-drift | `gitInitCmd.Long` lists 3 steps; code performs 4 operations (omits branch-protection hook installation) | wrong | in-release | |
| [hooks.md](../commands/hooks.md) | cobra-drift | Parent `Long` text enumerates 8 subcommands; `hooks init` is wired but unlisted | wrong | in-release | |
| [init.md](../commands/init.md) | cobra-drift | `initCmd.Long` lists 4 directories; code creates 5 (and with different paths: `crew` omitted, `refinery/rig/` not `refinery/`, `mayor/rig/` not `mayor/`) | wrong | in-release | |
| [install.md](../commands/install.md) | cobra-drift | `installCmd.Long` HQ structure lists 3 items; install creates at least 5 directories (omits `deacon/`, `plugins/`) | wrong | in-release | |
| [install.md](../commands/install.md) | cobra-drift | `installCmd.Long` references non-existent `docs/hq.md` | wrong | in-release | |
| [mail.md](../commands/mail.md) | cobra-drift | `mailCmd.Long` COMMANDS section lists 4 subcommands; 22 are registered (82% incomplete) | wrong | in-release | |
| [molecule.md](../commands/molecule.md) | cobra-drift | Parent `Long` hand-maintained subcommand categories omit four step-group members and the `await-signal` shortcut | wrong | in-release | |
| [namepool.md](../commands/namepool.md) | cobra-drift | `namepoolCmd.Long` "Examples:" shows 6 operations; 8 subcommands registered (omits `create`, `delete`) | wrong | in-release | |
| [plugin.md](../commands/plugin.md) | cobra-drift | `plugin run` gate check is cooldown-only; other gate types silently fall through as open | wrong | in-release | |
| [quota.md](../commands/quota.md) | cobra-drift | `quotaCmd.Long` COMMANDS block lists 4 subcommands; 5 registered (omits `watch`) | wrong | in-release | |
| [reaper.md](../commands/reaper.md) | cobra-drift | `reaperCmd.Long` "When run by a Dog" block lists 4 subcommands; 6 registered (omits `databases`, `run`) | wrong | in-release | |
| [repair.md](../commands/repair.md) | cobra-drift | Cobra `Long` text lists six repair targets; `runRepair` registers two checks | wrong | in-release | |
| [shell.md](../commands/shell.md) | cobra-drift | `shell install` silently re-enables Gas Town via `state.Enable(Version)` without mentioning it in `Long` text | wrong | in-release | |
| [status.md](../commands/status.md) | cobra-drift | `--fast` help text understates what it skips (omits hooks, MQ, per-agent mail — not just mail) | wrong | in-release | |
| [tap.md](../commands/tap.md) | cobra-drift | Long text omits two implemented subcommands (`list`, `polecat-stop-check`) | wrong | in-release | |
| [tap.md](../commands/tap.md) | cobra-drift | Long text advertises three unimplemented subcommands (`audit`, `inject`, `check` — all `[planned]`) | wrong | in-release | |
| [uninstall.md](../commands/uninstall.md) | cobra-drift | `--workspace` is a hardcoded two-location scan (`~/gt`, `~/gastown`); silently no-ops for any other workspace path | wrong | in-release | |
| [warrant.md](../commands/warrant.md) | cobra-drift | Long text hardcodes `~/gt/warrants/` but code uses `<townRoot>/warrants/` | wrong | in-release | |
| [witness.md](../commands/witness.md) | cobra-drift | `witness start --foreground` is advertised as functional in `Long` + flag description, but is a no-op notice | wrong | in-release | |
| [wrappers.md](../packages/wrappers.md) | cobra-drift | ABOUTME header omits `gt-gemini` wrapper (lists 2, code installs 3) | wrong | in-release | |

### Fix tier: docs (upstream docs PR)

| Entity page | Category | Summary | Severity | Release position | PR reference |
|---|---|---|---|---|---|
| [done.md](../commands/done.md) | drift | `docs/CLEANUP.md` claims `gt done` self-nukes the worktree and kills its session, but code transitions to IDLE with sandbox preserved | wrong | in-release | |
| [install.md](../commands/install.md) | drift | `docs/INSTALLING.md` shows `rigs/` in the `gt install` tree, but rigs are top-level directories | wrong | in-release | |
| [convoy.md](../concepts/convoy.md) | drift | Event-driven feeder misattributed to `operations.go` in `docs/skills/convoy/SKILL.md`; actually `convoy_manager.go:186-247` | wrong | in-release | |
| [convoy.md](../concepts/convoy.md) | drift | `docs/concepts/convoy.md` omits `staged_ready` and `staged_warnings` states (shows only `open` and `closed`) | wrong | in-release | |
| [identity.md](../concepts/identity.md) | drift | `docs/concepts/identity.md` claims `GIT_AUTHOR_NAME="gastown/polecats/toast"` (full BD_ACTOR path); code sets it to agent name only (`toast`) | wrong | in-release | |
| [go-mod.md](../files/go-mod.md) | drift | Go version disagrees across three build paths: `go.mod` 1.25.8, Dockerfile 1.25.6, Dockerfile.e2e 1.26 | wrong | in-release | |
| [goreleaser-yml.md](../files/goreleaser-yml.md) | drift | Release header advertises `brew install gastown` but no `brews:` block exists in `.goreleaser.yml` | wrong | in-release | |
| [health.md](../packages/health.md) | drift | Package doc claims Doctor Dog shares this package; only `gt health` imports it | wrong | in-release | |
| [telemetry.md](../packages/telemetry.md) | drift | `docs/design/otel/otel-architecture.md` claims `RecordMol*`/`RecordBeadCreate`/`RecordAgentInstantiate` don't exist; they do. Says "18 metric instruments"; code has 24 | wrong | in-release | |

### Fix tier: docs (docs-only — no wiki page annotation)

These findings were surfaced during Sweep 2 (docs-file audit) and affect only upstream docs files. No wiki entity page was annotated because the wiki already has the correct information.

| Docs file | Category | Summary | Severity | Release position | Batch | PR reference |
|---|---|---|---|---|---|---|
| `docs/design/persistent-polecat-pool.md` | drift | Says `gt polecat pool init <rig>` (space-separated); code registers as `pool-init` (hyphenated) | wrong | in-release | 11c | |
| `docs/HOOKS.md` | drift | Known Gap #2 claims `gt tap guard dangerous-command` doesn't exist; `tap_guard_dangerous.go` exists at HEAD and v1.0.0 | wrong | in-release | 10e | |
| `docs/design/directives-and-overlays.md` | drift | "CLI commands are being added" label stale; `gt directive show/edit/list` and `gt formula overlay show/edit/list` all exist | wrong | in-release | 11e | |
| `docs/design/architecture.md` | drift | MQ implementation phases table says GatesParallel "In progress" and batch-then-bisect "Blocked by Phase 1"; both fully implemented | wrong | in-release | 11f | |
| `docs/design/architecture.md` | drift | Dead reference to `[Watchdog Chain](watchdog-chain.md)`; no such file exists (closest: `dog-infrastructure.md`) | wrong | in-release | 11f | |
| `docs/design/property-layers.md` | drift | Dead reference to `[Watchdog Chain](watchdog-chain.md)` | wrong | in-release | 11g | |
| `docs/design/scheduler.md` | drift | Dead reference to `[Watchdog Chain](watchdog-chain.md)` | wrong | in-release | 11h | |
| `docs/design/polecat-self-managed-completion.md` | drift | Header says "Status: Design proposal" but all three migration phases are shipped in code | wrong | in-release | 11i | |
| `docs/design/mail-protocol.md` | drift | Stale Polecat->Witness->Refinery completion flow; polecat now nudges refinery directly (self-managed completion) | wrong | in-release | 11l | |
| `docs/design/polecat-lifecycle-patrol.md` | drift | Stale cleanup pipeline flow (same as mail-protocol.md); witness no longer required checkpoint | wrong | in-release | 11o | |

### Fix tier: preserve-as-vision (implementation-status: unbuilt/partial)

These findings are NOT bugs. They describe aspirational or partially-implemented features. Phase 6 action: add "not yet implemented" callouts where the docs don't already have them.

| Source | Category | Summary | Severity | Release position | Batch | PR reference |
|---|---|---|---|---|---|---|
| [witness.md](../commands/witness.md) | implementation-status: vestigial | `--foreground` flag kept as runtime notice; patrol loop moved to `mol-witness-patrol` molecule | wrong | in-release | 1d | |
| [keepalive.md](../packages/keepalive.md) | implementation-status: partial | Package fully implemented (Touch/Read/Age API) but zero importers in codebase | incomplete | in-release | 2b | |
| `docs/design/dog-infrastructure.md` | implementation-status: unbuilt | Dog Pool Architecture (ShutdownDanceState, DogPool struct, warrant queue, `gt dog dances/warrants/pool status`) — no code | wrong | in-release | 11j | |
| `docs/design/model-aware-molecules.md` | implementation-status: unbuilt | Entire model-aware molecules feature (per-step model constraints, subscription routing, OpenRouter pricing) — no code; `internal/models/` doesn't exist | wrong | in-release | 11m | |
| `docs/design/ledger-export-triggers.md` | implementation-status: unbuilt | Entire ledger export trigger system (bead closure triggers, convoy triggers, skill derivation engine, HOP economy) — no code | wrong | in-release | 11n | |
| `docs/design/mol-mall-design.md` | implementation-status: unbuilt | Mol Mall registry system (public/private/federated registries, HOP URI scheme, formula publishing) — no code beyond embedded formulas | wrong | in-release | 12a | |
| `docs/design/witness-at-team-lead.md` | implementation-status: unbuilt | Witness-as-AT-team-lead architecture (delegate mode, AT teammate spawning) — no code; depends on Claude Code Agent Teams | wrong | in-release | 12b | |
| `docs/design/sandboxed-polecat-execution.md` | implementation-status: unbuilt | Sandboxed execution (exitbox container backend, daytona cloud backend, SandboxAdapter interface) — no code; all execution is local tmux | wrong | in-release | 12d | |
| `docs/design/factory-worker-api.md` | implementation-status: unbuilt | Factory Worker API (7 structured endpoints) — no code; all agent interaction remains tmux-based (28 touch points) | wrong | in-release | 12f | |
| `docs/design/plugin-system.md` | implementation-status: partial | Core dispatch shipped (`gt dog dispatch --plugin`, `gt plugin` CLI); design doc status label stale ("not yet implemented"). Advanced features (versioning, marketplace) not implemented | incomplete | in-release | 12c | |
| `docs/design/federation.md` | implementation-status: partial | Federation infrastructure exists (Dolt remotes, wasteland commands); core features (HOP URI, cross-workspace queries, sovereignty model) not implemented | incomplete | in-release | 12e | |
| `docs/design/formula-resolution.md` | implementation-status: partial | Tier 2 (town/rig overrides) + Tier 3 (embedded) work; Tier 1 (project-committed formulas) + Mol Mall integration not implemented | incomplete | in-release | 12g | |

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

## Summary statistics

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
