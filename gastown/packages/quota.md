---
title: internal/quota
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/quota/state.go
  - /home/kimberly/repos/gastown/internal/quota/scan.go
  - /home/kimberly/repos/gastown/internal/quota/rotate.go
  - /home/kimberly/repos/gastown/internal/quota/executor.go
  - /home/kimberly/repos/gastown/internal/quota/keychain.go
  - /home/kimberly/repos/gastown/internal/quota/keychain_stub.go
tags: [package, quota, rotation, rate-limit, keychain, claude-code]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/quota

Claude Code account quota rotation for Gas Town. When a tmux session
hits a rate limit, this package scans tmux panes to detect the
condition, plans which fresh account to swap in, swaps the OAuth token
in macOS Keychain so `/resume` still finds the previous transcript,
and respawns the pane under the new account. State is persisted to
`mayor/quota.json` with flock-protected atomic writes.

**Go package path:** `github.com/steveyegge/gastown/internal/quota`
**File count:** 6 go files (including `keychain_stub.go` for non-darwin
builds), 5 test files
**Imports (notable):** stdlib (`encoding/json`, `os/exec`, `regexp`,
`sync`, `time`, `net/http` for token validation),
`github.com/gofrs/flock` for `mayor/.runtime/quota.lock`, and
[`internal/config`](config.md) (for `AccountsConfig`, `QuotaState`,
`BuildResumeCommand`, `GetSessionIDEnvVar`, `PrependEnv`),
[`internal/session`](session.md) (for `IsKnownSession`),
[`internal/util`](util.md) (for `ExpandHome`, `EnsureDirAndWriteJSON`),
`internal/constants` (for quota path and default patterns; no wiki
page yet).
**Imported by (notable):** [`gt quota`](../commands/quota.md) (the
scan/rotate/status command tree) and [`gt account`](../commands/account.md)
(Keychain-swap rotation primitives).

## What it actually does

### State manager (`state.go`)

- `Manager` (`state.go:24-31`) holds `townRoot` and exposes the file
  lock / state load / state save contract used by everything in the
  package. State lives at `mayor/quota.json`; the flock lives at
  `mayor/.runtime/quota.lock`.
- `lock()` / `WithLock(fn)` (`state.go:45-103`) — acquire a gofrs
  flock, run `fn`, unlock on return. `WithLock` is the preferred API
  for multi-step operations (Load + mutate + SaveUnlocked) because it
  eliminates the TOCTOU races that would occur if each mutation
  re-acquired the lock.
- `Load()` / `Save()` / `SaveUnlocked()` (`state.go:59-111`) — JSON
  read/write via `util.EnsureDirAndWriteJSON`. `SaveUnlocked` is the
  variant that assumes the caller already holds the lock.
- `MarkLimited(handle, resetsAt)` / `MarkAvailable(handle)`
  (`state.go:114-157`) — the fast path used by detection code. Each
  grabs the lock, reads, mutates one account entry, writes atomically.
- `AvailableAccounts(state)` / `LimitedAccounts(state)`
  (`state.go:160-182`) — list-accessors sorted by LastUsed ascending
  so the least-recently-used account is tried first.
- `ClearExpired(state)` (`state.go:244-271`) — walks limited accounts
  and marks them available when `ResetsAt` has passed. Exported for
  the scanner's auto-clear path so stale "limited" rows don't shrink
  the available pool.
- `ParseResetTime(resetsAt, reference)` (`state.go:285-328`) — parses
  human-readable reset strings like `"7pm (America/Los_Angeles)"` or
  `"3:30pm"` into a concrete `time.Time` in the given timezone. The
  regex at `state.go:274` handles `7pm` / `11am` / `3:30pm` / `7:00pm`
  forms.
- `RecordSwap` / `ClearSwap` / `ResolveSwapSourceDirs`
  (`state.go:214-239`) — manage the `ActiveSwaps` map that tracks
  which target config dirs are currently holding a swapped-in token
  from which source account handle.

### Pane scanner (`scan.go`)

- `ScanResult` (`scan.go:16-24`) — per-session detection output:
  session name, resolved `AccountHandle`, raw `ConfigDir`,
  `RateLimited` / `NearLimit` flags, the matched line, and a parsed
  `ResetsAt` string.
