# Wiki Index

Global catalog of all pages. Organized by topic, then category.

## gastown

### Binaries

- [gt](gastown/binaries/gt.md) — main CLI (entry point, ldflags, startup sequence, command groups)
- [gt-proxy-server](gastown/binaries/gt-proxy-server.md) — long-lived host-side mTLS proxy server for polecat containers
- [gt-proxy-client](gastown/binaries/gt-proxy-client.md) — container-side shim that forwards `gt`/`bd` calls to the proxy server

### Commands

- [gt command tree](gastown/commands/README.md) — inventory of all 111 top-level `rootCmd.AddCommand` registrations + 495 total `cobra.Command` definitions
- **Diagnostics group (22 commands)** — fully mapped in Batch 3a.
- **Configuration group (11 commands)** — fully mapped in Batch 3b.
- **Work Management group (26 commands)** — fully mapped in Batch 3c.
- **Agent Management group (12 commands)** — fully mapped in Batch 3d.
- **Communication group (7 commands)** — fully mapped in Batch 3e.
- **Services group (11 commands)** — fully mapped in Batch 3f.
- **Workspace group (7 commands)** — fully mapped in Batch 3g. All 7 cobra groups complete.
- **Ungrouped (15 commands)** — mapped in Batch 3h. **All 111 top-level commands now mapped.** Individual pages linked from [gastown/commands/README.md](gastown/commands/README.md) entity-page column.

### Packages

Platform services (Batch 4):

- [cli](gastown/packages/cli.md) — CLI binary-name override (`cli.Name()`)
- [config](gastown/packages/config.md) — town settings, agent registry, startup commands
- [session](gastown/packages/session.md) — tmux session substrate for all agent roles
- [style](gastown/packages/style.md) — lipgloss style wrappers
- [telemetry](gastown/packages/telemetry.md) — OTEL provider (opt-in only)
- [ui](gastown/packages/ui.md) — Ayu palette, theme init, capability detection
- [util](gastown/packages/util.md) — atomic file writes, exec, orphan cleanup
- [version](gastown/packages/version.md) — build commit + stale binary detection
- [workspace](gastown/packages/workspace.md) — town-root discovery

Data layer (Batch 5):

- [beads](gastown/packages/beads.md) — Go library interface to beads databases; ~10,300 lines of Gas Town domain logic
- [channelevents](gastown/packages/channelevents.md) — file-based per-channel pub/sub
- [doltserver](gastown/packages/doltserver.md) — per-town Dolt MySQL server lifecycle + imposter killing
- [events](gastown/packages/events.md) — append-only JSONL activity feed; 21 event types
- [lock](gastown/packages/lock.md) — agent-identity file locks with tmux-aware stale detection
- [mail](gastown/packages/mail.md) — durable messaging backed by beads with `gt:message` label
- [mq](gastown/packages/mq.md) — merge-request ID minter (NOT a message queue)
- [nudge](gastown/packages/nudge.md) — ephemeral file-drop queue for agent prompts

Agent runtime (Batch 6):

- [mayor](gastown/packages/mayor.md) — orchestrator lifecycle + ACP
- [deacon](gastown/packages/deacon.md) — daemon-tick watchdog
- [crew](gastown/packages/crew.md) — persistent workers
- [dog](gastown/packages/dog.md) — cross-rig reusable workers
- [polecat](gastown/packages/polecat.md) — ephemeral feature workers
- [refinery](gastown/packages/refinery.md) — per-rig merge queue processor
- [witness](gastown/packages/witness.md) — polecat health monitor library
- [reaper](gastown/packages/reaper.md) — wisp/bead TTL cleanup
- [wisp](gastown/packages/wisp.md) — small utility package
- [convoy](gastown/packages/convoy.md) — cross-rig work tracking
- [rig](gastown/packages/rig.md) — rig workspace manager
- [formula](gastown/packages/formula.md) — formula template loader
- [plugin](gastown/packages/plugin.md) — Deacon patrol plugin loader

Diagnostics & health (Batch 7):

- [doctor](gastown/packages/doctor.md) — health-check registry and fix engine (70+ files)
- [health](gastown/packages/health.md) — Dolt data-plane health primitives
- [keepalive](gastown/packages/keepalive.md) — agent-activity signaling
- [deps](gastown/packages/deps.md) — external binary prerequisites (bd + dolt)

Long-running processes (Batch 8):

- [daemon](gastown/packages/daemon.md) — per-town singleton; main heartbeat loop
- [tmux](gastown/packages/tmux.md) — tmux wrapper library (~150 methods)
- [runtime](gastown/packages/runtime.md) — agent-runtime integration helpers

Supporting libraries (Batch 9 — 24 packages):

