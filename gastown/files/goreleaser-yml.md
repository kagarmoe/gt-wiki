---
title: .goreleaser.yml
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/.goreleaser.yml
tags: [file, release, goreleaser, build, ldflags]
phase3_audited: 2026-04-14
phase3_findings: [drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# .goreleaser.yml

GoReleaser configuration for producing release artifacts at
`/home/kimberly/repos/gastown/.goreleaser.yml`. Drives the GitHub
release workflow that publishes binaries, archives, and changelog
entries. Alternative build path to [Makefile](makefile.md),
[Dockerfile](dockerfile.md), and [flake.nix](flake-nix.md) —
produces the [gt](../binaries/gt.md) binary for six
os/arch combinations with its own LDFLAGS.

Confirmed this is the page where **`Build={{.ShortCommit}}`** gets set
— which means every release binary satisfies the
`Build != "dev"` branch of the self-kill check at
`/home/kimberly/repos/gastown/internal/cmd/root.go:96-106`. There is
**no `BuiltProperly` flag** in any of the builds here; release binaries
rely on `Build` being non-`"dev"` instead.

## What it actually does

### Header

`.goreleaser.yml:1-4`:

```yaml
# GoReleaser configuration for Gas Town (gt)
# See https://goreleaser.com for documentation

version: 2
```

GoReleaser config schema version 2.

### Before hooks

`.goreleaser.yml:6-9`:

```yaml
before:
  hooks:
    - go mod tidy
```

Runs `go mod tidy` before any build starts. This means
[go.mod](go-mod.md) and `go.sum` may be rewritten during a release
build — the release process is mildly destructive to the working
tree.

### Builds section

`.goreleaser.yml:11-111`. Six builds, all against `./cmd/gt` with
`binary: gt`. Each has its own `env:`, `goos:`, `goarch:`, and
`ldflags:` blocks. Build IDs and their notable env vars:

| id                  | lines        | goos    | goarch | CGO_ENABLED | cross-compiler env                                                   |
|---------------------|--------------|---------|--------|-------------|----------------------------------------------------------------------|
| `gt-linux-amd64`    | 12-26        | linux   | amd64  | 1           | —                                                                    |
| `gt-linux-arm64`    | 28-44        | linux   | arm64  | 1           | `CC=aarch64-linux-gnu-gcc`, `CXX=aarch64-linux-gnu-g++`              |
| `gt-darwin-amd64`   | 46-60        | darwin  | amd64  | 1           | —                                                                    |
| `gt-darwin-arm64`   | 62-76        | darwin  | arm64  | 1           | —                                                                    |
| `gt-windows-amd64`  | 78-95        | windows | amd64  | 1           | `CC=x86_64-w64-mingw32-gcc`, `CXX=x86_64-w64-mingw32-g++`            |
| `gt-freebsd-amd64`  | 97-111       | freebsd | amd64  | **0**       | —                                                                    |

**CGO_ENABLED=0** on the FreeBSD build only. Every other target
requires CGO. The FreeBSD build therefore produces a different binary
surface — CGO-dependent functionality (Dolt's embedded server, the
`go-icu-regex` transitive dep used by beads) is unavailable.

The Windows build adds `-buildmode=exe` to ldflags
(`.goreleaser.yml:95`) in addition to the standard set.

#### LDFLAGS shared across all six builds

Every build's `ldflags:` list has the same five entries:

```yaml
- -s -w
- -X github.com/steveyegge/gastown/internal/cmd.Version={{.Version}}
- -X github.com/steveyegge/gastown/internal/cmd.Build={{.ShortCommit}}
- -X github.com/steveyegge/gastown/internal/cmd.Commit={{.Commit}}
- -X github.com/steveyegge/gastown/internal/cmd.Branch={{.Branch}}
```

Four variables are populated from GoReleaser template functions:

- `Version` → the release version (e.g. `0.8.0`, derived from the git
  tag).
- `Build` → `{{.ShortCommit}}` — the short git SHA. **This is the
  self-kill bypass.** Per [../binaries/gt.md](../binaries/gt.md)'s
  ldflag table, `Build != "dev"` skips the self-kill check. A short
  commit SHA is always non-`"dev"`, so release binaries never need
  `BuiltProperly`.
- `Commit` → the full git SHA.
- `Branch` → the release branch.

**No `BuiltProperly`** is set by any build. This contrasts with
[Makefile](makefile.md) (sets `BuiltProperly=1` explicitly) and
[flake.nix](flake-nix.md) (sets both `Build=nix` and `BuiltProperly=1`
defensively).

**No `BuildTime`** either — neither the Makefile's `BuildTime` field
nor any equivalent appears here. Release binaries report no build
timestamp.

### Archives section

`.goreleaser.yml:114-123`:

```yaml
archives:
  - id: gt-archive
    format: tar.gz
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    format_overrides:
      - goos: windows
        format: zip
    files:
      - LICENSE
      - README.md
```

- Default format: `tar.gz`.
- Windows override: `zip`.
- Name pattern: `gastown_<version>_<os>_<arch>.(tar.gz|zip)`.
- Each archive bundles the binary plus the repo's `LICENSE` and
  `README.md`.

### Checksum

`.goreleaser.yml:125-127`:

```yaml
checksum:
  name_template: "checksums.txt"
  algorithm: sha256
```

Writes a single `checksums.txt` with SHA256 digests of all archives.

### Snapshot

`.goreleaser.yml:129-130`:

```yaml
snapshot:
  version_template: "{{ incpatch .Version }}-next"
```

For non-release snapshot builds, the version becomes
`<current>-next` where `<current>` has its patch number incremented.

### Changelog

`.goreleaser.yml:132-149`. Auto-generated release notes config:

- `sort: asc`
- **Excludes** commits whose message matches:
  - `^docs:`
  - `^test:`
  - `^chore:`
  - `Merge pull request`
  - `Merge branch`
- **Groups** commits into:
  1. "Features" — matches `^.*feat(\(\w+\))?:.*$`
  2. "Bug Fixes" — matches `^.*fix(\(\w+\))?:.*$`
  3. "Others" — catch-all, order 999

This is standard conventional-commits changelog shaping.

### Release section

`.goreleaser.yml:151-176`:

```yaml
release:
  github:
    owner: steveyegge
    name: gastown
  draft: false
  prerelease: auto
  name_template: "v{{.Version}}"
  header: |
    ## Gas Town v{{.Version}}
    ...
```

- Target repo: `github.com/steveyegge/gastown`.
- `draft: false` — releases publish directly (not created as drafts).
- `prerelease: auto` — GoReleaser infers prerelease status from the
  version string.
- Release name: `v{{.Version}}`.
- Header is a literal Markdown block (`.goreleaser.yml:158-176`) that
  describes installation:
  - Homebrew: `brew install gastown`
  - npm: `npm install -g @gastown/gt`
  - Manual: download from the release page

### Announce

`.goreleaser.yml:178-180`:

```yaml
announce:
  skip: false
```

Announcements are enabled but no specific announcer is configured in
this file — GoReleaser defaults apply (Twitter/Mastodon/etc. are all
off unless explicitly configured, so this is effectively a no-op).

### What is NOT in this file

- **No `brews:` block.** The release header advertises
  `brew install gastown`, but this file does not define a Homebrew
  formula. Either the formula lives in a separate tap repository
  (standard pattern for homebrew-core) or the install message is
  aspirational. This is a real gap against
  [../binaries/gt.md](../binaries/gt.md), which states "Homebrew sets
  `Build` to `Homebrew`" — the mechanism by which that happens is
  **not in this file**.
- **No `nfpms:` / `snapcrafts:` / `dockers:` blocks.** Only raw
  archive releases.
- **No `upx:` / `notarize:` blocks.** No macOS notarization
  configured, which means the darwin builds are subject to macOS
  Gatekeeper warnings on first run — the same issue the self-kill
  check exists to warn about.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — every ldflag set by this
  file corresponds to a variable documented there.
- [Makefile](makefile.md) — sibling build path for dev builds. Sets
  `BuiltProperly=1` instead of relying on `Build != "dev"`.
- [flake.nix](flake-nix.md) — sibling build path. Sets `Build=nix`
  (same gate-bypass pattern as this file) AND `BuiltProperly=1`
  (redundantly).
- [Dockerfile](dockerfile.md) — sibling build path for the primary
  container; runs `make build` internally.
- [go.mod](go-mod.md) — `go mod tidy` is run before every release
  build via `before.hooks`.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Docs claim

### Source
- `/home/kimberly/repos/gastown/.goreleaser.yml` (lines 164-167)

### Verbatim
> **Homebrew (macOS/Linux):**
> ```bash
> brew install gastown
> ```

## Drift

### Release header advertises `brew install gastown` but no `brews:` block exists

- **Claim source:** `.goreleaser.yml:164-167` — release header template
  tells users to run `brew install gastown`.
- **Docs claim:** "Homebrew (macOS/Linux): `brew install gastown`"
- **Code does:** The `.goreleaser.yml` file defines no `brews:` block.
  GoReleaser cannot produce a Homebrew formula without this block.
  No separate `homebrew-tap` repository has been identified in the
  gastown GitHub org. The release header promise is unsubstantiated
  by any mechanism in this file.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — either add a `brews:` block to goreleaser
  (making the claim true) or remove the brew install line from the
  release header. The README also advertises Homebrew install.
- **Release position:** `in-release`

See also: [gastown/drift/README.md](../drift/README.md) (when the
consolidated index exists).

## Notes / open questions

- → Homebrew formula location question promoted to `## Drift` above.
- `npm-package/` exists at `/home/kimberly/repos/gastown/npm-package/`
  per [../inventory/repo-root.md](../inventory/repo-root.md), but the
  release flow here doesn't reference it. The npm release path
  appears to be separate from GoReleaser.
- `go mod tidy` as a pre-hook means release builds rewrite `go.sum` /
  `go.mod`. If a release is built from a dirty or partially-updated
  module state, the working tree is mutated.
- The darwin builds require CGO but aren't notarized. Users installing
  from raw downloads will hit Gatekeeper. This is the scenario the
  self-kill check warns about — yet release binaries bypass the
  self-kill check via `Build != "dev"`, so they run without warning
  users.
- FreeBSD build disables CGO. Any CGO-requiring feature (Dolt server
  embed, full unicode regex) is silently absent on FreeBSD.
- No `dockers:` block — the release process does not publish a Docker
  image. The primary [Dockerfile](dockerfile.md) build is separate
  from releases.
