---
title: Makefile
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/Makefile
tags: [file, build, ldflags, makefile]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [failure-modes]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# Makefile

Canonical build recipe for the gastown project. Lives at
`/home/kimberly/repos/gastown/Makefile`. Produces the
[gt](../binaries/gt.md),
[gt-proxy-server](../binaries/gt-proxy-server.md),
and [gt-proxy-client](../binaries/gt-proxy-client.md)
binaries used by every non-release build path (native
dev on Linux/macOS, `make install` to `~/.local/bin`, container builds via
the [Dockerfile](dockerfile.md)).

**Also known as:** "the Makefile", "`make build`" (most references in docs
and code invoke it by target name).

## What it actually does

### Declared phony targets

`Makefile:1` declares every target as `.PHONY`:
`build`, `desktop-build`, `desktop-run`, `install`, `safe-install`,
`check-forward-only`, `clean`, `test`, `test-e2e-container`,
`check-up-to-date`.

### Variables

`Makefile:3-11`:

- `BINARY := gt` — name of the main CLI binary on disk.
- `BINARY_DESKTOP := gt-desktop` — name of the experimental desktop binary.
- `BUILD_DIR := .` — artifacts go in the repo root.
- `INSTALL_DIR := $(HOME)/.local/bin` — install target directory.
- `E2E_IMAGE ?= gastown-test` — container tag used by `test-e2e-container`.
- `E2E_BUILD_FLAGS ?=` / `E2E_RUN_FLAGS ?= --rm` — pass-through flags for
  `docker build` / `docker run` in the e2e target.
- `E2E_BUILD_RETRIES ?= 1` / `E2E_RUN_RETRIES ?= 1` — retry counts for
  flaky docker operations.

### Version metadata

`Makefile:14-16` derives three strings used by `LDFLAGS`:

- `VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")`
- `COMMIT := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")`
- `BUILD_TIME := $(shell date -u +"%Y-%m-%dT%H:%M:%SZ")`

These are informational only. The execution gate is `BuiltProperly`, not
any of these — see [../binaries/gt.md](../binaries/gt.md).

### LDFLAGS

`Makefile:18-22`:

```
-s -w \
-X github.com/steveyegge/gastown/internal/cmd.Version=$(VERSION) \
-X github.com/steveyegge/gastown/internal/cmd.Commit=$(COMMIT) \
-X github.com/steveyegge/gastown/internal/cmd.BuildTime=$(BUILD_TIME) \
-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1
```

`-s -w` strips the DWARF and symbol tables (smaller binary, no debug).
The critical `-X` is `BuiltProperly=1`, which bypasses the self-kill
check at `/home/kimberly/repos/gastown/internal/cmd/root.go:96-106`.
Without it, the resulting binary exits on first run with the "built with
'go build' directly" error. See [../binaries/gt.md](../binaries/gt.md).

`BuildTime` is set here but the corresponding `internal/cmd` variable
may or may not exist at that exact name — the
[../binaries/gt.md](../binaries/gt.md) ldflag variable table only lists
`Version`, `Build`, `Commit`, `Branch`, `BuiltProperly`. The Makefile
and [.goreleaser.yml](goreleaser-yml.md) disagree about which fields
are actually populated per build path (see Notes).

### macOS ICU detection

`Makefile:27-33` — runs only when `uname` is `Darwin`:

```make
ifeq ($(shell uname),Darwin)
  ICU_PREFIX := $(shell brew --prefix icu4c 2>/dev/null)
  ifneq ($(ICU_PREFIX),)
    export CGO_CPPFLAGS += -I$(ICU_PREFIX)/include
    export CGO_LDFLAGS  += -L$(ICU_PREFIX)/lib
  endif
endif
```

Homebrew installs `icu4c` as a keg-only package, so its headers/libs
aren't on the default search path. The Makefile auto-detects the
prefix and exports CGo flags so the `go-icu-regex` transitive
dependency (via Dolt / beads pulls) can find them. Not needed on
Linux — system ICU is on the default search path.

### `build` target

`Makefile:35-38`. Produces three binaries by calling `go build` once per
entry point:

```make
build:
	go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY)-proxy-server ./cmd/gt-proxy-server
	go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY)-proxy-client ./cmd/gt-proxy-client
	go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY) ./cmd/gt
```

Order matters only cosmetically (proxy-server and proxy-client build
first, then `gt`). Outputs: `./gt-proxy-server`, `./gt-proxy-client`,
`./gt` in the repo root. See [../binaries/gt.md](../binaries/gt.md).

### `desktop-build` target

`Makefile:40-41`. One `go build` for an experimental desktop entry point:

```make
desktop-build:
	go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY_DESKTOP) ./cmd/gt-desktop
```

Output: `./gt-desktop`. Uses the same LDFLAGS as `build`. `cmd/gt-desktop`
is not in the `inventory/go-packages.md` enumeration of `cmd/` — verify
whether this directory even exists or whether the target is dormant.

### `desktop-run` target

`Makefile:43-44`:

```make
desktop-run:
	go run ./cmd/gt-desktop
```

No LDFLAGS. `go run` produces a temporary binary — this means
`BuiltProperly` is empty, so the self-kill check fires **unless**
`cmd.Build` defaults to something other than `"dev"` in the source.
See [../binaries/gt.md](../binaries/gt.md) for the gate semantics.

### `check-up-to-date` target

`Makefile:46-68`. Guard run before `install` / `safe-install` to refuse
rebuilding off a stale local branch. Gated by `SKIP_UPDATE_CHECK` env
var — if set, the body is a no-op.

Behavior (`Makefile:49-67`):

1. Skip on detached HEAD (tag checkouts, CI builds) — `git symbolic-ref HEAD`
   failure exits 0.
2. Resolve upstream tracking ref via `git rev-parse --abbrev-ref --symbolic-full-name @{u}`.
   Empty upstream prints a warning and exits 0.
3. Parse remote name and branch from the upstream string.
4. `git fetch <remote> <branch> --quiet` (failures silently tolerated).
5. Compare local HEAD to remote HEAD; if they differ, print an error
   with both short-SHAs and exit 1.

### `check-forward-only` target

`Makefile:70-92`. Guard run only by `safe-install`. Gated by
`SKIP_FORWARD_CHECK` env var.

Purpose (`Makefile:70-72` comment): ensure HEAD is a descendant of the
currently installed binary's commit. Prevents the documented crash loop
where a replaced binary broke session startup hooks — witness respawned
the session every 1–2 minutes.

Behavior (`Makefile:75-91`):

1. Extract the installed binary's commit by running
   `$(INSTALL_DIR)/$(BINARY) version --verbose`, grepping for `@[a-f0-9]*`,
   and stripping the `@`. This relies on the format emitted by
   [../commands/version.md](../commands/version.md).
2. If the binary's commit equals HEAD (or short HEAD), print
   "Binary is already at HEAD, nothing to do" and exit 1
   (intentionally non-zero — a no-op rebuild is treated as an error).
3. Otherwise run `git merge-base --is-ancestor "$BINARY_COMMIT" HEAD`.
   If that fails, print a DOWNGRADE error and exit 1.
4. If the commit can't be determined, print a warning and continue.

### `install` target

`Makefile:94-119`. Depends on `check-up-to-date` and `build`. Full install
path that also restarts the daemon. The "primary" install route used
interactively by humans.

Steps (`Makefile:95-119`):

1. `mkdir -p $(INSTALL_DIR)` — make sure `~/.local/bin` exists.
2. `rm -f $(INSTALL_DIR)/$(BINARY)` — remove the old binary.
3. `cp $(BUILD_DIR)/$(BINARY) $(INSTALL_DIR)/$(BINARY)` — install the
   newly-built binary (not atomic).
4. Loop over `$HOME/go/bin/gt` and `$HOME/bin/gt` and remove any that
   exist (`Makefile:99-104`) — prevents stale `go install` binaries
   from shadowing the canonical `~/.local/bin/gt` on `$PATH`.
