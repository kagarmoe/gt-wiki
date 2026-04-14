---
title: gastown topic landing
type: note
status: stub
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/
tags: [landing, index]
---

# gastown

## Mission (current phase)

This topic **maps the gastown codebase in the wiki** so that every part
is represented at inventory level and every load-bearing part has a
full entity page that **accurately describes what the code actually
does**, read off the source and cited with `file:line` references. See
the plan at `.claude/plans/2026-04-11-gastown-map.md` for the design,
the 12-layer traversal, the load-bearing test, and the cross-linking
discipline.

**Deliverable this phase:** a wiki that answers "what does gastown's
`<component>` actually do?" with code-grounded pages. Nothing else.

**Explicitly out of scope this phase:**

- Drift analysis, gap analysis against upstream docs, audience
  classification, upstream documentation rewrites — all deferred to
  later phases.
- Reading `docs/` content beyond inventory rows — phase 3.
- Edits to anything outside `~/repos/gt-wiki/`. `/home/kimberly/repos/gastown/`
  is read-only reference.

## Working model

- Read from `/home/kimberly/repos/gastown/` (source of truth, never
  modified).
- Produce wiki pages under this topic following the schema's entity
  template (frontmatter + "What it actually does" + `file:line`
  source refs + "Notes / open questions").
- Link every entity page to every adjacent entity per the cross-linking
  discipline in the plan — forward links, back-references, alias
  coverage, `grep`-verified before commit. This is what makes the wiki
  answer multi-hop questions later.
- Track follow-up investigation work in **beads** — `bd list -l gastown`
  for the live queue. Agent-task conventions in the "Task tracking"
  section of [../CLAUDE.md](../CLAUDE.md).

## Coordinates

- **Source under investigation:** `/home/kimberly/repos/gastown/`
- **Upstream:** https://github.com/gastownhall/gastown
- **Go module path declared in `go.mod`:** `github.com/steveyegge/gastown`
  (the `gastownhall` repo self-identifies under the `steveyegge`
  namespace, so `go install` must use the `steveyegge` path)
- **Separate runtime workspace:** `~/gt/` — the Docker-containerized
  deployment of `gt`. Not part of the wiki; independent from this
  mapping effort.

## Scope of the map

gastown is a multi-agent orchestration system for AI coding runtimes,
managed by the `gt` CLI. The map will cover, in order (see plan for
the 12 layers):

- Build & packaging (Makefile, Dockerfiles, flake.nix, .goreleaser.yml, go.mod)
- Binaries & entry points (`cmd/gt`, `cmd/gt-proxy-server`, `cmd/gt-proxy-client`)
- The command layer (all ~107 top-level commands in `internal/cmd/`)
- Platform services (`cli`, `config`, `session`, `workspace`, `style`, `ui`, `telemetry`, `version`, `util`)
- Data layer (`beads`, `doltserver`, `mail`, `nudge`, `events`, etc.)
- Agent runtime — packages + first-class domain entities: roles
  (mayor, polecat, crew, dog, deacon, refinery, witness, wisp, reaper),
  concepts (convoy, rig, molecule, formula, directive, identity,
  polecat-lifecycle), services, workflows
- Diagnostics & health (`doctor`, `health`, `keepalive`, `deps`)
- Long-running processes (`daemon`, `tmux`, `runtime`)
- Supporting libraries
- Plugins (`plugins/*`)
- Auxiliary trees (`scripts/`, `templates/`, `gt-model-eval/`, `npm-package/`, `.github/`, `.githooks/`, `.claude/`, `.opencode/`, `.runtime/`)
- Documentation tree (`docs/`) — inventory only this phase

## Sub-index

### Binaries

- [gt](binaries/gt.md) — main CLI; entry point, ldflags, startup sequence, command groups, exempt-command maps
- [gt-proxy-server](binaries/gt-proxy-server.md) — long-lived host-side mTLS proxy server for polecat containers
- [gt-proxy-client](binaries/gt-proxy-client.md) — container-side shim that forwards `gt`/`bd` calls to the proxy server (or execs the real binary)

### Commands