- `TmuxClient` interface (`scan.go:28-32`) — three methods
  (`ListSessions`, `CapturePane`, `GetEnvironment`). Separating the
  interface makes `Scanner` testable without a real tmux server.
- `Scanner` / `NewScanner` (`scan.go:35-63`) — compiles rate-limit
  regex patterns (case-insensitive) from the caller, or uses
  `constants.DefaultRateLimitPatterns`. `WithWarningPatterns`
  (`scan.go:67-82`) adds near-limit detection.
- `scanLines = 30`, `checkLines = 20` (`scan.go:88-96`) — the scanner
  captures 30 pane lines but only checks the bottom 20 for rate-limit
  patterns. The comment explains the tradeoff: if the rate limit was
  already resolved, new output pushes the old message above the check
  window, avoiding false positives.
- `ScanAll()` / `scanSession(session)` (`scan.go:100-189`) — the main
  loop. Filters to `isGasTownSession` (via `session.IsKnownSession`),
  captures the pane, applies hard-limit patterns first then warning
  patterns, and returns a result per session.
- `resolveAccountHandle(session)` (`scan.go:194-224`) — resolves which
  account a session is running under. Checks `GT_QUOTA_ACCOUNT` first
  (set by the keychain swap path so the mapping survives swapping),
  then falls back to matching `CLAUDE_CONFIG_DIR` against the
  registered accounts config.
- `parseResetTime(line)` (`scan.go:240-246`) — grabs the tail of a
  `"... resets 7pm (America/Los_Angeles)"` style line.

### Rotation planner (`rotate.go`)

- `RotateResult` (`rotate.go:11-19`) — per-session outcome: old/new
  account, `Rotated`, `ResumedSession` ID, `KeychainSwap` flag,
  `Error`.
- `RotatePlan` (`rotate.go:22-44`) — the full plan: `LimitedSessions`,
  `NearLimitSessions`, `AvailableAccounts`, per-session
  `Assignments`, per-config-dir `ConfigDirSwaps` (one keychain swap
  per config dir, not per session), and a `SkippedAccounts` map of
  handle→reason for tokens that were invalid at planning time.
- `PlanOpts` (`rotate.go:47-55`) — `FromAccount` targets all sessions
  using that account regardless of limit status (preemptive rotation);
  `IncludeNearLimit` pulls soft-warning sessions into the target list
  too.
- `PlanRotation(scanner, mgr, acctCfg, opts)` (`rotate.go:63-215`) —
  orchestrates: scan → load + ClearExpired state → pick target
  sessions by opts → select available accounts from **persisted
  state only** (the comment explicitly notes that using scan-detected
  limits here would let stale parked-rig panes block rotation) →
  validate each candidate token via `ValidateKeychainToken` (skipping
  and recording reasons for expired tokens) → collect unique config
  dirs from target sessions (one swap per config dir) → assign
  available accounts round-robin, skipping same-account candidates →
  expand config-dir assignments back to per-session assignments.

### Rotation executor (`executor.go`)

- `TmuxExecutor` interface (`executor.go:15-25`) — the write-side
  tmux operations: `SetEnvironment`, `GetPaneID`, `SetRemainOnExit`,
  `KillPaneProcesses`, `ClearHistory`, `RespawnPane`, plus dialog
  acceptance helpers.
- `Logger` (`executor.go:29-31`) — injection point for non-fatal
  warnings that don't depend on the CLI style package.
- `SessionLinker` (`executor.go:34-36`) — optional hook that symlinks
  the session's transcript into the target config dir so `--resume`
  can find it after rotation. May be nil to disable resume.
- `Rotator` / `NewRotator` (`executor.go:39-79`) — the injection
  point; everything needed to rotate is passed in by the caller.
- `Execute(plan, sessionOrder)` (`executor.go:87-140`) — the
  lifecycle: a single `WithLock` holds the quota lock across the
  whole operation, state is loaded once, per-session rotations run
  **concurrently** because each targets an independent tmux session
  (the only shared state is the in-memory `state.Accounts` map,
  protected by a `sync.Mutex`), and a single `SaveUnlocked` at the
  end persists results. This replaced N independent lock/load/save
  cycles that were racy and slow (kill-pane sleeps ~4s per session).
