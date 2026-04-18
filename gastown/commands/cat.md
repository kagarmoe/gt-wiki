---
title: gt cat
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/cmd/cat.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, work, bd-wrapper, display]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: agent
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt cat

Display a single bead's content — a thin convenience wrapper around
`bd show` with prefix-based rig routing.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`cat.go:16`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/cat.go:14-79` — the
entire file is 79 lines.

### Invocation

```
gt cat <bead-id> [--json]
```

Exactly one positional argument (`cobra.ExactArgs(1)` at `cat.go:28`).

### Behavior

`runCat` (`cat.go:37-61`):

1. **Validate the ID shape** via `isBeadID(beadID)` — must contain a
   hyphen (not at the start or end), and the prefix (everything before
   the first hyphen) must be all lowercase letters. See
   `cat.go:66-79`. Examples that pass: `gt-abc123`, `bd-xyz`,
   `hq-cv-foo`, `wisp-bar`, `mol-baz`.
2. **Build `bd show <bead-id>`** arguments; append `--json` if
   `--json` was given (`cat.go:45-49`).
3. **Resolve the bead directory** via `resolveBeadDir(beadID)` — if
   the function returns a non-empty and non-`"."` path, set
   `bdCmd.Dir` to that path and strip `BEADS_DIR` from the subprocess
   environment via `filterEnvKey(os.Environ(), "BEADS_DIR")`
   (`cat.go:55-58`). This ensures `bd` discovers the correct database
   from its working directory rather than using an inherited (possibly
   wrong) `BEADS_DIR` override.
4. **Exec `bd`** with stdout and stderr wired to the parent
   (`cat.go:51-60`) and return the exit code via `bdCmd.Run()`.

### Subcommands

None.

### Flags

Defined in `init()` at `cat.go:32-35`:

| flag | type | default | description |
|------|------|---------|-------------|
| `--json` | bool | `false` | Output as JSON |

### `isBeadID` heuristic

Minimal validation at `cat.go:66-79`:

```go
func isBeadID(s string) bool {
    dashIdx := strings.Index(s, "-")
    if dashIdx <= 0 || dashIdx >= len(s)-1 {
        return false
    }
    for _, c := range s[:dashIdx] {
        if c < 'a' || c > 'z' {
            return false
        }
    }
    return true
}
```

No maximum prefix length. `xyz-abc`, `verylongprefix-abc` both pass.
This is a shape check, not a whitelist — contrast with
`looksLikeIssueID` in [convoy.go](convoy.md) (`convoy.go:50-70`), which
consults a session-level prefix registry.

### Related commands

- [bead](bead.md) — `gt bead show` and `gt bead read` are also wrappers
  around `bd show` via a shared `runShow` helper (located outside
  `cat.go`).
- [../binaries/gt.md](../binaries/gt.md) — root binary.

## Notes / open questions

- **`resolveBeadDir` lives elsewhere.** The prefix-to-rig-database
  routing logic is in a helper not defined in `cat.go`. Worth a
  separate entity page once mapped — same helper is used by
  [close](close.md) at `close.go:90-94`.
- **No formatter in gt-land** — `gt cat` does not post-process `bd
  show` output. `--json` is forwarded to `bd` and its JSON output is
  emitted verbatim. Any pretty-printing is `bd`'s job.
