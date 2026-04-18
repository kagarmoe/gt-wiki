---
title: go.mod
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/go.mod
tags: [file, go, modules, dependencies]
phase3_audited: 2026-04-14
phase3_findings: [wiki-stale, drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# go.mod

Go module manifest at `/home/kimberly/repos/gastown/go.mod`. Declares
the module path, Go version, and every direct and indirect dependency.
Companion file `/home/kimberly/repos/gastown/go.sum` (29634 bytes)
holds cryptographic checksums for the full module graph — not
transcribed here, managed by `go mod tidy`.

## What it actually does

### Module path

`go.mod:1`:

```
module github.com/steveyegge/gastown
```

The module lives under the `steveyegge` GitHub namespace. This is the
path every `import` and every `-X` ldflag (`Version`, `Build`,
`Commit`, `Branch`, `BuiltProperly`) must reference — see
[../binaries/gt.md](../binaries/gt.md) for the variables and
[Makefile](makefile.md), [.goreleaser.yml](goreleaser-yml.md),
[flake.nix](flake-nix.md), [Dockerfile.e2e](dockerfile-e2e.md) for
the per-build-path ldflag values.

### Go version

`go.mod:3`:

```
go 1.25.8
```

This is the language / toolchain version the module declares it needs.
Three build paths in the repo disagree with this value:

- [Dockerfile](dockerfile.md) pins `GO_VERSION=1.25.6` — one patch
  behind.
- [Dockerfile.e2e](dockerfile-e2e.md) uses `golang:1.26-alpine` — one
  minor ahead.
- [flake.nix](flake-nix.md) uses a Go overlay to bump nixpkgs from
  1.25.7 to 1.25.8 — matches `go.mod`.

Only the Nix path exactly matches the `go.mod` declaration.

### Direct requires

`go.mod:5-36` — 30 direct `require` entries (no `// indirect`
comment). Grouped below by rough functional category:

**CLI framework:**
- `github.com/spf13/cobra v1.10.2` (line 17) — command tree framework.

**Terminal UI (Bubble Tea ecosystem):**
- `github.com/charmbracelet/bubbles v1.0.0` (line 7)
- `github.com/charmbracelet/bubbletea v1.3.10` (line 8)
- `github.com/charmbracelet/glamour v0.10.0` (line 9) — Markdown
  rendering.
- `github.com/charmbracelet/lipgloss v1.1.1-0.20250404203927-76690c660834` (line 10)
- `github.com/muesli/termenv v0.16.0` (line 16) — terminal
  capabilities.
- `golang.org/x/term v0.41.0` (line 31)

**Beads / Dolt (data plane):**
- `github.com/steveyegge/beads v0.63.3` (line 18) — the beads library
  this project links against. Note: [flake.nix](flake-nix.md)
  overlay comment references the older `v0.60.0`.
- `github.com/go-sql-driver/mysql v1.9.3` (line 13) — Dolt server
  speaks MySQL protocol.
- `github.com/testcontainers/testcontainers-go v0.41.0` (line 20)
- `github.com/testcontainers/testcontainers-go/modules/dolt v0.41.0` (line 21) —
  Dolt testcontainer module.

**Config and file formats:**
- `github.com/BurntSushi/toml v1.6.0` (line 6)
- `gopkg.in/yaml.v3 v3.0.1` (line 35)

**Telemetry (OpenTelemetry):**
- `go.opentelemetry.io/otel v1.42.0` (line 22)
- `go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp v0.18.0` (line 23)
- `go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp v1.42.0` (line 24)
- `go.opentelemetry.io/otel/log v0.18.0` (line 25)
- `go.opentelemetry.io/otel/metric v1.42.0` (line 26)
- `go.opentelemetry.io/otel/sdk v1.42.0` (line 27)
- `go.opentelemetry.io/otel/sdk/log v0.18.0` (line 28)
- `go.opentelemetry.io/otel/sdk/metric v1.42.0` (line 29)

Seven OTEL modules pinned at consistent version pairs. Cross-reference
the `persistentPreRun` and `Execute` sequences in
[../binaries/gt.md](../binaries/gt.md) — OTEL is initialized per
`gt` invocation and exports attributes to child `bd` processes.

**Browser automation:**
- `github.com/go-rod/rod v0.116.2` (line 12) — headless browser
  driver.

**File and process utilities:**
- `github.com/fsnotify/fsnotify v1.9.0` (line 11) — filesystem
  watching.
- `github.com/gofrs/flock v0.13.0` (line 14) — file locking.
- `gopkg.in/natefinch/lumberjack.v2 v2.2.1` (line 34) — log
  rotation.

**Standard utilities:**
- `github.com/google/uuid v1.6.0` (line 15)
- `github.com/stretchr/testify v1.11.1` (line 19) — testing assertion
  library.
- `golang.org/x/sys v0.42.0` (line 30)
- `golang.org/x/text v0.35.0` (line 32)
- `golang.org/x/time v0.15.0` (line 33)

### Indirect requires

`go.mod:38-134` — one large `require (...)` block. Every entry has
`// indirect`. **95 indirect dependencies** (confirmed by
`grep -c '// indirect'`).

Not transcribed line-by-line. Worth noting a few by category:

- **Docker / containerd stack** (needed by `testcontainers-go`):
  `github.com/docker/docker`, `github.com/docker/go-connections`,
  `github.com/containerd/*`, `github.com/moby/*`,
  `github.com/opencontainers/*`.
- **gosec-excluded Spf13 viper / afero chain**:
  `github.com/spf13/viper v1.21.0`, `github.com/spf13/afero v1.15.0`,
  `github.com/spf13/pflag v1.0.10`, `github.com/spf13/cast v1.10.0`.
  These show up indirectly through beads / testcontainers.
- **Charm internal packages**:
  `github.com/charmbracelet/colorprofile`,
  `github.com/charmbracelet/x/ansi`,
  `github.com/charmbracelet/x/cellbuf`,
  `github.com/charmbracelet/x/exp/slice`,
  `github.com/charmbracelet/x/term`.
- **Gopsutil process metrics**:
  `github.com/shirou/gopsutil/v4 v4.26.2` — likely the backing for
  [`gt vitals`](../commands/vitals.md) / [`gt health`](../commands/health.md) process introspection.
- **OTEL transitive**:
  `go.opentelemetry.io/auto/sdk v1.2.1`,
  `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.61.0`,
  `go.opentelemetry.io/otel/trace v1.42.0`,
  `go.opentelemetry.io/proto/otlp v1.9.0`.
- **gRPC / protobuf** (needed by OTEL OTLP exporters):
  `google.golang.org/grpc v1.79.3`,
  `google.golang.org/protobuf v1.36.11`,
  `google.golang.org/genproto/googleapis/{api,rpc}`.
- **Ysmood / go-rod support libs**:
  `github.com/ysmood/fetchup`, `goob`, `got`, `gson`, `leakless`.

### `replace` directives

None. The file has no `replace (...)` block. Every dependency is
fetched directly from the module cache at the declared version.

### `retract` directives

None.

### Supporting file: go.sum

`/home/kimberly/repos/gastown/go.sum` is the checksum database —
29634 bytes, managed automatically by `go mod tidy` and `go mod
download`. Not transcribed. Read and verify it any time either
`go.mod` changes or a build path wants to confirm module hashes.

Build paths that specifically touch `go.sum`:

- [Dockerfile.e2e](dockerfile-e2e.md):59 copies `go.mod` and `go.sum`
  into the image and runs `go mod download` as a cache-friendly
  pre-source layer.
- [.goreleaser.yml](goreleaser-yml.md):6-9 runs `go mod tidy` as a
  pre-build hook — release builds may rewrite `go.sum`.
- [flake.nix](flake-nix.md)'s `buildGoModule` has a `vendorHash` that
  must match the module set declared in `go.mod` / `go.sum` — hash
  mismatch breaks Nix builds until manually updated.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — the module path declared
  here is the base for every ldflag in every build path.
- [Makefile](makefile.md) — invokes `go build` which reads this file.
- [Dockerfile](dockerfile.md) — builds with the Go version
  `GO_VERSION=1.25.6` (off by one from this file's `go 1.25.8`).
- [Dockerfile.e2e](dockerfile-e2e.md) — explicitly stages `go.mod` +
  `go.sum` for `go mod download` caching.
- [flake.nix](flake-nix.md) — `buildGoModule` uses a `vendorHash`
  computed from this file and `go.sum`.
- [.goreleaser.yml](goreleaser-yml.md) — runs `go mod tidy` before
  every release build.
- [.golangci.yml](golangci-yml.md) — lints code that imports from
  this dependency set.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  rows for `go.mod` and `go.sum`.
- [../inventory/go-packages.md](../inventory/go-packages.md) —
  enumeration of packages within the module (not the external
  dependencies).

## Drift

### Go version disagrees across three build paths

- **Claim source:** `go.mod:3` declares `go 1.25.8` as the module's
  required Go version. Three build paths in the same repo each use a
  different Go version.
- **Docs claim:** `go.mod:3` — `go 1.25.8`.
- **Code does:** [Dockerfile](dockerfile.md):5 pins `GO_VERSION=1.25.6`
  (one patch behind). [Dockerfile.e2e](dockerfile-e2e.md):12 uses
  `golang:1.26-alpine` (one minor ahead). [flake.nix](flake-nix.md):24
  overlays to `1.25.8` (matches).
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — the Dockerfile `GO_VERSION` should be updated
  to match `go.mod`, or `go.mod` should document which version is
  canonical. Dockerfile.e2e using 1.26 is a forward-compatibility choice
  that's less concerning.
- **Release position:** `in-release`

See also: [gastown/drift/README.md](../drift/README.md) (when the
consolidated index exists).

## Notes / open questions

- The beads library version here (`v0.63.3`) disagrees with
  [flake.nix](flake-nix.md)'s overlay comment ("beads v0.60.0 deps").
  The Nix overlay may no longer be necessary under the current beads
  version.
- 95 indirect dependencies is a large surface area. `testcontainers-go`
  alone pulls in the entire Docker / containerd Go client stack.
- `github.com/shirou/gopsutil/v4` showing up indirectly suggests some
  upstream dependency (likely beads or testcontainers) is using
  process metrics. Worth verifying where.
- `github.com/spf13/viper` is indirect even though the project uses
  config files heavily — this implies gastown's own config loading
  uses a different library (BurntSushi/toml is a direct require) and
  viper only comes in via beads or another dependency.
- No `replace` directives means no local-path overrides — every
  dependency is fetched canonically.
