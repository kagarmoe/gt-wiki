---
title: gt version
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/version.go
tags: [command, diagnostics, version, polecat-safe]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt version

Prints version information for the `gt` binary.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** yes (annotated `polecatSafe` via
`AnnotationPolecatSafe` constant)
**Beads-exempt:** yes (works without `bd` installed — in
`beadsExemptCommands` map on
`/home/kimberly/repos/gastown/internal/cmd/root.go:45`)
**Branch-check-exempt:** yes (does not warn on non-main town root — in
`branchCheckExemptCommands` on `root.go:82`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/version.go:31-73`.

### Invocation

```
gt version [--verbose] [--short]
```

Flags:
- `--verbose` / `-v` — include timestamp and Go runtime version.
- `--short` — output only `<Version>-<Build>` (e.g. `1.0.0-dev`).

### Output formats

Driven by a cascade on
`/home/kimberly/repos/gastown/internal/cmd/version.go:40-60`:

- `--short` → `<Version>-<Build>\n`, e.g. `1.0.0-dev`
- With both commit and branch resolved:
  `gt version <Version> (<Build>: <branch>@<short-commit>)`
- With commit only:
  `gt version <Version> (<Build>: <short-commit>)`
- With neither:
  `gt version <Version> (<Build>)`

`--verbose` additionally appends:
```
Timestamp: <RFC3339 now>
Go version: <runtime.Version()>
```

### Version source

Five variables in `version.go:16-26` are set at build time via ldflags:

```go
var (
    Version       = "1.0.0"
    Build         = "dev"
    Commit        = ""
    Branch        = ""
    BuiltProperly = ""
)
```

The canonical `make build` LDFLAGS recipe that populates these is
documented in [../files/makefile.md](../files/makefile.md).

**Commit resolution cascade** (`version.go:75-89`):
1. If the `Commit` ldflag was set at build time, use it.
2. Otherwise, try `debug.ReadBuildInfo()` and look for
   `vcs.revision` in build settings (Go 1.18+ module VCS embedding).
3. Otherwise, empty string.

**Branch resolution cascade** (`version.go:91-115`):
1. If the `Branch` ldflag was set at build time, use it.
2. Otherwise, try `debug.ReadBuildInfo()` and look for `vcs.branch`.
3. Otherwise, run `git symbolic-ref --short HEAD` in the current
   directory at runtime and use that (skipping if it returns `HEAD` or
   empty — indicating detached HEAD).

Only short commits are printed — the full hash is truncated via
`version.ShortCommit()`.

### Side effect: `version.SetCommit`

`version.go:69-72` — in `init()`, if the build-time `Commit` ldflag is
non-empty, the version subcommand calls `version.SetCommit(Commit)` to
pass the value to the [internal/version](../packages/version.md)
package. This is used by the stale-binary-check logic in
`persistentPreRun` — see [../binaries/gt.md](../binaries/gt.md)
"persistentPreRun sequence".
Importantly, this means **the version command's `init()` participates
in bootstrapping the stale-binary-check machinery**, not just printing
a version string.

## Inline help text

`/home/kimberly/repos/gastown/internal/cmd/version.go:36-39` `Long`
field (what `gt version --help` prints as the long description):

> Print the gt version, build type, git branch, and commit hash.
> Output includes the semantic version, whether this is a dev or
> release build, and the git revision the binary was built from (if
> available).

## Failure modes

No significant failure modes. Pure read-only command that prints
build-time constants. The `resolveBranch` fallback shells out to
`git symbolic-ref` (`version.go:106-113`) but swallows errors and
returns empty string on failure, which is the correct degradation
(output simply omits the branch field).

## Notes / open questions

- **The polecatSafe annotation** — `version.go:34` sets
  `Annotations: map[string]string{AnnotationPolecatSafe: "true"}`. What
  does a polecat-safe command actually mean for runtime dispatch?
  Does any runtime code check for this annotation and branch? See
  `internal/cmd/proxy_subcmds.go:15` where the constant is defined.
- **`version.SetCommit` consumer** — which part of `internal/version`
  reads the value, and what stale-check logic uses it?
  `version.CheckStaleBinary` is referenced in `root.go:268` — worth a
  dedicated entity page under `packages/version.md`.
- **Homebrew formula**: does it set `Build=Homebrew`, `Commit`, and
  `Branch` via ldflags? If so, the output under Homebrew would be
  something like `gt version 1.0.0 (Homebrew: <branch>@<commit>)` —
  verify against an actual Homebrew install.
- **`gt --version`** (from cobra's automatic flag) vs **`gt version`**
  (this command) — the root command
  (`/home/kimberly/repos/gastown/internal/cmd/root.go:27`) sets
  `Version: Version`, so cobra will also print via `--version`. The
  two outputs are **not identical** — cobra's default `--version`
  output is minimal and doesn't go through the cascade above. Drift
  risk if users expect parity.