- [acp](gastown/packages/acp.md) — Agent Client Protocol proxy
- [activity](gastown/packages/activity.md), [agentlog](gastown/packages/agentlog.md), [constants](gastown/packages/constants.md), [estop](gastown/packages/estop.md), [feed](gastown/packages/feed.md), [git](gastown/packages/git.md), [github](gastown/packages/github.md), [hooks](gastown/packages/hooks.md), [hookutil](gastown/packages/hookutil.md), [krc](gastown/packages/krc.md), [protocol](gastown/packages/protocol.md), [quota](gastown/packages/quota.md), [scheduler](gastown/packages/scheduler.md), [shell](gastown/packages/shell.md), [state](gastown/packages/state.md), [suggest](gastown/packages/suggest.md), [templates](gastown/packages/templates.md), [testutil](gastown/packages/testutil.md), [townlog](gastown/packages/townlog.md), [tui](gastown/packages/tui.md), [wasteland](gastown/packages/wasteland.md), [web](gastown/packages/web.md), [wrappers](gastown/packages/wrappers.md)

### Roles (Batch 6)

Gas Town agent personas:

- [mayor](gastown/roles/mayor.md) — town-level orchestrator
- [polecat](gastown/roles/polecat.md) — primary feature-building worker
- [crew](gastown/roles/crew.md) — persistent user-managed worker
- [dog](gastown/roles/dog.md) — cross-rig infrastructure worker
- [deacon](gastown/roles/deacon.md) — town-level watchdog
- [refinery](gastown/roles/refinery.md) — per-rig merge queue processor
- [witness](gastown/roles/witness.md) — per-rig polecat health monitor
- [reaper](gastown/roles/reaper.md) — on-demand cleanup via `mol-dog-reaper`

### Concepts (Batch 6)

Abstract domain ideas:

- [rig](gastown/concepts/rig.md) — workspace abstraction
- [convoy](gastown/concepts/convoy.md) — cross-rig work-tracking unit
- [formula](gastown/concepts/formula.md) — static TOML workflow template
- [molecule](gastown/concepts/molecule.md) — running instance of a formula
- [wisp](gastown/concepts/wisp.md) — ephemeral bead (TTL-compactable)
- [directive](gastown/concepts/directive.md) — operator-provided role override
- [identity](gastown/concepts/identity.md) — agent identity system

### Workflows (Batch 6)

Multi-step flows:

- [convoy-launch](gastown/workflows/convoy-launch.md) — staging → launch → dispatch → merge landing
- [polecat-lifecycle](gastown/workflows/polecat-lifecycle.md) — name allocation → work → nuke

### Files

- [Makefile](gastown/files/makefile.md) — canonical build recipe; produces `gt`, `gt-proxy-server`, `gt-proxy-client` with the `BuiltProperly` ldflag
- [Dockerfile](gastown/files/dockerfile.md) — primary container image for `gt`
- [Dockerfile.e2e](gastown/files/dockerfile-e2e.md) — end-to-end test container
- [docker-compose.yml](gastown/files/docker-compose.md) — Compose spec
- [docker-entrypoint.sh](gastown/files/docker-entrypoint.md) — container entrypoint shell script
- [flake.nix](gastown/files/flake-nix.md) — Nix flake (dev shell, packages)
- [.goreleaser.yml](gastown/files/goreleaser-yml.md) — GoReleaser release-pipeline build config
- [.golangci.yml](gastown/files/golangci-yml.md) — golangci-lint configuration
- [go.mod](gastown/files/go-mod.md) — Go module manifest
- [.claude/ directory](gastown/files/claude-dir.md) — Claude Code agent-facing surface (Batch 11)
- [.opencode/ directory](gastown/files/opencode-dir.md) — OpenCode agent-facing surface (Batch 11)
- [templates/agents/ directory](gastown/files/templates-agents.md) — agent-runtime templates (Batch 11)

### Plugins (Batch 10)

- [Plugin inventory](gastown/plugins/README.md) — 14 plugin directories; 13 declarative (shell + TOML), 1 with Go source
- [dolt-snapshots](gastown/plugins/dolt-snapshots.md) — Dolt snapshot plugin (the only Go-based plugin); tags production databases at convoy lifecycle boundaries

### Drift (Phase 3)

- [Phase 3 drift index](gastown/drift/README.md) — consolidated index of ~83 findings: drift (wrong), gaps (missing), coverage decisions
- [Gap findings](gastown/drift/gaps.md) — systematic code-to-wiki gap enumeration (Batch 14): 6 missing packages, 4 subcommand gaps, 9 deliberate exclusions

### Inventory

- [gastown/inventory/README.md](gastown/inventory/README.md) — A-level enumeration index for the gastown topic
  - [repo-root](gastown/inventory/repo-root.md) — top-level files and directories at `/home/kimberly/repos/gastown/`
  - [docs-tree](gastown/inventory/docs-tree.md) — every file under `docs/` with line counts
  - [go-packages](gastown/inventory/go-packages.md) — every Go package under `cmd/`, `internal/`, `plugins/` with file counts
  - [auxiliary](gastown/inventory/auxiliary.md) — 9 auxiliary top-level directories (scripts/, templates/, gt-model-eval/, npm-package/, .github/, .githooks/, .claude/, .opencode/, .runtime/) — Batch 11
