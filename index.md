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

### Inventory

- [gastown/inventory/README.md](gastown/inventory/README.md) — A-level enumeration index for the gastown topic
  - [repo-root](gastown/inventory/repo-root.md) — top-level files and directories at `/home/kimberly/repos/gastown/`
  - [docs-tree](gastown/inventory/docs-tree.md) — every file under `docs/` with line counts
  - [go-packages](gastown/inventory/go-packages.md) — every Go package under `cmd/`, `internal/`, `plugins/` with file counts
