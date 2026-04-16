---
title: Dockerfile
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/Dockerfile
  - /home/kimberly/repos/gastown/docker-entrypoint.sh
  - /home/kimberly/repos/gastown/docker-compose.yml
tags: [file, docker, container, sandbox]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# Dockerfile

The primary production container recipe, at
`/home/kimberly/repos/gastown/Dockerfile`. Builds a single-image sandbox
that runs [gt](../binaries/gt.md) under `tini` + a small init script,
with Go 1.25.6 installed from the official tarball and `bd`/`dolt`
installed from their upstream shell installers. This image is the one
referenced by [docker-compose.yml](docker-compose.md).

**Also known as:** "the primary Dockerfile", "production image",
"`gastown:latest`" (the invocation comment at line 2).

## What it actually does

### Invocation

`Dockerfile:1-2` — leading comment spells out the expected command:

```
# Run with
# docker build -t gastown:latest -f Dockerfile .
```

### Base image

`Dockerfile:3`:

```
FROM docker/sandbox-templates:claude-code
```

The base is Anthropic's Claude Code sandbox template. It ships with
`agent` as the default non-root user. No multi-stage — this is a single
stage build.

### Build argument

`Dockerfile:5` — `ARG GO_VERSION=1.25.6`. The only build-time argument.
Used later to fetch the Go tarball. Note: this is pinned to 1.25.6 while
[go.mod](go-mod.md) declares `go 1.25.8` and [flake.nix](flake-nix.md)
overlays Go to 1.25.8. The container image is one patch behind.

### Root-phase system package install

`Dockerfile:7` switches to `USER root`.

`Dockerfile:10-22` — `apt-get update && apt-get install -y`:

- `build-essential` — C/C++ toolchain for CGO builds.
- `git`
- `sqlite3`
- `tmux` — required by Gas Town's session layer.
- `curl`
- `ripgrep`
- `zsh` — the default login shell inside the sandbox.
- `gh` — GitHub CLI.
- `netcat-openbsd`
- `tini` — used as PID 1 via the ENTRYPOINT.
- `vim`

Followed by `rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*` to
shrink the image.

### Go installation

`Dockerfile:25-26`:

```
RUN ARCH=$(dpkg --print-architecture) && \
    curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" | tar -C /usr/local -xz
```

Pulls the official Go tarball for the host architecture and unpacks it
to `/usr/local/go`. The Dockerfile comment at line 24 explains the
reason: "apt golang-go is too old".

### PATH environment

`Dockerfile:27`:

```
ENV PATH="/app/gastown:/usr/local/go/bin:/home/agent/go/bin:${PATH}"
```

`/app/gastown` is where the source tree is copied later (and where
`make build` drops its binaries). `/home/agent/go/bin` is the agent
user's GOPATH bin dir. Both are prepended to the existing base-image
PATH.

### Beads and Dolt installation

`Dockerfile:30-31`:

```
RUN curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
RUN curl -fsSL https://github.com/dolthub/dolt/releases/latest/download/install.sh | bash
```

Runs the upstream install scripts as root. Neither is pinned — `main`
for beads, `latest` for dolt. This image will drift over time as
upstream releases change.

### Directory setup

`Dockerfile:34`:

```
RUN mkdir -p /app /gt /gt/.dolt-data && chown -R agent:agent /app /gt
```

Creates `/app` (source tree location), `/gt` (Gas Town workspace, the
runtime equivalent of `~/gt/`), and `/gt/.dolt-data` (Dolt data dir).
Ownership handed to the `agent` user.

### Shell environment exports

`Dockerfile:37-42` — appends three env vars to both `/etc/profile.d/`
(for bash) and `/etc/zsh/zshenv` (for zsh):

- `PATH="/app/gastown:$PATH"` — ensures the built binaries are visible
  to both shells, even outside the Dockerfile's `ENV PATH`.
- `COLORTERM="truecolor"` — enables 24-bit color for styled output
  (used by bubbletea / lipgloss rendering).
- `TERM="xterm-256color"` — standard terminal type.

### User switch

`Dockerfile:44`:

```
USER agent
```

All remaining steps run as the unprivileged `agent` user.

### Source copy and build

`Dockerfile:46-48`:

```
COPY --chown=agent:agent . /app/gastown
RUN cd /app/gastown && make build
```

The entire repository is copied into `/app/gastown` with agent
ownership, then `make build` is invoked. This is the critical
cross-reference with [../binaries/gt.md](../binaries/gt.md): the
container uses the **Makefile** path, so the resulting binary has
`BuiltProperly=1` set via ldflags (per
[Makefile](makefile.md):18-22). The self-kill check at
`/home/kimberly/repos/gastown/internal/cmd/root.go:96-106` is
satisfied.

### Entrypoint copy

`Dockerfile:50-51`:

```
COPY --chown=agent:agent docker-entrypoint.sh /app/docker-entrypoint.sh
RUN chmod +x /app/docker-entrypoint.sh
```

Copies the init script separately (after the `RUN make build` layer)
so that changes to the entrypoint script don't bust the build cache.
See [docker-entrypoint.sh](docker-entrypoint.md) for line-by-line
coverage.

### Workdir

`Dockerfile:53`:

```
WORKDIR /gt
```

The container starts in the Gas Town workspace mount point.

### ENTRYPOINT and CMD

`Dockerfile:55-56`:

```
ENTRYPOINT ["tini", "--", "/app/docker-entrypoint.sh"]
CMD ["sleep", "infinity"]
```

`tini` runs as PID 1 and execs the entrypoint script, which itself
`exec "$@"` at the end (`docker-entrypoint.sh:22`), so `tini` ends up
supervising whatever `CMD` resolves to. The default `CMD` is
`sleep infinity`, matching the compose file's `command: sleep
infinity` (`docker-compose.yml:38`). The container is intended to be
kept running while humans / agents `docker compose exec` into it.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — the built binary; the
  `make build` step here produces it with the right ldflag gates.
- [Makefile](makefile.md) — the build recipe invoked inside the image.
- [docker-entrypoint.sh](docker-entrypoint.md) — the script referenced
  by the `ENTRYPOINT` line; receives `CMD` args via `exec "$@"`.
- [docker-compose.yml](docker-compose.md) — the compose file that
  builds this Dockerfile by path and mounts external volumes around
  the resulting image.
- [Dockerfile.e2e](dockerfile-e2e.md) — the sibling e2e test
  Dockerfile. Different base image, different build strategy, different
  purpose.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Notes / open questions

- The base image `docker/sandbox-templates:claude-code` is an Anthropic
  artifact. Pinning / versioning / provenance of that base is out of
  scope here but worth a follow-up.
- `bd` is installed via `curl | bash` against `main`, not a pinned tag.
  Same for `dolt` (`latest`). Reproducibility depends on upstream being
  stable.
- `GO_VERSION=1.25.6` disagrees with [go.mod](go-mod.md)'s `go 1.25.8`.
  If any of the modules in the build use a language feature from
  1.25.7 or 1.25.8, the container build would fail. That it currently
  works implies either a tolerant module or that the go.mod version
  directive is advisory.
- The final CMD is `sleep infinity` — there is no "start Gas Town"
  command baked in. The entrypoint script only runs
  [`gt install`](../commands/install.md); the user is expected to
  `docker compose exec gastown zsh` and drive the sandbox interactively.
- No `HEALTHCHECK` directive. No `EXPOSE`.