5. Print `"Installed gt to $INSTALL_DIR/gt"`.
6. **Daemon restart** (`Makefile:108-115`): if [`gt daemon`](../commands/daemon.md) `status` returns
   0, run `gt daemon stop`, sleep 1, `gt daemon start`. Warn on failure.
   Motivation in the Makefile comment: "A stale daemon is a recurring
   source of bugs (wrong session prefixes, etc.)".
7. **Plugin sync** (`Makefile:118-119`): run [`gt plugin`](../commands/plugin.md)
   `sync --source $(CURDIR)/plugins`. Silent failure allowed. Prevents
   plugin drift between the build repo and the town runtime directories.

### `safe-install` target

`Makefile:124-137`. Depends on `check-up-to-date`, `check-forward-only`,
and `build`. Non-interactive install path — replaces the binary atomically
without restarting the daemon, intended for automated rebuilds
(`rebuild-gt` plugin cited in comments).

Steps (`Makefile:125-137`):

1. `mkdir -p $(INSTALL_DIR)`.
2. `cp $(BUILD_DIR)/$(BINARY) $(INSTALL_DIR)/$(BINARY).new` then
   `mv $(INSTALL_DIR)/$(BINARY).new $(INSTALL_DIR)/$(BINARY)` — uses a
   rename to make the swap atomic on the same filesystem.
3. Same stale-binary nuking loop as `install`
   (`$HOME/go/bin/gt`, `$HOME/bin/gt`).
4. Print "Installed gt ... (daemon NOT restarted)" and the reminder
   "Sessions will pick up new binary on next cycle."

Key difference from `install`: **no daemon stop/start**, **no plugin
sync**. Running sessions keep using the old in-memory binary until their
next natural cycle/handoff.

### `clean` target

`Makefile:139-140`:

```make
clean:
	rm -f $(BUILD_DIR)/$(BINARY)
```

Removes `./gt` only. Does **not** remove `./gt-proxy-server`,
`./gt-proxy-client`, or `./gt-desktop`. A full cleanup of all build
artifacts is not provided by this target.

### `test` target

`Makefile:142-143`:

```make
test:
	go test ./...
```

No `-tags=e2e` (so e2e tests are skipped — those run via
`test-e2e-container`). No `-race`. No coverage. Just recursive unit tests.

### `test-e2e-container` target

`Makefile:146-167`. Runs the e2e test suite inside a docker container
built from [Dockerfile.e2e](dockerfile-e2e.md). Has two parallel
implementations gated on `$(OS)`:

**Windows path (`Makefile:148-149`):** two `powershell` invocations that
each run a retry loop. Retry counts come from `E2E_BUILD_RETRIES` /
`E2E_RUN_RETRIES`. First loop does `docker build -f Dockerfile.e2e
-t $(E2E_IMAGE) .`. Second loop does `docker run $(E2E_RUN_FLAGS)
$(E2E_IMAGE)`. Each failure sleeps 2s and retries up to the max.

**POSIX path (`Makefile:151-166`):** shell retry loops with the same
semantics. The `docker build` loop passes `$(E2E_BUILD_FLAGS)`;
the `docker run` loop passes `$(E2E_RUN_FLAGS)`. The `attempt`
variable starts at 1 and increments on failure.

The Makefile comment at `Makefile:145` says this is "the only supported
way to run" the e2e tests.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — the binary this Makefile
  produces; documents the `BuiltProperly` gate, the ldflag variable
  table, and the `persistentPreRun` sequence.
- [../commands/version.md](../commands/version.md) — `check-forward-only`
  parses this command's `--verbose` output to get the installed binary's
  commit SHA.
- [Dockerfile](dockerfile.md) — the primary Dockerfile runs `make build`
  inside the container image.
- [Dockerfile.e2e](dockerfile-e2e.md) — built and run by
  `test-e2e-container`.
- [.goreleaser.yml](goreleaser-yml.md) — alternative build path used for
  release artifacts. Does NOT invoke `make build`; runs its own
  `go build` per platform with different LDFLAGS (no `BuildTime`, adds
  `Build` and `Branch`).
- [flake.nix](flake-nix.md) — alternative Nix-based build path. Does
  NOT invoke `make build`; uses `buildGoModule` with its own LDFLAGS.