- [gt command tree](commands/README.md) — inventory of all 111 top-level `rootCmd.AddCommand` registrations + 495 total `cobra.Command` definitions
- **Diagnostics group** (22 commands, fully mapped) — see the `entity page` column of [commands/README.md](commands/README.md) for individual links: [activity](commands/activity.md), [audit](commands/audit.md), [checkpoint](commands/checkpoint.md), [costs](commands/costs.md), [dashboard](commands/dashboard.md), [doctor](commands/doctor.md), [feed](commands/feed.md), [heartbeat](commands/heartbeat.md), [info](commands/info.md), [log](commands/log.md), [metrics](commands/metrics.md), [patrol](commands/patrol.md), [prime](commands/prime.md), [repair](commands/repair.md), [seance](commands/seance.md), [stale](commands/stale.md), [status](commands/status.md), [thanks](commands/thanks.md), [upgrade](commands/upgrade.md), [version](commands/version.md), [vitals](commands/vitals.md), [whoami](commands/whoami.md)
- **Configuration group** (11 commands, fully mapped): [account](commands/account.md), [config](commands/config.md), [directive](commands/directive.md), [disable](commands/disable.md), [enable](commands/enable.md), [hooks](commands/hooks.md), [issue](commands/issue.md), [plugin](commands/plugin.md), [shell](commands/shell.md), [theme](commands/theme.md), [uninstall](commands/uninstall.md)
- **Work Management group** (26 commands, fully mapped): [assign](commands/assign.md), [bead](commands/bead.md), [cat](commands/cat.md), [changelog](commands/changelog.md), [cleanup](commands/cleanup.md), [close](commands/close.md), [compact](commands/compact.md), [convoy](commands/convoy.md), [done](commands/done.md), [formula](commands/formula.md), [handoff](commands/handoff.md), [hook](commands/hook.md), [molecule](commands/molecule.md), [mountain](commands/mountain.md), [mq](commands/mq.md), [orphans](commands/orphans.md), [prune-branches](commands/prune-branches.md), [ready](commands/ready.md), [release](commands/release.md), [resume](commands/resume.md), [scheduler](commands/scheduler.md), [sling](commands/sling.md), [synthesis](commands/synthesis.md), [trail](commands/trail.md), [unsling](commands/unsling.md), [wl](commands/wl.md)
- **Agent Management group** (12 commands, fully mapped): [agents](commands/agents.md), [boot](commands/boot.md), [callbacks](commands/callbacks.md), [deacon](commands/deacon.md), [dog](commands/dog.md), [mayor](commands/mayor.md), [polecat](commands/polecat.md), [refinery](commands/refinery.md), [role](commands/role.md), [session](commands/session.md), [signal](commands/signal.md), [witness](commands/witness.md)
- **Communication group** (7 commands, fully mapped): [broadcast](commands/broadcast.md), [dnd](commands/dnd.md), [escalate](commands/escalate.md), [mail](commands/mail.md), [notify](commands/notify.md), [nudge](commands/nudge.md), [peek](commands/peek.md)
- **Services group** (11 commands, fully mapped): [daemon](commands/daemon.md), [dolt](commands/dolt.md), [down](commands/down.md), [estop](commands/estop.md), [maintain](commands/maintain.md), [quota](commands/quota.md), [reaper](commands/reaper.md), [shutdown](commands/shutdown.md), [start](commands/start.md), [thaw](commands/thaw.md), [up](commands/up.md)
- **Workspace group** (7 commands, fully mapped): [crew](commands/crew.md), [git-init](commands/git-init.md), [init](commands/init.md), [install](commands/install.md), [namepool](commands/namepool.md), [rig](commands/rig.md), [worktree](commands/worktree.md)
- **Ungrouped** (commands with no `GroupID`, mapped in Batch 3h): [agent-log](commands/agent-log.md), [commit](commands/commit.md), [cycle](commands/cycle.md), [forget](commands/forget.md), [health](commands/health.md), [krc](commands/krc.md), [memories](commands/memories.md), [nudge-poller](commands/nudge-poller.md), [proxy-subcmds](commands/proxy-subcmds.md), [remember](commands/remember.md), [show](commands/show.md), [status-line](commands/status-line.md), [tap](commands/tap.md), [town](commands/town.md), [warrant](commands/warrant.md) — **all 111 top-level commands now mapped**

### Packages

Platform service packages under `internal/` (Batch 4 — Layer d):