- `executeOne` (`executor.go:147-261`) — the per-session rotation.
  Structured in a validation phase (resolve old/new account from
  tmux env, build respawn command, attempt session-resume link, set
  remain-on-exit, verify pane exists) then a mutation phase (set
  `CLAUDE_CONFIG_DIR` on the tmux session, kill pane processes,
  clear history, respawn with the new account, accept startup
  dialogs, update `LastUsed` under the mutex).

### Keychain manipulation (`keychain.go`, darwin-only)

- Build tag `//go:build darwin` (`keychain.go:1`). Non-darwin systems
  get the stub at `keychain_stub.go` where every function returns
  `errNotDarwin` or a zero value.
- `KeychainServiceName(configDirPath)` (`keychain.go:35-51`) — computes
  the exact macOS Keychain service name Claude Code uses:
  `"Claude Code-credentials"` for the default `~/.claude`, or
  `"Claude Code-credentials-<sha256(configDir)[:8]>"` for non-default
  config dirs. The magic 8-hex-char suffix is load-bearing — it
  matches Claude Code's own name derivation.
- `ReadKeychainToken` / `WriteKeychainToken`
  (`keychain.go:54-76`) — thin shells around
  `security find-generic-password -w` and
  `security add-generic-password -U`.
- `SwapKeychainCredential(target, source)` (`keychain.go:85-110`) —
  backs up the target's token, reads the source's token, writes it
  into the target's keychain entry. This is the core of
  context-preserving rotation: the config dir doesn't change, so
  `/resume` still finds the previous session transcript, but the
  effective OAuth token is swapped. Returns a `*KeychainCredential`
  backup for later `RestoreKeychainToken`.
- `SwapOAuthAccount` / `RestoreOAuthAccount`
  (`keychain.go:125-201`) — the companion dance for
  `~/.claude.json`'s `oauthAccount` field (Claude Code caches
  accountUuid/organizationUuid there, and it must match the swapped
  token for the UI to identify correctly).
- `ValidateKeychainToken(configDir)` (`keychain.go:207-253`) — best-effort
  liveness check. Tries three strategies in order: parse as JSON
  with an `expires_at` field, parse as a JWT and check the `exp`
  claim, or treat the token as opaque and assume valid. The inner
  `validateTokenHTTP` (`keychain.go:257-278`) exists for completeness
  but is not called from the main path because Claude's OAuth flow
  would always return 401 against the bare `/v1/messages` endpoint.
- `SyncSwappedTokens(swapDirs)` (`keychain.go:292-324`) — propagates
  fresh tokens from source accounts to target keychain entries. After
  re-authentication of a source account writes a fresh token to the
  source's own keychain entry, this function copies it to any target
  entries that still hold the old (now stale) swapped-in token.

## Related wiki pages

- [gt](../binaries/gt.md) — binary containing the quota CLI surface.
- [gt quota](../commands/quota.md) — scan/rotate/status user interface.
- [gt account](../commands/account.md) — Keychain-aware account
  management (account add/remove/switch) that sits on top of this
  package.
- [internal/config](config.md) — supplies `AccountsConfig`, `QuotaState`,
  `BuildResumeCommand`, `GetSessionIDEnvVar`, `PrependEnv`.
- [internal/session](session.md) — provides
  `session.IsKnownSession` for the scanner's session filter.
- [internal/tmux](tmux.md) — implementation of the `TmuxClient` and
  `TmuxExecutor` interfaces.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The `.krc.yaml` pattern from krc doesn't apply here — quota state
  lives at `mayor/quota.json` (a proper JSON file) and uses a sibling
  `mayor/.runtime/quota.lock` via gofrs/flock.
- Non-darwin builds have working planning (scan, plan, validate) but
  will fail at execution time when `SwapKeychainCredential` returns
  `errNotDarwin` (`keychain_stub.go:21`). There is no alternative
  rotation backend for Linux/Windows.
- `ValidateKeychainToken`'s three-strategy decision
  (`keychain.go:207-253`) is conservative — it assumes unknown token
  formats are valid rather than rejecting them, so stale tokens in
  opaque formats will survive planning and fail at swap time with
  a runtime error instead.
