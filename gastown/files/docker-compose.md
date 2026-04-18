---
title: docker-compose.yml
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/docker-compose.yml
  - /home/kimberly/repos/gastown/Dockerfile
  - /home/kimberly/repos/gastown/docker-entrypoint.sh
tags: [file, docker, compose, sandbox]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# docker-compose.yml

Compose specification for running the primary [Dockerfile](dockerfile.md)
image as a long-lived sandbox container. Lives at
`/home/kimberly/repos/gastown/docker-compose.yml`. Defines one service
(`gastown`) plus two named volumes.

## What it actually does

### Invocation

`docker-compose.yml:1-2` — the header comment spells out the expected
command:

```
# Run with
# GIT_USER="<your name>" GIT_EMAIL="<your email>" FOLDER="/Users/you/code" docker compose up -d
```

Three environment variables must be supplied: `GIT_USER`, `GIT_EMAIL`,
and `FOLDER`. `FOLDER` is the host directory that gets mounted into
`/gt` (the Gas Town workspace root).

### No top-level `version:` key

`docker-compose.yml:4` starts directly at `services:`. The file omits
the deprecated top-level `version:` field entirely, which Compose v2
ignores anyway.

### `services.gastown` — single service definition

The only service is `gastown`, defined at `docker-compose.yml:5-38`.

#### `build:` block (`docker-compose.yml:6-8`)

```yaml
build:
  context: .
  dockerfile: Dockerfile
```

Builds from the repo root using [Dockerfile](dockerfile.md) — NOT
[Dockerfile.e2e](dockerfile-e2e.md). There is no `image:` field, so
the image has no explicit tag.

#### `container_name: gastown-sandbox` (`docker-compose.yml:9`)

Fixed container name. Only one of these can run per host.

#### STDIN/TTY (`docker-compose.yml:10-11`)

```yaml
stdin_open: false
tty: false
```

Both off. You don't attach directly — the expected interaction is
`docker compose exec gastown zsh` (comment at line 40).

#### Security hardening (`docker-compose.yml:12-22`)

```yaml
security_opt:
  - no-new-privileges:true
cap_drop:
  - ALL
cap_add:
  - CHOWN
  - SETUID
  - SETGID
  - DAC_OVERRIDE
  - FOWNER
  - NET_RAW
```

- `no-new-privileges:true` — blocks setuid/setgid escalation inside
  the container.
- `cap_drop: ALL` then `cap_add:` of six specific capabilities. This
  is the standard Linux-capability-allowlist pattern. `NET_RAW` is
  the one that allows `ping` and raw sockets; the others are
  filesystem ownership/permission primitives needed by the container's
  install flow and package managers.

#### Environment (`docker-compose.yml:23-26`)

```yaml
environment:
  IS_SANDBOX: 1
  GIT_USER: ${GIT_USER:-TestUser}
  GIT_EMAIL: ${GIT_EMAIL:-test@example.com}
```

- `IS_SANDBOX=1` — used internally by [gt](../binaries/gt.md) to
  detect the sandboxed runtime. Purpose not yet mapped in this wiki.
- `GIT_USER` / `GIT_EMAIL` — consumed by
  [docker-entrypoint.sh](docker-entrypoint.md):6-12 to set global git
  and dolt identity on every container start. Default values
  (`TestUser` / `test@example.com`) fire when the user didn't export
  the real ones.

#### Volumes (`docker-compose.yml:27-35`)

```yaml
volumes:
  - agent-home:/home/agent
  - ${FOLDER}:/gt
  - dolt-data:/gt/.dolt-data
```

Three mounts:

1. `agent-home:/home/agent` — named volume for the agent user's home
   directory. Persists across container lifetime (caches, shell
   history, etc.).
2. `${FOLDER}:/gt` — **bind mount** from the host. The
   `FOLDER` env var must be set; the comment at lines 28-30 tells
   you to put it in `.env` or export it on the command line. This is
   where the user's actual work lives.
3. `dolt-data:/gt/.dolt-data` — named volume for the Dolt data
   directory, **nested inside** the bind mount. Comment at
   `docker-compose.yml:33-35` explains why:

   > Dolt data on a proper ext4 volume (not the macOS bind mount) to
   > avoid journal corruption from VirtioFS fsync semantics.

   The later volume mount overrides the earlier bind mount at that
   subpath — an intentional Docker Compose pattern to keep the Dolt
   data off VirtioFS without losing the host bind mount for everything
   else.

#### Ports (`docker-compose.yml:36-37`)

```yaml
ports:
  - "${DASHBOARD_PORT:-8080}:8080"
```

Publishes container port 8080 to host port `DASHBOARD_PORT` (default
8080). This is presumably the [`gt dashboard`](../commands/dashboard.md)
HTTP listener, which defaults to port 8080.

#### Command (`docker-compose.yml:38`)

```yaml
command: sleep infinity
```

Overrides the Dockerfile's `CMD ["sleep", "infinity"]`. Same effect —
the container stays running so you can exec into it. The dockerfile's
ENTRYPOINT is `tini -- /app/docker-entrypoint.sh`
([Dockerfile](dockerfile.md):55), which runs once and then
`exec "$@"`s into `sleep infinity`.

### Volumes block (`docker-compose.yml:42-44`)

```yaml
volumes:
  agent-home:
  dolt-data:
```

Two empty named-volume definitions. Compose creates them on first
`docker compose up` with default driver/options.

### No `networks:` block

No custom networks. The service uses the default bridge network that
Compose creates per project.

## Cross-references

- [Dockerfile](dockerfile.md) — the build target this compose file
  references via `build:`.
- [docker-entrypoint.sh](docker-entrypoint.md) — the init script
  invoked by the Dockerfile's `ENTRYPOINT`; consumes `GIT_USER` and
  `GIT_EMAIL` from this file's `environment:` block.
- [../binaries/gt.md](../binaries/gt.md) — the binary baked into the
  image by `make build` inside the Dockerfile.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Notes / open questions

- The [`gt dashboard`](../commands/dashboard.md) page now confirms the
  default port is 8080, matching the compose file's hardcoded mapping.
- `IS_SANDBOX=1` is consumed somewhere in [gt](../binaries/gt.md).
  `grep -rn IS_SANDBOX` in `internal/` would locate the consumer —
  follow-up.
- The `${FOLDER}` bind mount is **required** — there's no default. A
  user who `docker compose up` without exporting `FOLDER` gets a
  compose error. The header comment is the only documentation of this
  requirement inside the file.
- `cap_add` includes `DAC_OVERRIDE` and `FOWNER`, which are substantial
  filesystem privileges. Probably needed so the entrypoint's `gt
  install --git --force` can operate on a bind-mounted host directory
  with unexpected ownership.
- No `restart:` policy. A crashed container stays down until manually
  restarted.
- No `healthcheck:` block.