- [../inventory/repo-root.md](../inventory/repo-root.md) — lists the
  Makefile in the top-level inventory.

## Notes / open questions

- What [gt-proxy-server](../binaries/gt-proxy-server.md) and
  [gt-proxy-client](../binaries/gt-proxy-client.md) actually do — now
  mapped as entity pages.
- The `desktop-build` / `desktop-run` targets reference `./cmd/gt-desktop`,
  but the `inventory/go-packages.md` enumeration of `cmd/` only lists
  `gt`, `gt-proxy-client`, `gt-proxy-server`. Does `cmd/gt-desktop` exist
  on disk? If not, these targets are dead.
## Failure modes

### Precondition violations (what does it assume?)

- **Git repo with tracking branch:** `check-up-to-date` at `Makefile:47-68` requires a git repo with an upstream tracking branch. On detached HEAD (tag checkouts, CI), it silently exits 0. If the local branch has no upstream, it warns and exits 0. **Present** — guarded with explicit skip conditions.
- **Forward-only install:** `check-forward-only` at `Makefile:73-92` prevents downgrade installs by verifying HEAD is a descendant of the installed binary's commit. If the installed binary can't report its commit, the check is skipped with a warning. **Present** — explicit guard prevents crash-loop regression (documented: "caused a crash loop where the replaced binary broke session startup hooks").
- **ICU4C on macOS:** `Makefile:27-33` — `brew --prefix icu4c` auto-detects Homebrew's ICU path. If Homebrew isn't installed or `icu4c` isn't installed, the CGO flags are not set and the build may fail on macOS with missing ICU headers. **Absent** — no error message if ICU detection fails; the build just proceeds and the Go compiler emits a cryptic linker error.

### Partial completion (what doesn't it clean up?)

- **`install` target daemon restart failure:** `Makefile:108-115` — after copying the binary, the target tries to restart the daemon. If `daemon stop` succeeds but `daemon start` fails, the binary is installed but the daemon is down. The target prints a warning ("daemon restart failed") but exits 0. **Present** — warning printed, but the install is considered successful with a dead daemon.
- **`safe-install` exit 1 on no-op:** `Makefile:124` — `check-forward-only` exits 1 when the binary is already at HEAD. Callers must treat this "nothing to do" exit 1 as success. **Present** — the message "Binary is already at HEAD" distinguishes this from a real error, but the exit code is misleading.

### Silent suppression (what errors are swallowed?)

- **Plugin sync failure:** `Makefile:118-119` — `gt plugin sync` errors are swallowed with `|| true`. If plugin sync fails, the install succeeds but runtime plugins may be stale. **Absent** — error completely suppressed.
- **Stale binary removal failure:** `Makefile:99-104,130-135` — `rm -f` on stale binaries at `$HOME/go/bin/gt` and `$HOME/bin/gt` cannot fail (rm -f), but if the stale binary is running, it will be unlinked but still running in memory. **Present** — `rm -f` handles the common case.

- The Makefile sets `BuildTime` via `-X internal/cmd.BuildTime=`, but the
  variable table in [../binaries/gt.md](../binaries/gt.md) does not list
  a `BuildTime` field on `internal/cmd`. Either the variable exists and
  is unlisted, or the `-X` is setting a non-existent variable (no-op).
- `install` target kills stale binaries at `$HOME/go/bin/gt` and
  `$HOME/bin/gt`, but not e.g. `/usr/local/bin/gt`. The blast radius
  is scoped to per-user install directories.
- `desktop-run` uses `go run` without ldflags — if the source defaults
  `Build` to `"dev"` and `BuiltProperly` to `""`, the self-kill check
  fires on first invocation. Whether the target works in practice
  depends on source defaults in
  `/home/kimberly/repos/gastown/internal/cmd/version.go`.
- `safe-install` deliberately exits 1 in `check-forward-only` when the
  binary is already at HEAD. This is a "nothing to do" signal masquerading
  as an error — callers of `safe-install` need to treat exit 1 with the
  "Binary is already at HEAD" stdout as success.
