---
title: gt quota
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/quota.go
tags: [command, services, quota, accounts, rate-limits, keychain, rotation]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt quota

Manages Claude Code account quota rotation. Detects rate-limited
sessions, plans rotations from a pool of registered accounts, and
performs context-preserving credential swaps via the macOS Keychain so
sessions can `--continue` after a rotation instead of losing their
conversation history.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupServices` ("Services")
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/quota.go:36-981`.

### Invocation

```
gt quota                            # parent — RunE: requireSubcommand
gt quota status [--json]
gt quota scan [--update] [--json]
gt quota rotate [--from H] [--idle] [--dry-run] [--json]
gt quota clear [handle...]
gt quota watch [--interval D] [--dry-run]
```

The bare `gt quota` is a parent dispatched via `requireSubcommand`
(`quota.go:40`).

### Subcommands

**`status`** (`quota.go:53-65`, run: `quota.go:78-199`)
Loads `MayorAccountsPath(townRoot)` via `config.LoadAccountsConfig`
and the per-town quota state via `quota.NewManager(townRoot).Load()`.

- Calls `mgr.EnsureAccountsTracked(state, acctCfg.Accounts)` so newly-
  added accounts get a default quota row.
- **Auto-clears expired limits** via `mgr.ClearExpired(state)` and
  persists if any were cleared (`quota.go:110-114`).
- Renders text or JSON via `printQuotaStatusText` /
  `printQuotaStatusJSON`.
- The text view uses `*` to mark the default account, color badges
  for `available` / `limited` / `cooldown` / `unknown`, and prints a
  summary `N available, M limited`.

**`scan`** (`quota.go:206-221`, run: `quota.go:223-257`)
Captures recent pane output from every Gas Town tmux session and
checks for rate-limit indicators via `quota.NewScanner(t, nil,
acctCfg).ScanAll()`.

- Reports per-session rate-limited / near-limit findings with the
  resolved account handle and (when known) the predicted reset time.
- `--update` (`scanUpdate` flag) persists detected limits into the
  quota state via `updateQuotaState` (`quota.go:259-283`), which
  takes the manager's lock and writes
  `Status: QuotaStatusLimited`, `LimitedAt: now`, `ResetsAt:
  r.ResetsAt`. Falls through silently if no accounts are configured.

**`rotate`** (`quota.go:359-385`, run: `quota.go:387-591`)
The big one. Plans and executes account rotations.

Preconditions:
- An accounts config must exist (otherwise: `no accounts configured
  (run 'gt account add' first)`).
- At least 2 accounts (otherwise: `need at least 2 accounts for
  rotation (have N)`).
- If `--from <handle>` is set, that account must exist.

Planning (`quota.go:411-422`):
- Build `quota.NewScanner(t, nil, acctCfg)`.
- `quota.PlanRotation(scanner, mgr, acctCfg, quota.PlanOpts{
  FromAccount: rotateFrom})` returns
  `LimitedSessions`, `Assignments` (session → new account, in LRU
  order), and `SkippedAccounts` (handle → reason).

**Important non-persistence**: scan-detected limits are *not*
persisted into quota state during planning (`quota.go:424-428`).
Stale sessions (e.g., parked rigs with old rate-limit messages still
in their pane) would otherwise poison the available pool. Account
state is updated only after successful rotation execution
(`LastUsed`, recorded in `executeKeychainRotation`).

Reporting (`quota.go:430-541`):
- Counts `noConfigDir` sessions (sessions with no
  `CLAUDE_CONFIG_DIR` and no resolvable account).
- Counts `unassignable` sessions (limited but no available account
  to swap to).
- With `--idle`, filters `Assignments` down to sessions that are
  `t.IsIdle(session)` — busy sessions are skipped with a `(busy)`
  marker so the rotation doesn't interrupt active agents.
- Prints the rotation plan: `<session> <oldAccount> → <newAccount>`.
- For skipped accounts (e.g., expired login), prints
  `Run: claude /login (in CLAUDE_CONFIG_DIR=…)`.

Execution (`quota.go:555-590`):
- Iterates sorted assignments and calls `executeKeychainRotation` per
  session, passing a shared `swappedConfigDirs` map so multiple
  sessions sharing a config dir only trigger one keychain swap.
- Per-session result emits `<session> → <newAccount>` with
  `(resumed)` and/or `[keychain]` suffixes when applicable.
- `--dry-run` stops after the plan and emits the JSON plan if
  `--json` is set.