- [cli](packages/cli.md) — `cli.Name()` for `GT_COMMAND` binary-name override
- [config](packages/config.md) — town settings, agent registry, startup command builders, cost-tier system
- [session](packages/session.md) — tmux session substrate for every agent role
- [style](packages/style.md) — lipgloss style wrappers + icon prefixes + small table renderer
- [telemetry](packages/telemetry.md) — OTEL provider (**strictly opt-in**; defaults to localhost VictoriaMetrics/VictoriaLogs; no off-box leakage)
- [ui](packages/ui.md) — Ayu palette, theme init, capability detection, glamour markdown rendering
- [util](packages/util.md) — atomic file writes, exec helpers, process-group management, orphan Claude cleanup
- [version](packages/version.md) — build-time Commit var + `CheckStaleBinary` with false-positive guards
- [workspace](packages/workspace.md) — town-root discovery walks with env-var fallbacks

Data layer packages (Batch 5 — Layer e):

- [beads](packages/beads.md) — Go library interface to beads databases; hybrid subprocess/SDK dispatch; ~10,300 lines of Gas Town domain logic (agent beads, molecules, merge slots, rig identity, channels, escalations, delegation, routing)
- [channelevents](packages/channelevents.md) — file-based pub/sub for named channels (one file per event; rendezvous semantics)
- [doltserver](packages/doltserver.md) — per-town Dolt MySQL server (port 3307) lifecycle manager + imposter killing
- [events](packages/events.md) — append-only JSONL activity feed at `<townRoot>/.events.jsonl`; 21 event types
- [lock](packages/lock.md) — agent-identity file locks with tmux-session-aware stale detection
- [mail](packages/mail.md) — durable messaging: messages are beads with `gt:message` label; ~3500 lines of routing/threading/claim logic
- [mq](packages/mq.md) — merge-request ID minter (SHA-256 based; NOT a message queue despite the name)
- [nudge](packages/nudge.md) — ephemeral queue: JSON files under `<townRoot>/.runtime/nudge_queue/<session>/`; hook-drained or poller-drained

Agent runtime packages (Batch 6 — Layer f):

- [mayor](packages/mayor.md) — orchestrator lifecycle + ACP mode + cleanup veto
- [deacon](packages/deacon.md) — daemon-tick watchdog; 5 state-file substrates (heartbeat, paused, feed-stranded, redispatch, health-check)
- [crew](packages/crew.md) — persistent user-managed workers; full git clones
- [dog](packages/dog.md) — cross-rig reusable workers with worktrees into every rig; managed by the Deacon
- [polecat](packages/polecat.md) — ephemeral feature-building workers; identity via name pool + agent bead; Dolt-retry-aware spawn
- [refinery](packages/refinery.md) — per-rig merge queue processor
- [witness](packages/witness.md) — per-rig polecat health monitor (library for `mol-witness-patrol` molecule)
- [reaper](packages/reaper.md) — wisp/bead TTL cleanup (SQL-direct; zero gastown package imports)
- [wisp](packages/wisp.md) — small utility package; the wisp **concept** lives in `internal/beads` + `internal/doltserver`
- [convoy](packages/convoy.md) — cross-rig work tracking unit primitives (most convoy logic lives in `internal/cmd/convoy.go`)
- [rig](packages/rig.md) — rig workspace manager
- [formula](packages/formula.md) — formula/molecule template loader
- [plugin](packages/plugin.md) — Deacon-patrol plugin loader

Diagnostics & health packages (Batch 7 — Layer g):

- [doctor](packages/doctor.md) — Gas Town's health-check registry and fix engine. Largest package in the codebase (70+ non-test files); powers `gt doctor`, `gt repair`, and parts of `gt upgrade`.
- [health](packages/health.md) — reusable Dolt-data-plane health-check primitives (TCP, latency, database count, zombie servers, backup freshness)
- [keepalive](packages/keepalive.md) — best-effort agent-activity signaling via `<townRoot>/.runtime/keepalive.json`; nil-sentinel design
- [deps](packages/deps.md) — external binary prerequisites (`bd` + `dolt`) with version pinning; feeds `gt doctor` prereq checks

Long-running processes (Batch 8 — Layer h):

