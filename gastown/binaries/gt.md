---
title: gt
type: binary
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/cmd/gt/main.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
  - /home/kimberly/repos/gastown/internal/cmd/version.go
  - /home/kimberly/repos/gastown/internal/cli/name.go
  - /home/kimberly/repos/gastown/Makefile
  - /home/kimberly/repos/gastown/README.md
tags: [cli, build, self-kill, root-command]
---

# gt

The main Gas Town CLI binary. The entry point for all `gt` workflows.

## What it actually does

### Entry point

`/home/kimberly/repos/gastown/cmd/gt/main.go` is a 12-line stub:

```go
package main

import (
    "os"
    "github.com/steveyegge/gastown/internal/cmd"
)

func main() {
    os.Exit(cmd.Execute())
}
```

All meaningful work is in `internal/cmd/`. The binary's command name is
**not hardcoded** — it is resolved at runtime from the `GT_COMMAND`
environment variable (falls back to `"gt"`), via
`/home/kimberly/repos/gastown/internal/cli/name.go`. Rationale (from the
comment): "This allows coexistence with other tools that use `gt` (e.g.,
Graphite)." `sync.Once` caches the resolved name for the life of the
process.

`/home/kimberly/repos/gastown/internal/cmd/root.go:32-40` calls
`cli.Name()` in `init()` and rewrites `rootCmd.Use` and `rootCmd.Long`
accordingly, so the help text and error messages all reflect the dynamic
name. The `$(MAKE) build` binary is still named `gt` on disk — the
renaming is purely for the cobra presentation layer.

### Build-time variables (ldflags)

`/home/kimberly/repos/gastown/internal/cmd/version.go:16-26` declares
five variables set at build time by `make build`:

| Var            | Default     | Purpose                                                                 |
|----------------|-------------|-------------------------------------------------------------------------|
| `Version`      | `"1.0.0"`   | Semantic version string                                                 |
| `Build`        | `"dev"`     | Build type. `make build` leaves default; Homebrew sets to `"Homebrew"`  |
| `Commit`       | `""`        | Git commit hash                                                         |
| `Branch`       | `""`        | Git branch                                                              |
| `BuiltProperly`| `""`        | Gate for the self-kill check; set to `"1"` by `make build`              |

The canonical `make build` LDFLAGS recipe lives in
[../files/makefile.md](../files/makefile.md).

### Self-kill check at startup

`/home/kimberly/repos/gastown/internal/cmd/root.go:94-107` — the
`persistentPreRun` function runs before every subcommand. First thing it
does:

```go
if BuiltProperly == "" && Build == "dev" {
    fmt.Fprintln(os.Stderr, "ERROR: This binary was built with 'go build' directly.")
    fmt.Fprintln(os.Stderr, "       macOS will SIGKILL unsigned binaries. Use 'make build' instead.")
    // ...
    os.Exit(1)
}
```

Any command (including `gt --version` and `gt completion bash`) hits
this gate. Bypassed only when `BuiltProperly="1"` OR `Build != "dev"`.

### persistentPreRun sequence

After the self-kill gate, `persistentPreRun`
(`/home/kimberly/repos/gastown/internal/cmd/root.go:94-157`) runs these
steps in order for every `gt` invocation:

1. **Self-kill check** (above).
2. **CLI theme init** — reads town settings for `CLITheme` and applies
   dark/light mode via `internal/ui`.
3. **Telemetry init** — via `internal/telemetry.Init` (see the `Execute`
   function below; telemetry is actually initialized in `Execute`, not
   `persistentPreRun`, but the effect is observable before subcommand
   bodies run).
4. **Command usage logging** — fire-and-forget; excludes `tap` and
   `signal` commands.
5. **Session registry init** — discovers the town root (CWD walk with
   `GT_TOWN_ROOT`/`GT_ROOT` env-var fallback) and initializes the
   session prefix registry via `session.InitRegistry`. Used to route
   commands to the correct town socket even when invoked from outside
   the town directory.
6. **Polecat heartbeat touch** — if `GT_SESSION` is set and `GT_ROLE`
   contains `"polecat"`, `"crew"`, or `"dog"`, touches a per-session
   heartbeat file via `polecat.TouchSessionHeartbeat`. Used by the
   stale-session-detection logic (`gt-qjtq: ZFC liveness fix`).
