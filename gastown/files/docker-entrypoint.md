---
title: docker-entrypoint.sh
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/docker-entrypoint.sh
  - /home/kimberly/repos/gastown/Dockerfile
  - /home/kimberly/repos/gastown/docker-compose.yml
tags: [file, docker, entrypoint, init, shell]
phase3_audited: 2026-04-14
phase3_findings: [wiki-stale]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [failure-modes]
---

# docker-entrypoint.sh

22-line POSIX shell script that runs as PID 1 inside the primary
container image. Lives at
`/home/kimberly/repos/gastown/docker-entrypoint.sh`. Referenced by the
primary [Dockerfile](dockerfile.md):55 as `ENTRYPOINT ["tini", "--",
"/app/docker-entrypoint.sh"]`. [Dockerfile.e2e](dockerfile-e2e.md) does
**not** use this script.

Because the file is small, this page documents every line.

## What it actually does

### Shebang and error handling

`docker-entrypoint.sh:1-2`:

```sh
#!/bin/sh
set -e
```

Runs under POSIX `/bin/sh` (not bash). `set -e` makes the script exit
on any unchecked command failure.

### Conditional git / dolt identity configuration

`docker-entrypoint.sh:4-12`:

```sh
# Re-apply git/dolt config on every start so env var changes take effect
# even when the home volume already exists from a previous run.
if [ -n "$GIT_USER" ] && [ -n "$GIT_EMAIL" ]; then
    git config --global user.name "$GIT_USER"
    git config --global user.email "$GIT_EMAIL"
    git config --global credential.helper store
    dolt config --global --add user.name "$GIT_USER"
    dolt config --global --add user.email "$GIT_EMAIL"
fi
```

Reads `GIT_USER` and `GIT_EMAIL` from the environment. Both are
supplied by [docker-compose.yml](docker-compose.md):23-26, which
defaults them to `TestUser` / `test@example.com` if the user did not
export their own. If either is unset, the entire block is skipped.

When both are set, five config writes happen:

1. `git config --global user.name`
2. `git config --global user.email`
3. `git config --global credential.helper store` — enables git to
   persist credentials to disk (HTTPS auth).
4. `dolt config --global --add user.name`
5. `dolt config --global --add user.email`

The comment at lines 4-5 explains the re-apply: the `agent-home` named
volume (`docker-compose.yml:31`) persists across container restarts,
so a container started with different `GIT_USER`/`GIT_EMAIL` would
otherwise inherit the stale values from the volume. Re-writing on
every start makes the env vars authoritative.

### Gas Town workspace initialization

`docker-entrypoint.sh:14-20`:

```sh
if [ ! -f /gt/mayor/town.json ]; then
    echo "Initializing Gas Town workspace at /gt..."
    /app/gastown/gt install /gt --git
else
    echo "Refreshing Gas Town workspace at /gt..."
    /app/gastown/gt install /gt --git --force
fi
```

Detects whether `/gt` has already been initialized by checking for the
marker file `/gt/mayor/town.json`. `/gt` is the bind mount from the
host's `$FOLDER` environment variable, defined at
[docker-compose.yml](docker-compose.md):32.

- **First run** (no marker): runs `gt install /gt --git` to initialize
  the workspace.
- **Subsequent runs** (marker present): runs `gt install /gt --git
  --force` to refresh it in place.

The `gt install` binary is invoked by absolute path at
`/app/gastown/gt` — the location where [Dockerfile](dockerfile.md):48
drops the output of `make build` via `RUN cd /app/gastown && make
build`. This means the binary **always** has `BuiltProperly=1` set, so
the self-kill check at
`/home/kimberly/repos/gastown/internal/cmd/root.go:96-106` is bypassed.

Cross-reference: [../binaries/gt.md](../binaries/gt.md) (build-time
variables table) and [Makefile](makefile.md) (LDFLAGS block).

### Exec to CMD

`docker-entrypoint.sh:22`:

```sh
exec "$@"
```

Replaces the current shell with whatever Docker is passing as the
command, preserving PID. Under the primary Dockerfile's
`ENTRYPOINT ["tini", "--", "/app/docker-entrypoint.sh"]` + `CMD
["sleep", "infinity"]`, `$@` expands to `sleep infinity`. Under
[docker-compose.yml](docker-compose.md):38's `command: sleep
infinity`, same result.

The script's lifecycle: tini starts, tini execs the script, the script
configures identity + runs `gt install`, the script execs the CMD,
tini reaps the resulting process tree.

## Cross-references

- [Dockerfile](dockerfile.md) — copies this script to
  `/app/docker-entrypoint.sh` and sets it as the ENTRYPOINT via tini.
- [docker-compose.yml](docker-compose.md) — supplies the `GIT_USER`
  and `GIT_EMAIL` environment variables and mounts `${FOLDER}` at
  `/gt`.
- [../binaries/gt.md](../binaries/gt.md) — the `gt` binary at
  `/app/gastown/gt` whose `install` subcommand this script invokes.
- [../commands/install.md](../commands/install.md) — the `gt install`
  command that this entrypoint runs to initialize/refresh the workspace
  at `/gt`; documents the `--force` refresh path, `--git` flag, and the
  guard behavior on already-installed towns.
- [Makefile](makefile.md) — builds the binary referenced here; the
  LDFLAGS it sets are what allow the binary to run without the
  self-kill check firing.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file.

## Failure modes

### Precondition violations (what does it assume?)

- **`GIT_USER` and `GIT_EMAIL` both set or both unset:** `docker-entrypoint.sh:6` — the `if` guard requires both to be set. If only one is provided, the entire git/dolt config block is skipped. **Absent** — no partial-set warning; git/dolt identity silently left unconfigured.
- **`/gt/mayor/town.json` as install marker:** `docker-entrypoint.sh:14` — presence of this file determines fresh install vs refresh. If the file is corrupt or from an incompatible version, the script takes the `--force` refresh path instead of a clean install. **Absent** — no version check on the marker file.
- **`gt install` binary at `/app/gastown/gt`:** `docker-entrypoint.sh:16,19` — if the binary doesn't exist or is corrupt, the `set -e` at line 2 causes the container to exit immediately on first use. **Present** — `set -e` catches the error, but the error message is just the shell's default.

### Partial completion (what doesn't it clean up?)

- **`set -e` with no trap:** `docker-entrypoint.sh:2` — the script uses `set -e` for error handling but has no `trap` for cleanup. If `git config` succeeds but `dolt config` fails (line 10-11), git identity is configured but dolt identity is not. The container exits and the partial config persists on the volume. **Absent** — no rollback of partial config.

## Notes / open questions

- The script does not `cd` anywhere. The Dockerfile's
  `WORKDIR /gt` (`Dockerfile:53`) puts the entrypoint in `/gt`
  already, so `gt install /gt` runs against an absolute path from
  within the target directory.
- `gt install --force` is the refresh path. Its exact behavior
  (overwrite vs merge of per-town state) is not yet mapped in this
  wiki — follow-up.
- The marker `/gt/mayor/town.json` is a strong cross-reference point
  to the `mayor` role and its on-disk state. Neither is documented
  here yet.
- If `GIT_USER` is set but `GIT_EMAIL` is unset (or vice versa), the
  `if` block is skipped entirely — both must be set. The compose
  file's defaults ensure this rarely matters in practice.
