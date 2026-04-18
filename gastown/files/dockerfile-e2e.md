---
title: Dockerfile.e2e
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/Dockerfile.e2e
  - /home/kimberly/repos/gastown/Dockerfile
  - /home/kimberly/repos/gastown/Makefile
tags: [file, docker, testing, e2e]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [failure-modes]
---

# Dockerfile.e2e

Isolated container recipe for running the `TestInstall*` integration
tests, at `/home/kimberly/repos/gastown/Dockerfile.e2e`. Built and run
exclusively by the [Makefile](makefile.md)'s `test-e2e-container`
target — the Makefile comment at line 145 calls this "the only
supported way to run" e2e tests.

Unlike the primary [Dockerfile](dockerfile.md) (which builds a long-
lived sandbox for interactive use), this image's CMD is `go test`.
It builds, runs, and exits.

## What it actually does

### Purpose (from header comment)

`Dockerfile.e2e:1-10` documents the motivation:

- No interference from host `tmux` sessions
- Clean filesystem without existing Gas Town installations
- Independent beads daemon and Dolt server

Usage block in the same comment:

```
docker build -f Dockerfile.e2e -t gastown-test .
docker run --rm gastown-test
```

These are also the exact commands the Makefile runs (`Makefile:148-166`).

### Base image

`Dockerfile.e2e:12`:

```
FROM golang:1.26-alpine
```

Note the divergence: e2e uses **Go 1.26** on Alpine, while the primary
[Dockerfile](dockerfile.md) uses Go 1.25.6 on the Claude Code sandbox
base. This page does not adjudicate the mismatch — see Notes.

### Build arguments

`Dockerfile.e2e:14-15`:

```
ARG DOLT_VERSION=1.82.4
ARG BD_VERSION=v0.57.0
```

Both pinned, unlike the primary Dockerfile which installs `bd` from
`main` and `dolt` from `latest`.

### Apk install

`Dockerfile.e2e:19-28`:

```
RUN apk add --no-cache \
    git \
    bash \
    sqlite \
    build-base \
    zstd-dev \
    icu-dev \
    procps \
    lsof \
    tmux
```

Comments at lines 17-18 explain two non-obvious additions: CGO build
requirements (`build-base`, `zstd-dev`, `icu-dev`) are needed for the
beads daemon build, and `procps`/`lsof` are required by
[`gt dolt`](../commands/dolt.md) `start` for process verification.

### Beads daemon install

`Dockerfile.e2e:31-35` — retry loop (up to 3 attempts, 2s sleep)
around `go install github.com/steveyegge/beads/cmd/bd@${BD_VERSION}`.
Exits non-zero if all attempts fail.

### Dolt install

`Dockerfile.e2e:38-46` — retry loop (up to 3 attempts, 2s sleep)
that clones `dolthub/dolt` at tag `v${DOLT_VERSION}` (shallow clone)
and runs `go install ./cmd/dolt` from `dolt/go/`. Cleans up `/tmp/dolt`
on success. Comment: "pinned for stability".

### Git and Dolt identity configuration

`Dockerfile.e2e:49-53`:

```
RUN git config --global user.name "Test User" && \
    git config --global user.email "test@test.com" && \
    git config --global init.defaultBranch main && \
    dolt config --global --add user.name "Test User" && \
    dolt config --global --add user.email "test@test.com"
```

Test-only identity. [`gt install --git`](../commands/install.md)
requires valid git identity to run, and `dolt init` requires dolt
identity — these lines make both available inside the container so
the tests don't need host credentials.

### Workspace setup

`Dockerfile.e2e:56`:

```
WORKDIR /app
```

### Module download layer

`Dockerfile.e2e:59-64` — copies `go.mod` and `go.sum` first and runs
`go mod download` (with a 3-attempt retry) **before** copying the
source. Standard layer-caching pattern: dependency changes bust this
layer, source-only changes don't. Cross-reference with
[go.mod](go-mod.md).

### Source copy

`Dockerfile.e2e:67`:

```
COPY . .
```

### Build step

`Dockerfile.e2e:70`:

```
RUN go build -ldflags "-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1" -o /usr/local/bin/gt ./cmd/gt
```

Single `go build`, not `make build`. Sets only `BuiltProperly=1` via
ldflags — no `Version`, no `Commit`, no `BuildTime`. This is the
minimum needed to bypass the self-kill check at
`/home/kimberly/repos/gastown/internal/cmd/root.go:96-106`. Since the
binary goes to `/usr/local/bin/gt`, it's on `$PATH` automatically.
Cross-reference: [../binaries/gt.md](../binaries/gt.md).

### CMD

`Dockerfile.e2e:74`:

```
CMD ["go", "test", "-tags=e2e", "-timeout=5m", "-v", "-count=1", "-parallel", "1", "-run", "TestInstall", "./internal/cmd/..."]
```

Runs every Go test function whose name starts with `TestInstall` in
`internal/cmd/...`, with the `e2e` build tag enabled. Flags:

- `-tags=e2e` — selects tests gated by `//go:build e2e`.
- `-timeout=5m` — 5-minute total cap.
- `-v` — verbose output.
- `-count=1` — disables test result caching (per Dockerfile comment
  line 73).
- `-parallel 1` — forces sequential execution. Comment: "sequential
  execution".
- `-run TestInstall` — name-anchored filter.

### No ENTRYPOINT

The Dockerfile does not set an ENTRYPOINT, so the CMD runs under
Docker's default. When `docker run --rm gastown-test` exits, the
container is removed.

## Differences from the primary Dockerfile

| aspect                   | [Dockerfile](dockerfile.md)                                                   | this file                                                                    |
|--------------------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| Base image               | `docker/sandbox-templates:claude-code` (Debian-ish)                           | `golang:1.26-alpine`                                                         |
| Go version               | 1.25.6 via curl tarball                                                       | 1.26 from base image                                                         |
| User                     | `agent` (non-root after root install phase)                                   | `root` throughout (no `USER` directive)                                      |
| Build path               | `make build` — produces `gt`, `gt-proxy-server`, `gt-proxy-client`            | single `go build` — produces only `gt`                                       |
| LDFLAGS                  | Full set from [Makefile](makefile.md):18-22 including Version/Commit/BuildTime | Only `-X internal/cmd.BuiltProperly=1`                                       |
| bd install               | `curl \| bash` from `main` (unpinned)                                         | `go install ...@v0.57.0` in a retry loop                                     |
| Dolt install             | `curl \| bash` from `latest` (unpinned)                                       | Git clone + `go install` at `v1.82.4`                                        |
| Entrypoint               | `tini -- /app/docker-entrypoint.sh` running `gt install`                      | None — CMD runs directly                                                     |
| Workload                 | `sleep infinity` (interactive sandbox)                                        | `go test ./internal/cmd/...` (run once, exit)                                |
| Exit behavior            | Long-running, kept alive by compose                                           | One-shot, removed by `--rm`                                                  |
| Identity setup           | From `GIT_USER`/`GIT_EMAIL` env at runtime (in the entrypoint)                | Hardcoded `Test User` / `test@test.com` at build time                        |

## Cross-references

- [Makefile](makefile.md) — builds and runs this file via the
  `test-e2e-container` target.
- [Dockerfile](dockerfile.md) — sibling primary Dockerfile. See
  "Differences" table above.
- [../binaries/gt.md](../binaries/gt.md) — the `gt` binary produced;
  this file uses the `BuiltProperly=1` ldflag to bypass the self-kill
  check.
- [go.mod](go-mod.md) — copied into the image alongside `go.sum` for
  the `go mod download` cache layer.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Failure modes

### Precondition violations (what does it assume?)

- **Network access for dependency installs:** `Dockerfile.e2e:31-35,38-46,60-64` — three separate `for i in 1 2 3` retry loops for `bd`, `dolt`, and `go mod download`. Each retries 3 times with 2-second sleeps. If all three attempts fail (e.g., network outage, GitHub rate limit), the build exits. **Present** — retry logic with clear failure after 3 attempts.
- **`BD_VERSION` / `DOLT_VERSION` version skew:** `Dockerfile.e2e:14-15` pins `BD_VERSION=v0.57.0` and `DOLT_VERSION=1.82.4`, but `go.mod` requires `beads v0.63.3`. The pinned `bd` binary is older than the Go library. **Absent** — no build-time check for version consistency between the binary and library.
- **BuiltProperly ldflag set manually:** `Dockerfile.e2e:70` — `go build -ldflags "-X ...BuiltProperly=1"` bypasses the self-kill check, but does NOT set `Version`, `Commit`, or `BuildTime` (unlike `make build`). **Absent** — tests run with `Version="1.0.0"` and `Build="dev"` defaults, which may not match production behavior.

### Silent suppression (what errors are swallowed?)

- **Tests run as root:** No `USER` directive — all tests run as root. Any test that checks permission errors or non-root behavior will get different results than production. **Absent** — no documentation or guard for root-vs-user behavioral differences.

## Notes / open questions

- Base image Go version (1.26) is **ahead** of [go.mod](go-mod.md)'s
  declared `go 1.25.8`, while the primary [Dockerfile](dockerfile.md)'s
  pinned `GO_VERSION=1.25.6` is **behind** it. Three different Go
  versions in three places — which is authoritative?
- No `USER` switch in this Dockerfile — everything runs as root,
  including the test suite. Any test that assumes non-root behavior
  would need rewriting.
- `DOLT_VERSION=1.82.4` is pinned here but the production image pulls
  `latest`. E2e tests could pass while production breaks on a new
  Dolt release.
- `BD_VERSION=v0.57.0` — [go.mod](go-mod.md) at line 18 requires
  `github.com/steveyegge/beads v0.63.3`. The pinned `bd` binary is
  older than the Go library the tests link against. Whether this
  matters depends on beads' binary/library ABI compatibility.
- Only `TestInstall*` tests are run. Other e2e tests (if any) under
  `//go:build e2e` elsewhere in the tree are not executed by this
  image.
