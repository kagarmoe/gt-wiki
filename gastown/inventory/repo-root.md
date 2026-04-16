---
title: gastown repo-root inventory
type: note
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/
tags: [inventory, enumeration, repo-root]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gastown repo-root inventory

Every file and directory at the top level of `/home/kimberly/repos/gastown/`
as of 2026-04-11. Neutral enumeration — no interpretation.

**Capture method:** `ls -la` for file sizes; `find <dir> -mindepth 1
-maxdepth 1 | wc -l` for immediate children of each subdirectory.

## Top-level files (24)

| file                  | bytes  | notes                                                                                          |
|-----------------------|--------|------------------------------------------------------------------------------------------------|
| `AGENTS.md`           |   4334 | Agent-facing instructions file. Separate from `README.md`. Content not yet read.               |
| `CHANGELOG.md`        |  86563 | Project changelog. Large.                                                                      |
| `CODE_OF_CONDUCT.md`  |   1553 | Code of conduct text.                                                                          |
| `CONTRIBUTING.md`     |   5561 | Contributor guide.                                                                             |
| `Dockerfile`          |   1734 | Primary Dockerfile. See [../files/dockerfile.md](../files/dockerfile.md).                       |
| `Dockerfile.e2e`      |   2396 | E2E test Dockerfile. See [../files/dockerfile-e2e.md](../files/dockerfile-e2e.md).              |
| `LICENSE`             |   1068 | License file. Content not yet read (size consistent with MIT).                                 |
| `Makefile`            |   7688 | Build recipes. See [../files/makefile.md](../files/makefile.md).                                |
| `README.md`           |  25265 | Project README. 740 lines per earlier `wc -l`.                                                 |
| `RELEASING.md`        |   4785 | Release process documentation.                                                                 |
| `SECURITY.md`         |   1209 | Security policy.                                                                               |
| `codecov.yml`         |   2305 | Codecov configuration.                                                                         |
| `docker-compose.yml`  |   1178 | Docker Compose spec. See [../files/docker-compose.md](../files/docker-compose.md).              |
| `docker-entrypoint.sh`|    705 | Container entrypoint shell script. See [../files/docker-entrypoint.md](../files/docker-entrypoint.md). |
| `flake.lock`          |   2425 | Nix flake lockfile. Covered alongside [../files/flake-nix.md](../files/flake-nix.md).           |
| `flake.nix`           |   2284 | Nix flake definition. See [../files/flake-nix.md](../files/flake-nix.md).                       |
| `go.mod`              |   6364 | Go module manifest. Module path: `github.com/steveyegge/gastown`. See [../files/go-mod.md](../files/go-mod.md). |
| `go.sum`              |  29634 | Go module checksum database. Covered alongside [../files/go-mod.md](../files/go-mod.md).        |
| `renovate.json`       |   1529 | Renovate bot configuration.                                                                    |
| `.dockerignore`       |    147 | Docker build context ignore patterns.                                                          |
| `.gitattributes`      |      1 | One-byte file (essentially empty).                                                             |
| `.gitignore`          |   1375 | Git ignore patterns.                                                                           |
| `.golangci.yml`       |   3357 | golangci-lint configuration. See [../files/golangci-yml.md](../files/golangci-yml.md).          |
| `.goreleaser.yml`     |   4460 | GoReleaser release configuration. See [../files/goreleaser-yml.md](../files/goreleaser-yml.md). |

## Top-level directories (15)

| directory        | immediate children | notes                                                                                                                                 |
|------------------|-------------------:|---------------------------------------------------------------------------------------------------------------------------------------|
| `cmd/`           |                  3 | Go entry points for the three built binaries (`gt`, `gt-proxy-client`, `gt-proxy-server`). See [go-packages.md](go-packages.md).     |
| `docs/`          |                 19 | Upstream documentation tree. 60 files total when fully recursive. See [docs-tree.md](docs-tree.md).                                   |
| `gt-model-eval/` |                 10 | Model-evaluation subdirectory. Content not yet enumerated.                                                                            |
| `internal/`      |                 67 | Go packages (unexported). 67 immediate subdirectories, each a package. See [go-packages.md](go-packages.md).                          |
| `npm-package/`   |                  6 | npm-package wrapper. Content not yet enumerated.                                                                                      |
| `plugins/`       |                 14 | Plugin subdirectories. See [go-packages.md](go-packages.md) for Go file counts — most entries are empty or near-empty.                |
| `scripts/`       |                 16 | Shell/utility scripts. Content not yet enumerated.                                                                                    |
| `templates/`     |                  3 | Template directories (one known to be `agents/`). Content not yet enumerated.                                                         |
| `.beads/`        |                  5 | Beads issue-tracker scaffolding. Part of upstream repo (upstream also uses bd).                                                       |
| `.claude/`       |                  2 | Claude Code configuration.                                                                                                            |
| `.git/`          |                 11 | Git internals. Out of scope.                                                                                                          |
| `.github/`       |                  4 | GitHub-specific metadata (issue templates, workflows, etc.). Content not yet enumerated.                                              |
| `.githooks/`     |                  2 | Git hook scripts.                                                                                                                     |
| `.opencode/`     |                  2 | opencode.ai (or similar) agent configuration.                                                                                         |
| `.runtime/`      |                  1 | Purpose unknown. One child. Content not yet enumerated.                                                                               |

## Totals

- **24 top-level files** (4 of which are dotfile config: `.dockerignore`, `.gitattributes`, `.gitignore`, `.golangci.yml`, `.goreleaser.yml` → 5 dotfiles; 19 non-dotfile files).
- **15 top-level directories**, 8 visible + 7 hidden.
- **60 documentation files** total under `docs/` (recursive).
- **~67 Go packages** under `internal/` + 3 under `cmd/` + 14 under `plugins/` = **~84 package locations**, though many `plugins/` entries are empty.