7. **Stale binary warning** — unless command is in `beadsExemptCommands`
   (see below), checks if the running binary is N commits behind the
   repo it was built from. Warning only, once per shell session
   (memoized via `GT_STALE_WARNED` env var).
8. **Town-root branch check** — unless command is in
   `branchCheckExemptCommands`, warns if the town root is on a branch
   other than `main`, `master`, or `gt_managed`. Warning only.
9. **Beads version check** — unless exempt, runs `CheckBeadsVersion` to
   warn if the installed `bd` is missing or outdated.

### `Execute()` function

`/home/kimberly/repos/gastown/internal/cmd/root.go:291-317` — wraps
`rootCmd.Execute()` with:

1. **OpenTelemetry provider init** via
   `telemetry.Init(ctx, "gastown", Version)`. Set as process OTEL
   resource attributes so all `bd` subprocesses spawned via
   `exec.Command` inherit GT context automatically.
2. **`rootCmd.Execute()`** — runs the cobra dispatch.
3. **Deferred shutdown** of the telemetry provider with a 2-second
   timeout.
4. **Silent-exit handling** via `IsSilentExit` — some scripting commands
   signal status via exit code only, bypassing cobra's normal error
   printing.

### Command groups

Seven help-output groups defined in
`/home/kimberly/repos/gastown/internal/cmd/root.go:319-348`, in this
order:

| ID                    | Title                 |
|-----------------------|-----------------------|
| `GroupWork`           | Work Management       |
| `GroupAgents`         | Agent Management      |
| `GroupComm`           | Communication         |
| `GroupServices`       | Services              |
| `GroupWorkspace`      | Workspace             |
| `GroupConfig`         | Configuration         |
| `GroupDiag`           | Diagnostics           |

The built-in `help` command is placed in `GroupDiag`; `completion` is
placed in `GroupConfig`. Every `rootCmd.AddCommand`-registered command
is assigned a `GroupID` in its own definition — see
[../commands/README.md](../commands/README.md) for the full mapping.

### `beadsExemptCommands`

`/home/kimberly/repos/gastown/internal/cmd/root.go:44-77` — 28 commands
that are exempt from the `bd` version check (they must work even when
`bd` is missing or outdated):

```
version, help, completion, crew, polecat, witness, refinery, status,
mail, hook, prime, nudge, seance, doctor, dolt, handoff, costs, feed,
rig, config, install, tap, dnd, estop, thaw, signal, metrics, krc,
run-migration, health, upgrade, heartbeat
```

(Note: the grep count is 32 entries as written; some are agent/crew
commands that run from inside sessions where `bd` may not be available,
some are diagnostic/recovery commands that must work even when bd is
broken.)

### `branchCheckExemptCommands`

`/home/kimberly/repos/gastown/internal/cmd/root.go:81-91` — 9 commands
exempt from the town-root-branch warning:

```
version, help, completion, doctor, estop, thaw, install, git-init, upgrade
```

### Command surface (the big number)

- **111 `rootCmd.AddCommand` invocations** across 109 files in
  `/home/kimberly/repos/gastown/internal/cmd/` (two files register two
  top-level commands each: `estop.go` adds `estop` + `thaw`, `start.go`
  adds `start` + `shutdown`).
- **~107 unique top-level subcommands** (the gap is because some files
  register the same command via different code paths; exact unique
  count pending a verification pass).
- **495 total `cobra.Command` definitions** across `internal/cmd/`
  (top-level commands + nested subcommands like `rig add`,
  `mayor start`, `convoy launch`, etc.).

The canonical list is compiled in [../commands/README.md](../commands/README.md).

## Docs claim

### README.md (740 lines at `/home/kimberly/repos/gastown/README.md`)

Main user-facing doc. Claims `gt` has a user-visible CLI surface of
roughly a dozen subcommands (specific list pending — see bead for
systematic README-claim extraction).

### Installation section (`/home/kimberly/gt/GASTOWN-README.md:141`)

> `$ go install github.com/steveyegge/gastown/cmd/gt@latest  # From source (macOS/Linux)`

Implies `go install` produces a working binary.

> `# If using go install, add Go binaries to PATH (add to ~/.zshrc or ~/.bashrc)`
> `export PATH="$PATH:$HOME/go/bin"`

Implies `$HOME/go/bin` is where the binary lives.

