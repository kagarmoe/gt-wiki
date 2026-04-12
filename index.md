# Wiki Index

Global catalog of all pages. Organized by topic, then category.

## gastown

### Binaries

- [gt](gastown/binaries/gt.md) — main CLI (entry point, ldflags, startup sequence, command groups)

### Commands

- [gt command tree](gastown/commands/README.md) — inventory of all 111 top-level `rootCmd.AddCommand` registrations + 495 total `cobra.Command` definitions
- [version](gastown/commands/version.md) — `gt version` subcommand

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