**`clear [handle...]`** (`quota.go:593-644`)
Marks one or more accounts as available again via
`mgr.MarkAvailable(handle)`. With no args, iterates all accounts in
state and clears every `QuotaStatusLimited` or `QuotaStatusCooldown`
row.

**`watch`** (`quota.go:812-869`)
Continuously polls all sessions on `--interval` (default 5 minutes)
for both hard rate limits and near-limit warning signals. When a
session is approaching its limit, rotates proactively before the hard
429 hits. Handles `SIGTERM`/`SIGINT` for graceful shutdown.

Watch cycle (`runWatchCycle`, `quota.go:872-958`):
- Builds a scanner with warning patterns enabled
  (`scanner.WithWarningPatterns(nil)`).
- **Syncs swapped tokens** via
  `quota.SyncSwappedTokens(quota.ResolveSwapSourceDirs(state.ActiveSwaps,
  acctCfg.Accounts))` — if a *source* account re-authenticated since
  the last rotation, the fresh token is propagated to all
  *target* keychain entries that were previously swapped from it.
- Plans rotation with `IncludeNearLimit: true` so near-limit sessions
  are rotated alongside hard-limited ones.
- Reports `LIMITED` and `NEAR` findings with timestamps.
- Executes via the same `executeKeychainRotation` path unless
  `--dry-run`.

### `executeKeychainRotation` — context-preserving rotation

`quota.go:663-802`. **The core mechanism that makes Gas Town quota
rotation distinct from simply restarting Claude with a different
config dir.** Instead of changing `CLAUDE_CONFIG_DIR` (which would
destroy `/resume` context), it swaps the OAuth token in the macOS
Keychain so the *same* config dir suddenly authenticates as a
different account.

Steps per session:

1. Read the session's current `CLAUDE_CONFIG_DIR` from tmux env via
   `t.GetEnvironment(session, "CLAUDE_CONFIG_DIR")`. Falls back to
   `$HOME/.claude` if unset.
2. Resolve `OldAccount` by matching `currentConfigDir` against any
   `acctCfg.Accounts[h].ConfigDir` (with `util.ExpandHome`).
3. Resolve `sourceConfigDir` from the *new* account's config dir.
4. **Keychain swap (deduplicated)**: if this `currentConfigDir` has
   not yet been swapped in this batch,
   `quota.SwapKeychainCredential(currentConfigDir, sourceConfigDir)`
   moves the fresh token into the current dir's keychain entry. The
   returned backup is recorded in `swappedConfigDirs`.
5. **OAuth account identity swap**:
   `quota.SwapOAuthAccount(currentConfigDir, sourceConfigDir)` updates
   `.claude.json` so Claude Code identifies as the new account
   (correct `accountUuid` / `organizationUuid` for rate limits).
6. Build a restart command via `buildRestartCommandWithOpts({
   ContinueSession: true})`. The `--continue` flag is what makes
   `/resume` work without a fresh handoff cycle. Sessions that can't
   be restarted (e.g., `hq-boot`, `hq-deacon`) still benefit from the
   keychain swap and are marked rotated without a respawn.
7. **Keep the SAME config dir** (`quota.go:734-741`). Inject env via
   `config.PrependEnv`:
   - `CLAUDE_CONFIG_DIR=<currentConfigDir>` (unchanged)
   - `GT_QUOTA_ACCOUNT=<newAccount>` so the scanner can resolve the
     active account (the config dir still maps to the *old* account).
8. Tmux respawn:
   - `t.GetPaneID(session)`
   - `t.SetRemainOnExit(pane, true)` so the pane survives the restart
   - `t.KillPaneProcesses(pane)`
   - `t.ClearHistory(pane)`
   - `t.RespawnPane(pane, restartCmd)`
9. `t.SetEnvironment(session, "GT_QUOTA_ACCOUNT", newAccount)` so the
   tmux session env (where the scanner reads) is in sync with the
   process env injected via the shell export.
10. **Quota state update**, under `mgr.WithLock`:
    - `state.Accounts[newAccount].LastUsed = now (UTC RFC3339)`
    - `quota.RecordSwap(state, currentConfigDir, newAccount)` so
      `SyncSwappedTokens` (called by `watch`) can later propagate
      fresh tokens if the source account re-authenticates.

### Flags

(`quota.go:960-973`) — all subcommand-scoped:

| Subcommand | Flag        | Type     | Default |
|------------|-------------|----------|---------|
| `status`   | `--json`    | bool     | false   |
| `scan`     | `--json`    | bool     | false   |
| `scan`     | `--update`  | bool     | false   |
| `rotate`   | `--dry-run` | bool     | false   |
| `rotate`   | `--json`    | bool     | false   |
| `rotate`   | `--from`    | string   | ""      |
| `rotate`   | `--idle`    | bool     | false   |
| `watch`    | `--interval`| duration | 5m      |
| `watch`    | `--dry-run` | bool     | false   |

The shared `quotaJSON` package var (`quota.go:33`) is bound to
`--json` on `status`, `scan`, and `rotate` — so it's a single global
toggle, not per-flag state. (Drift risk if flags are added in
non-init order.)

### Account / state file paths

- Accounts: `constants.MayorAccountsPath(townRoot)` — loaded via
  `config.LoadAccountsConfig`. Provides handle → email/`ConfigDir`
  metadata.
- Quota state: `quota.NewManager(townRoot)` — file path lives in the
  `internal/quota` package. Holds `Accounts[handle].Status`,
  `LimitedAt`, `ResetsAt`, `LastUsed`, plus `ActiveSwaps` for the
  swap-mapping table that `watch` uses.

## Related

- [account](./account.md) — manages the account pool that this
  command rotates between
- [costs](./costs.md) — token / dollar accounting (separate axis from
  rate-limit rotation)
- [metrics](./metrics.md) — dashboarding for usage signals
- [scheduler](./scheduler.md) — adjacent scheduling primitives
- [feed](./feed.md) — observability stream that may consume rotation
  events
- [../packages/quota.md](../packages/quota.md) — the Claude Code
  rate-limit detection + Keychain token rotation library (darwin-only).

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/quota.go:47-54` — Cobra `Long` text

### Verbatim
> Commands:
>   gt quota status            Show account quota status
>   gt quota scan              Detect rate-limited sessions
>   gt quota rotate            Swap blocked sessions to available accounts
>   gt quota clear             Mark account(s) as available again

## Drift

### `quotaCmd.Long` COMMANDS block lists 4 subcommands; 5 registered
- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/quota.go:47-54`
- **Docs claim:** COMMANDS block enumerates 4 subcommands: `status`, `scan`, `rotate`, `clear`.
- **Code does:** `init()` at `quota.go:974-978` registers 5 subcommands: `status`, `scan`, `rotate`, `clear`, **`watch`**. The `watch` subcommand (`quota.go:812-869`) continuously polls sessions for rate limits and rotates proactively. It is a long-running foreground process with `--interval` and `--dry-run` flags.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code`
- **Release position:** `in-release`

## Failure modes

No failure modes discovered. Read-only query command. Fetches API usage data and displays it. All errors propagated. No state mutations.

## Notes / open questions

- **`buildRestartCommandWithOpts`** is referenced from
  `executeKeychainRotation` but defined elsewhere in `internal/cmd`.
  Worth a follow-up grep — likely shared with other rotation /
  respawn commands.
- **Linux/Windows support**: the keychain swap is macOS-specific (via
  the `security` CLI inside `quota.SwapKeychainCredential`). On
  Linux, behavior is unclear — does the swap silently no-op? Worth
  testing or grounding in `internal/quota`.
- **`GT_QUOTA_ACCOUNT` resolution**: the scanner uses this env var as
  the source of truth when `CLAUDE_CONFIG_DIR` no longer maps to the
  "right" account (because of the keychain swap). Worth its own
  small concept page — this is a non-obvious shadow attribute.
- **`ActiveSwaps` schema**: stored in `state.ActiveSwaps`, used by
  `watch`'s sync step. Schema lives in `internal/quota/state.go` (or
  similar) — worth grounding next time someone touches the package.
- **No `gt quota status --json` schema doc**: the `QuotaStatusItem`
  shape (`quota.go:68-76`) is the de-facto contract. Add to a
  data-types page if a consumer pins to it.
- **`watch` is the only subcommand that runs indefinitely**: fits the
  Services group, but is structurally closer to a daemon. Worth
  noting whether it's supposed to be supervised by `gt daemon` /
  launchd or run interactively.
- **`scan --update` race**: `updateQuotaState` takes the manager
  lock, but the scan itself is unlocked. If `rotate` runs
  concurrently with `scan --update`, they may serialize on the lock —
  worth confirming there's no read-modify-write tear.