### AGENTS.md (`/home/kimberly/repos/gastown/AGENTS.md`)

Separate agent-facing doc (4334 bytes). Content not yet mapped — tracked
in a follow-up bead.

## Drift

1. **`go install` is a broken install path.** The README tells users to
   `go install github.com/steveyegge/gastown/cmd/gt@latest`. The
   resulting binary self-kills on first run with:

   > ERROR: This binary was built with 'go build' directly.
   >        macOS will SIGKILL unsigned binaries. Use 'make build' instead.

   No workaround is mentioned in the README. The binary works only if
   built with `make build` or an equivalent `-ldflags "-X
   ...BuiltProperly=1"`.

2. **Error message misattributes the platform.** The message mentions
   `macOS will SIGKILL unsigned binaries`, but the check is
   unconditional — it fires on Linux (and presumably Windows, WSL, any
   OS) even though macOS code signing has no relevance there.

3. **Stale comment claims warning-only behavior.**
   `/home/kimberly/repos/gastown/internal/cmd/root.go:97` comment reads
   "Warning only - doesn't block execution." The code at line 106 calls
   `os.Exit(1)` — it blocks. The comment was never updated when the
   behavior changed.

4. **`$HOME/go/bin` path is environment-specific.** README says binaries
   land in `$HOME/go/bin`. In the upstream `golang:1.26-bookworm` Docker
   image (a common container base), `GOPATH=/go`, so binaries land in
   `/go/bin` instead. `~/gt/Dockerfile` was originally wrong about this
   assumption.

5. **Binary name is dynamic, undocumented.**
   `/home/kimberly/repos/gastown/internal/cli/name.go` allows `gt` to be
   invoked as any name via `GT_COMMAND`. The README and help output do
   not mention this mechanism; a user who sees the help text showing
   `gt` in an environment where the binary was renamed would have no
   way to know why the help examples use `gt`. Purpose-stated: coexist
   with Graphite (also uses `gt`).

6. **Command surface is ~10× larger than docs claim.** README documents
   roughly a dozen top-level subcommands; code registers **~107**.
   Specific list of undocumented commands pending beads mapping each
   one, but the categorical scale of the miss is the drift. See
   [../commands/README.md](../commands/README.md).

7. **Three prominent runtime checks are undocumented.** None of the
   following are mentioned in README:
   - Stale binary warning (warns when binary is N commits behind repo).
   - Town-root branch warning (warns when repo is on a non-`main`-ish
     branch).
   - Beads version check (warns when `bd` is missing/outdated).
   Each is a visible side-effect on `stderr` that users will encounter
   and may misinterpret as an error.

8. **Telemetry init is undocumented.** `gt` runs OpenTelemetry by
   default on every invocation (`Execute()` initializes a provider),
   exports OTEL resource attributes to the process environment so child
   `bd` processes inherit them, and shuts down the provider on exit.
   None of this is mentioned in README. Users who care about
   telemetry / privacy / offline-use have no way to know this from the
   docs. Endpoint/destination not yet investigated — tracked in a
   follow-up bead.

## Notes / open questions

- The Homebrew formula sets `Build` to `"Homebrew"` via ldflags, which
  bypasses the self-kill check without needing `BuiltProperly`. Need to
  verify the exact formula content.
- The `@gastown/gt` npm package — does it wrap a prebuilt binary or
  build from source on install?
- `internal/telemetry` — what endpoint does it export to? Is there an
  opt-out env var? This is a privacy-relevant undocumented behavior.
- `polecat.TouchSessionHeartbeat` — who reads the heartbeat file, and
  when does "dead session" detection fire? Tracked as `gt-qjtq` in the
  gastown commit history (referenced in the persistentPreRun comment).
- `CheckBeadsVersion` — what version constraint does it enforce? The
  wiki's own embedded `bd` may or may not satisfy it.
- The `beadsExemptCommands` list has 32 entries but the grep shows some
  duplication patterns (e.g., `estop`/`thaw` listed both here and in
  `branchCheckExemptCommands`). Verify no typos/dead entries.
- `internal/cmd/proxy_subcmds.go:15` defines
  `AnnotationPolecatSafe = "polecatSafe"` — a cobra annotation tagging
  commands as safe inside polecat sessions. `version` uses it
  (`version.go:34`). How many commands are polecat-safe, and what does
  "polecat-safe" actually gate on?
