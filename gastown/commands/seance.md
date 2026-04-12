---
title: gt seance
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/seance.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, diagnostics, sessions, handoff, claude-code, fork-session, predecessor]
---

# gt seance

Discovers predecessor agent sessions via the event stream, and
optionally spawns a `claude --fork-session --resume <id>` subprocess
so the current operator can literally talk to a predecessor session
without modifying it.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupDiag` ("Diagnostics")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/seance.go:33-64`)

> Note: the plan brief listed seance as polecat-safe based on a
> file-count heuristic, but the actual source has no
> `AnnotationPolecatSafe` entry in the `Annotations` map. Verified by
> `grep AnnotationPolecatSafe seance.go` returning no matches.

**Beads-exempt:** **yes** — in `beadsExemptCommands` at
`/home/kimberly/repos/gastown/internal/cmd/root.go:59`; seance reads
only from the event stream and spawns a Claude subprocess, never
touching `bd`.
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:82-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/seance.go:33-600+`.

### Invocation

```
gt seance [--role NAME] [--rig NAME] [-n N] [--json]
gt seance -t SESSION_ID [-p "prompt"]
```

### Behavior

`runSeance` (`seance.go:85-93`) is a dispatch:

- **`--talk` set** → `runSeanceTalk(sessionID, prompt)`
- **otherwise** → `runSeanceList()`

### List mode — `runSeanceList`

Source: `seance.go:95-192`.

1. Requires a Gas Town workspace via `workspace.FindFromCwd`
   (`seance.go:96-99`).
2. Calls `discoverSessions(townRoot)` (`seance.go:307-343`) which
   reads `~/gt/.events.jsonl` (resolved via `events.EventsFile` and
   the town root), scans line-by-line with a 1 MB buffer, unmarshals
   each line as a `sessionEvent`, and keeps those of type
   `events.TypeSessionStart`. Results are sorted by timestamp
   descending.
3. Applies `--role` and `--rig` filters as case-insensitive
   substring matches on the `Actor` field (`seance.go:108-123`).
4. Truncates to `--recent N` (default 20, `seance.go:125-128`).
5. If `--json`: encodes the filtered list to stdout and returns
   (`seance.go:130-134`).
6. Otherwise renders a text table with columns
   `SESSION_ID | ROLE | STARTED | TOPIC` — column widths are
   hard-coded at `seance.go:147-150`. Oversized fields are truncated
   with `…` single-character ellipsis.
7. Footer prints the two talk-mode invocations as hints
   (`seance.go:187-189`).

### Talk mode — `runSeanceTalk`

Source: `seance.go:208-289`.

1. Calls `resolveSeanceCommand()` (`seance.go:196-206`) which iterates
   `config.ListAgentPresets()`, returns the first preset whose
   `SupportsForkSession` is true, and resolves it via
   `config.RuntimeConfigFromPreset(name)`. Error if no preset
   supports fork-session.
2. `cleanupOrphanedSessionSymlinks()` (`seance.go:216`) — clears
   stale symlinks from prior interrupted talks.
3. Resolves town root via `workspace.FindFromCwd` (non-fatal if
   missing).
4. **Prefix resolution**: if the ID is shorter than 36 chars (a UUID
   prefix), `resolveSessionPrefix(townRoot, sessionID)`
   (`seance.go:347-376`) re-scans the event stream for session IDs
   starting with that prefix. Zero matches → error; one → returned
   UUID; multiple → error with up to 3 ambiguous matches.
5. Prints `🔮 Summoning session <id>...` (`seance.go:234`).
6. **Cross-account symlink**:
   `symlinkSessionToCurrentAccount(townRoot, sessionID)`
   (`seance.go:545-560`) finds the session file in any account's
   `~/.claude/projects/<hash>/sessions-index.json` via
   `findSessionLocation` (`seance.go:444-540`) and symlinks it into
   the current account's project dir so Claude's `--resume` can see
   it. Locking handled by `lockSessionsIndex` with a 5 s flock
   timeout (`seance.go:419-440`). A cleanup function is `defer`red
   to remove the symlink when the subprocess exits.
7. Builds args `["--fork-session", "--resume", sessionID]`
   (`seance.go:245`) and clears `CLAUDECODE` / `CLAUDE_CODE_ENTRYPOINT`
   env vars via `clearClaudeCodeEnv` so the subprocess doesn't
   trigger the nested-session guard (`seance.go:247-250, 294-304`).
8. **One-shot mode** (`-p` prompt set): appends `-p <prompt>` and
   runs the subprocess with stdout/stderr inherited
   (`seance.go:252-266`).
9. **Interactive mode**: inherits stdin/stdout/stderr, prints
   "You are now talking to your predecessor. Ask them anything."
   and "Exit with `/exit` or Ctrl+C" (`seance.go:268-276`). Exit
   codes 0 and 130 (SIGINT) are treated as normal exits
   (`seance.go:278-286`).

### Session discovery

`sessionEvent` (`seance.go:78-83`):

```go
type sessionEvent struct {
    Timestamp string                 `json:"ts"`
    Type      string                 `json:"type"`
    Actor     string                 `json:"actor"`
    Payload   map[string]interface{} `json:"payload"`
}
```

The payload is read via `getPayloadString` (`seance.go:378-385`) which
safely extracts string fields. `session_id` and `topic` are the
primary payload keys consulted.

### Subcommands

None (terminal command).

### Flags

Defined in `init()` (`seance.go:66-75`):

| Flag                  | Type | Default | Purpose                                                  |
|-----------------------|------|---------|----------------------------------------------------------|
| `--role`              | str  | `""`    | Filter by actor role substring                           |
| `--rig`               | str  | `""`    | Filter by rig name substring                             |
| `--recent` / `-n`     | int  | `20`    | Max sessions to list                                     |
| `--talk` / `-t`       | str  | `""`    | Session ID (or prefix) to commune with                   |
| `--prompt` / `-p`     | str  | `""`    | One-shot prompt (with `--talk`)                          |
| `--json`              | bool | `false` | JSON output for the list                                 |

## Related

- [feed](feed.md) — feed reads the same `~/gt/.events.jsonl` stream
  via `feed.NewGtEventsSource`; seance scans it for
  `session_start` events specifically.
- [audit](audit.md) — audit is the historical reader for
  `.events.jsonl`; seance is the interactive "open a specific
  session" path over the same data.
- [checkpoint](checkpoint.md) — checkpoint writes predecessor
  context for the seance to load; both are tools for the
  handoff/resume workflow.
- [prime](prime.md) — prime emits the `session_start` events that
  seance discovers (see prime's `emitSessionEvent`).
- [../binaries/gt.md](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.

## Notes / open questions

- The cross-account symlink dance exists because Claude Code accounts
  don't share session storage. `findSessionLocation` walks every
  account's `projects/` tree looking for the session UUID in
  `sessions-index.json` files. For multi-account installs this can
  be slow.
- `config.ListAgentPresets` and `SupportsForkSession` flag live in
  `internal/config/`. Today only Claude-family presets support
  fork-session; pi/opencode presets do not.
- The `[GAS TOWN]` beacon string mentioned in the `Long` text
  (`seance.go:62`) is emitted by the session-start hook (`gt prime
  --hook`) so predecessor sessions are discoverable via Claude's
  `/resume`. This creates a dependency on prime.
- The event stream grows without bound; `discoverSessions` reads the
  entire file each call. A filtering index would help on long-lived
  towns.
- Non-fatal prefix resolution vs. exit-error-on-ambiguity: if a user
  pastes a 7-char prefix that matches two sessions, the command
  fails rather than prompting. There is no interactive disambiguation.
