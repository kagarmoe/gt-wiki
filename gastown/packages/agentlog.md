---
title: internal/agentlog
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/agentlog/event.go
  - /home/kimberly/repos/gastown/internal/agentlog/claudecode.go
  - /home/kimberly/repos/gastown/internal/agentlog/opencode.go
tags: [package, agent-log, telemetry, claudecode, opencode, jsonl]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/agentlog

Pluggable adapter layer for watching AI-agent conversation logs and
streaming normalized events upstream. Powers the `gt agent-log` command,
which feeds conversation content (text turns, tool calls, thinking, token
usage) into OTEL telemetry so Gas Town can observe what its managed agents
are actually doing.

**Go package path:** `github.com/steveyegge/gastown/internal/agentlog`
**File count:** 3 go files (`event.go`, `claudecode.go`, `opencode.go`), 1
test file.
**Imports (notable):** stdlib only (`bufio`, `context`, `encoding/json`,
`os`, `path/filepath`, `strings`, `time`). No internal deps; consumers
translate `AgentEvent` into telemetry themselves.
**Imported by (notable):** [`gt`](../binaries/gt.md)'s `agent-log`
command (see [internal/telemetry](telemetry.md) for the OTEL side that
actually receives these events).

## What it actually does

### Extension point: `AgentAdapter` (event.go)

Source: `/home/kimberly/repos/gastown/internal/agentlog/event.go`.

- `AgentEvent` struct (`event.go:16-31`) is the normalized event type every
  adapter emits. Fields: `AgentType`, `SessionID` (Gas Town tmux session
  name), `NativeSessionID` (agent-native UUID), `EventType` (`text`,
  `tool_use`, `tool_result`, `thinking`, `usage`), `Role`
  (`assistant`/`user`), `Content`, `Timestamp`, plus four token-count
  fields (`InputTokens`, `OutputTokens`, `CacheReadTokens`,
  `CacheCreationTokens`) that are non-zero only for `EventType == "usage"`.
- `AgentAdapter` interface (`event.go:35-46`): implement `AgentType()
  string` and `Watch(ctx, sessionID, workDir, since) (<-chan AgentEvent,
  error)`. The channel is closed when `ctx` is canceled or a fatal error
  occurs. `since` filters out log files modified before that time — zero
  means "any age".
- `NewAdapter(agentType string) AgentAdapter` (`event.go:50-59`) —
  registry. Known types: `"claudecode"` (also the default on empty
  string), `"opencode"`. Unknown types return `nil`.

### `ClaudeCodeAdapter` (claudecode.go)

The only fully implemented adapter. Watches Claude Code's JSONL
conversation files under `~/.claude/projects/<hash>/<session-uuid>.jsonl`
where `<hash>` is the working-directory path with `/` replaced by `-`
(`claudeProjectDirFor`, `claudecode.go:102-120`). Windows-aware:
normalizes backslashes to forward slashes and strips a drive-letter
prefix (e.g. `C:`) before hashing, matching Claude Code's cross-platform
storage layout.

Flow (`ClaudeCodeAdapter.Watch`, `claudecode.go:52-95`):

1. Resolve `projectDir` from `workDir`.
2. Spawn a goroutine that loops:
   - `waitForNewestJSONL` (`claudecode.go:125-140`) polls `projectDir`
     every 500 ms (`watchPollInterval`, `claudecode.go:20`) for the most
     recently modified `*.jsonl` whose mod time is `>= since`. Deadline
     30 s (`watchFileTimeout`, `claudecode.go:23`) — on timeout, reset
     `since` to now and retry so agent restarts or late starts don't
     wedge the watcher.
   - `tailJSONL` (`claudecode.go:187-238`) opens the file, reads all
     existing lines, then polls at `watchPollInterval` for new content.
     At every EOF it checks whether a newer JSONL file has appeared in
     `projectDir`; if so, returns so the outer loop re-runs
     `waitForNewestJSONL` to pick up the next Claude session (within one
     500 ms tick). Agent restarts never lose events because the current
     file is fully tailed before switching.
3. `parseClaudeCodeLine` (`claudecode.go:283-361`) parses one JSONL line
   into zero or more `AgentEvent`s:
   - Drops entries whose type is not `"assistant"` or `"user"`.
   - Walks `message.content` blocks and emits one event per block for
     types `text`, `thinking`, `tool_use` (content = `name + ": " +
     json-input`), and `tool_result`.
   - For assistant turns with a `usage` object, emits a single extra
     `EventType == "usage"` event carrying all four token counts. Only
     emitted if at least one of the four token counts is non-zero — so
     cache-only turns (input 0, output 0, cache-read non-zero) are still
     recorded for cost accounting.

### `OpenCodeAdapter` (opencode.go)

Stub — `AgentType() string` returns `"opencode"` but `Watch(...)` returns
`fmt.Errorf("opencode adapter not yet implemented")` (`opencode.go:19-21`).
Placeholder for future OpenCode support; the file references
`github.com/sst/opencode` as the target storage format. Adding OpenCode
means filling out `Watch` here.

## Related wiki pages

- [gt](../binaries/gt.md) — host binary; `gt agent-log` is the consumer.
- [internal/telemetry](telemetry.md) — OTEL plumbing that receives
  `AgentEvent`s downstream. `agentlog` itself doesn't touch OTEL; the
  command handler does the translation.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The spec note that agentlog is "Unix-only; Windows stub" does not match
  the code. There is no `*_windows.go` stub in this directory: the
  Claude adapter's `claudeProjectDirFor` has explicit Windows handling
  (drive-letter strip), the rest is pure stdlib, and file polling works
  on both platforms. Flagging as possible drift in the batch spec or in
  the note author's expectation — the package itself compiles cross-
  platform.
- `NewAdapter` silently returns `nil` for unknown types. Callers that
  don't check get a nil-deref on `adapter.Watch(...)`. The `gt agent-log`
  command presumably validates `--agent` before calling; worth spot-
  checking when the command page is written.
- `waitForNewestJSONL`'s 30-second deadline is reset-and-retry rather
  than fatal, so a watcher started before Claude launches will patiently
  wait for the first file to appear. No upper bound on total wait — ctx
  cancellation is the only exit.
- `parseClaudeCodeLine` emits one event per content block but batches
  token usage into a single "usage" event per assistant turn. Downstream
  OTEL aggregation should not double-count by summing content-block-
  scoped fields.