- [daemon](packages/daemon.md) — per-town daemon singleton; main heartbeat loop driving patrol cycles; owns Dolt server lifecycle + backup + restart-tracker exponential backoff
- [tmux](packages/tmux.md) — wrapper library for driving tmux under automation; ~150 methods on one `Tmux` struct; cross-platform (Unix + Windows/psmux support)
- [runtime](packages/runtime.md) — agent-runtime integration helpers (Claude Code, Codex, Copilot, Gemini); per-provider hook settings + startup fallback matrix

Supporting libraries (Batch 9 — Layer i):

- [acp](packages/acp.md) — Agent Client Protocol proxy between IDE clients and Claude Code agents
- [activity](packages/activity.md) — activity-tracking color/age classification
- [agentlog](packages/agentlog.md) — AI-agent conversation-log tailer (ClaudeCode + OpenCode adapter)
- [constants](packages/constants.md) — shared string/duration/emoji constants
- [estop](packages/estop.md) — town-wide and per-rig emergency-stop sentinel primitives
- [feed](packages/feed.md) — feed curator daemon; tails `.events.jsonl`, writes `.feed.jsonl`
- [git](packages/git.md) — 2,300-line subprocess wrapper around git + 14 clone permutations + worktree lifecycle
- [github](packages/github.md) — REST + GraphQL HTTP client for PR lifecycle (distinct from git's `gh` CLI bridges)
- [hooks](packages/hooks.md) — Claude hook-settings base + role override merge engine + embedded-template installer
- [hookutil](packages/hookutil.md) — `IsAutonomousRole` role classifier (single function)
- [krc](packages/krc.md) — Key Record Chronicle TTL pruner and forensic decay model
- [protocol](packages/protocol.md) — typed Witness↔Refinery↔Polecat messages over mail
- [quota](packages/quota.md) — Claude Code rate-limit detection + Keychain token rotation (darwin-only)
- [scheduler](packages/scheduler.md) — empty-namespace parent; all code in `capacity/` subpackage (pure scheduling functions)
- [shell](packages/shell.md) — shell-integration installer; zsh/bash `cd` hook script
- [state](packages/state.md) — XDG-compliant global enable/disable toggle via `~/.local/state/gastown/state.json`
- [suggest](packages/suggest.md) — did-you-mean command suggestion (weighted similarity scoring)
- [templates](packages/templates.md) — embedded role/message/CLAUDE.md templates + commands subpackage
- [testutil](packages/testutil.md) — env-sanitized subprocess builders + Dolt testcontainer lifecycle
- [townlog](packages/townlog.md) — human-readable agent-lifecycle log writer at `<townRoot>/logs/town.log`
- [tui](packages/tui.md) — empty-namespace parent; two large subpackages (`convoy/`, `feed/`) totaling ~4.6k lines
- [wasteland](packages/wasteland.md) — DoltHub federation (Spider Protocol fraud detection, trust tiers)
- [web](packages/web.md) — dashboard HTTP layer (embedded templates, CSRF API, setup mode)
- [wrappers](packages/wrappers.md) — `gt-codex`/`gt-gemini`/`gt-opencode` shell-script installer

### Roles (Batch 6 — Layer f)

Gas Town agent personas — the "characters" with identity, decisions, and autonomy. Each role has a dedicated persona page + a code-side package page.

- [mayor](roles/mayor.md) — town-level orchestrator; one per machine; the only agent that runs in two substrates (tmux + ACP)
- [polecat](roles/polecat.md) — primary feature-building worker; persistent identity + ephemeral sessions; rig-scoped; themed names
- [crew](roles/crew.md) — persistent user-managed worker; full git clones; rig-scoped; stable names
- [dog](roles/dog.md) — cross-rig infrastructure worker; worktrees into every rig; managed by the Deacon; idle ↔ working; invisible to `gt agents`
- [deacon](roles/deacon.md) — town-level watchdog; mechanical sibling of the Mayor; patrol loops + heartbeat + escalation budget
- [refinery](roles/refinery.md) — per-rig merge queue processor; pre-merge and post-squash quality gates; batch-then-bisect merging
- [witness](roles/witness.md) — per-rig polecat health monitor; three completion-detection paths; spawn-storm circuit breaker
- [reaper](roles/reaper.md) — on-demand cleanup role performed by a Dog executing `mol-dog-reaper`; decision-less SQL pipeline

### Concepts (Batch 6 — Layer f)

Abstract domain ideas that span multiple packages.

- [rig](concepts/rig.md) — workspace abstraction; one repo + one refinery + one witness + polecats + crew; `<rigRoot>` canonical layout
- [convoy](concepts/convoy.md) — cross-rig work-tracking unit; `tracks` dependency type; multi-stage lifecycle; `hq-` prefix
- [formula](concepts/formula.md) — static TOML template with steps, variables, overlays; four types (convoy/workflow/expansion/aspect)
- [molecule](concepts/molecule.md) — running bead-tracked instance of a formula (formula : molecule :: class : instance)
- [wisp](concepts/wisp.md) — ephemeral bead in the `wisps` table; TTL-compactable; promotion-gated
- [directive](concepts/directive.md) — operator-provided role override; concatenated town-then-rig; injected at prime time
- [identity](concepts/identity.md) — agent identity system; `detectSender()` chain; `GT_ROLE` / `GT_RIG` / agent bead / session name

### Workflows (Batch 6 — Layer f)

Multi-step flows that cross packages and roles.

- [convoy-launch](workflows/convoy-launch.md) — stage → launch → initial dispatch → watch subscription → reactive continuation → Deacon safety-net polling → refinery merges → last-merge landing
- [polecat-lifecycle](workflows/polecat-lifecycle.md) — name allocation → worktree + bead → session spawn + hook → working → `gt done` → Witness POLECAT_DONE → refinery merges → idle or nuke → zombie detection

### Files

- [Makefile](files/makefile.md) — canonical build recipe; produces `gt`, `gt-proxy-server`, `gt-proxy-client` with the `BuiltProperly` ldflag
- [Dockerfile](files/dockerfile.md) — primary container image for `gt`
- [Dockerfile.e2e](files/dockerfile-e2e.md) — end-to-end test container
- [docker-compose.yml](files/docker-compose.md) — Compose spec
- [docker-entrypoint.sh](files/docker-entrypoint.md) — container entrypoint shell script
- [flake.nix](files/flake-nix.md) — Nix flake (dev shell, packages)
- [.goreleaser.yml](files/goreleaser-yml.md) — GoReleaser release-pipeline build config
- [.golangci.yml](files/golangci-yml.md) — golangci-lint configuration
- [go.mod](files/go-mod.md) — Go module manifest
- [.claude/ directory](files/claude-dir.md) — Claude Code agent-facing surface: 3 slash commands (`/backup`, `/patrol`, `/reaper`) + 4 skills (crew-commit, ghi-list, pr-list, pr-sheriff) — Batch 11 — Layer k
- [.opencode/ directory](files/opencode-dir.md) — OpenCode agent-facing surface: `/handoff` slash command + `gastown.js` plugin (injects `gt prime` into session system prompt, records cost on session.deleted) — Batch 11 — Layer k
- [templates/agents/ directory](files/templates-agents.md) — agent-runtime templates: `opencode.json.tmpl` + `opencode-models.json` (per-model `ready_delay_ms` presets). Consumed by the templates package at agent-spawn time — Batch 11 — Layer k

### Plugins (Batch 10 — Layer j)

Declarative Deacon-patrol plugins at `plugins/` in the gastown repo root. 14 plugin directories; 13 are declarative (shell + TOML), 1 has Go source.

- [Plugin inventory](plugins/README.md) — all 14 plugins with one-line descriptions, gate types, and observations
- [dolt-snapshots](plugins/dolt-snapshots.md) — the only plugin with Go source code; creates immutable Dolt tags (and branches) at convoy lifecycle boundaries for audit, diff, and rollback

### Inventory

- [inventory/README.md](inventory/README.md) — neutral A-level enumeration of repo contents
  - [repo-root](inventory/repo-root.md) — top-level files and directories at `/home/kimberly/repos/gastown/`
  - [docs-tree](inventory/docs-tree.md) — every file under `docs/` with line counts
  - [go-packages](inventory/go-packages.md) — every Go package under `cmd/`, `internal/`, `plugins/` with file counts
  - [auxiliary](inventory/auxiliary.md) — 9 top-level auxiliary directories: `scripts/`, `templates/`, `gt-model-eval/`, `npm-package/`, `.github/`, `.githooks/`, `.claude/`, `.opencode/`, `.runtime/` — Batch 11 — Layer k
