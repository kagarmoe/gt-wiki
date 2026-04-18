---
title: gt info
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/info.go
  - /home/kimberly/repos/gastown/internal/cmd/version.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, version, changelog, whats-new]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [silent-suppression]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# gt info

Prints a short Gas Town version banner, or (with `--whats-new`) a
hand-maintained in-binary changelog of agent-actionable changes per
version.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/info.go:13-66`)
**Beads-exempt:** no (not in `beadsExemptCommands`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/info.go:13-484`.

### Invocation

```
gt info                    # short version banner
gt info --whats-new        # recent-version changelog
gt info --whats-new --json # machine-readable changelog
```

Single terminal command (no subcommands). `Run` (not `RunE`) is inlined
as a closure at `info.go:27-65`.

### Behavior

#### Default mode — version banner

`info.go:27-65` — `Run` closure:

1. If `--whats-new` is set, delegates to `showWhatsNew(jsonFlag)`
   and returns (`info.go:31-34`).
2. Otherwise builds an info map from the same build-time ldflags that
   [version](version.md) uses — `Version`, `Build`, and the cascades for
   `commit` and `branch` (`info.go:37-47`).
3. Commit and branch resolution reuse helpers defined in
   [version](version.md):
   - `resolveCommitHash()` — see `version.go:75-89`. Checks ldflag
     first, then `debug.ReadBuildInfo`, returning short-commit via
     `version.ShortCommit`.
   - `resolveBranch()` — see `version.go:91-115`. Checks ldflag, then
     BuildInfo, then `git symbolic-ref --short HEAD`.
4. Emits either JSON (`info.go:49-54`) or a short human string:
   `"Gas Town v<Version> (<Build>)"` plus `"<branch>@<commit>"` when
   both are known (`info.go:56-63`).
5. Appends a tip line: `"Use 'gt info --whats-new' to see recent
   changes"` (`info.go:64`).

`resolveCommitHash` and `resolveBranch` are the same helpers the
`version` command uses — [version.md](version.md) documents the full
cascade.

#### `--whats-new` mode

`showWhatsNew` (`info.go:443-477`): prints a hand-maintained changelog
from the `versionChanges` slice (`info.go:76-441`). The current binary
version is marked with a `<- current` suffix when the slice entry's
`Version` equals the build-time `Version` ldflag (`info.go:461-466`).

In JSON mode, emits `{ current_version, recent_changes }` with the
slice serialized as `[]VersionChange` (`info.go:445-453`).

### `versionChanges` data

`info.go:76-441` — a static, in-binary slice of
`VersionChange{Version, Date, Changes}` entries from **v0.1.0
(2026-01-02)** through **v1.0.0 (2026-04-02)**.

```go
type VersionChange struct {
    Version string   `json:"version"`
    Date    string   `json:"date"`
    Changes []string `json:"changes"`
}
```

Each entry is a short bullet list under NEW / CHANGED / FIX / REMOVED
prefixes. The list is hand-curated (not auto-generated from git log)
and is the in-binary source of truth for what agents should know about
recent releases.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`info.go:479-483`):

| Flag           | Type | Default | Purpose                                            |
|----------------|------|---------|----------------------------------------------------|
| `--whats-new`  | bool | `false` | Show agent-relevant changes from recent versions   |
| `--json`       | bool | `false` | Output in JSON format                              |

## Related

- [version.md](version.md) — the dedicated version-printing command;
  shares the ldflag cascade for `commit` / `branch` resolution and
  the same `Version` / `Build` build-time variables.
- [../files/makefile.md](../files/makefile.md) — the canonical
  `make build` LDFLAGS recipe that populates the version variables
  read by info.
- [../binaries/gt.md](../binaries/gt.md) — parent binary; documents
  the five ldflag variables that info reads.
- [README.md](README.md) — command tree index.

## Failure modes

### Silent suppression (what errors are swallowed?)
- **JSON encoding error in `--json` mode:** `_ = enc.Encode(info)` at
  `info.go:53` discards the error from JSON encoding. If encoding
  fails (unlikely for a simple map), the user sees no output and no
  error. **Absent** — error is silently discarded.
- **JSON encoding error in `--whats-new --json`:** `_ = enc.Encode(...)` at
  `info.go:448` discards the error similarly. **Absent.**

Both suppressions are low-risk (encoding static data), but violate the
principle of never discarding an error without at least logging it.

## Notes / open questions

- `resolveCommitHash` and `resolveBranch` are defined in
  `version.go:75-115` — `info.go` relies on them being in the same
  package. No alternative resolution path exists.
- The `versionChanges` slice is hand-maintained in the source; drift
  between it and the actual git history is unchecked by any automated
  test.
- `gt info --whats-new --json` emits the full `versionChanges` slice;
  callers that only want the current version entry must filter
  client-side.
- `info` and `version` have substantial conceptual overlap — `info`
  adds the changelog and emits slightly different default formatting
  (`"Gas Town v1.0.0 (dev)"` vs `"gt version 1.0.0 (dev)"`). No
  hard drift, but two commands serving similar needs.
