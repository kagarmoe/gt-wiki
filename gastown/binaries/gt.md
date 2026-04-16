---
title: gt
type: binary
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/cmd/gt/main.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
  - /home/kimberly/repos/gastown/internal/cmd/version.go
  - /home/kimberly/repos/gastown/internal/cli/name.go
  - /home/kimberly/repos/gastown/Makefile
tags: [cli, build, root-command]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
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
4. **Command usage logging** — fire-and-forget; excludes [`tap`](../commands/tap.md) and
   [`signal`](../commands/signal.md) commands.
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
  top-level commands each: `estop.go` adds [`estop`](../commands/estop.md) + [`thaw`](../commands/thaw.md), `start.go`
  adds [`start`](../commands/start.md) + [`shutdown`](../commands/shutdown.md)).
- **~107 unique top-level subcommands** (the gap is because some files
  register the same command via different code paths; exact unique
  count pending a verification pass).
- **495 total `cobra.Command` definitions** across `internal/cmd/`
  (top-level commands + nested subcommands like `rig add`,
  `mayor start`, `convoy launch`, etc.).

The canonical list is compiled in [../commands/README.md](../commands/README.md).

### Sibling binaries

`gt` is one of three binaries produced by
[`make build`](../files/makefile.md). The other two are thin,
purpose-built shims around the `internal/proxy/` package with no
overlap in Go dependencies with `gt`:

- [gt-proxy-server](gt-proxy-server.md) — long-lived mTLS proxy server
  that runs on the host and accepts `gt`/`bd` invocations from
  polecat containers over an authenticated HTTP endpoint.
- [gt-proxy-client](gt-proxy-client.md) — container-side shim
  installed as both `gt` and `bd` inside polecat containers.
  Forwards `argv[1:]` to the proxy server when
  `GT_PROXY_URL`/`GT_PROXY_CERT`/`GT_PROXY_KEY`/`GT_PROXY_CA` are
  set, otherwise `execve`s the real `gt`/`bd` binary at
  `/usr/local/bin/gt.real` (or `$GT_REAL_BIN`).

Neither proxy binary imports `internal/cmd`, so the `BuiltProperly`
self-kill gate described above does NOT apply to them — it's a
`gt`-only mechanism.

## Notes / open questions

- `internal/cli/name.go`: `gt` reads `GT_COMMAND` env var at startup
  (via `cli.Name()` + `sync.Once`) and uses the value as the cobra
  command name, falling back to `"gt"`. Purpose stated in the file
  comment: coexist with Graphite (also uses `gt`).
- `Execute()` at `internal/cmd/root.go:291-317` initializes an
  OpenTelemetry provider via `telemetry.Init(ctx, "gastown", Version)`
  on every invocation and calls `telemetry.SetProcessOTELAttrs` to
  export OTEL resource attributes into the process environment so
  child `bd` processes inherit them. Deferred shutdown with a 2-second
  timeout. Endpoint, opt-out env var, and data shipped all TBD —
  tracked in bead `wiki-9u4`.
- `persistentPreRun` runs three checks on every invocation that emit
  stderr warnings: stale binary (`version.CheckStaleBinary`),
  town-root branch (`warnIfTownRootOffMain`), beads version
  (`CheckBeadsVersion`). Each has its own exempt-command list; see
  "beadsExemptCommands" and "branchCheckExemptCommands" above.
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
  duplication patterns (e.g., [`estop`](../commands/estop.md)/[`thaw`](../commands/thaw.md) listed both here and in
  `branchCheckExemptCommands`). Verify no typos/dead entries.
- `internal/cmd/proxy_subcmds.go:15` defines
  `AnnotationPolecatSafe = "polecatSafe"` — a cobra annotation tagging
  commands as safe inside polecat sessions. `version` uses it
  (`version.go:34`). How many commands are polecat-safe, and what does
  "polecat-safe" actually gate on?
