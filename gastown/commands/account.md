---
title: gt account
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/account.go
  - /home/kimberly/repos/gastown/internal/cmd/root.go
tags: [command, configuration, claude-code, accounts, multi-account]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [partial-completion, silent-suppression]
---

# gt account

Manages multiple Claude Code accounts (e.g., personal vs. work) so the
user can switch between them per-spawn or globally. Each account is a
handle that points at a `CLAUDE_CONFIG_DIR` directory containing a
distinct Claude login.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupConfig` ("Configuration")
**Polecat-safe:** no (no `AnnotationPolecatSafe` on the cobra.Command at
`/home/kimberly/repos/gastown/internal/cmd/account.go:24-39`)
**Beads-exempt:** no (not in `beadsExemptCommands` on `root.go:44-77`)
**Branch-check-exempt:** no (not in `branchCheckExemptCommands` on
`root.go:81-91`)

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/account.go:24-521`.

### Invocation

```
gt account list [--json]
gt account add <handle> [--email <email>] [--desc <description>]
gt account default <handle>
gt account status
gt account switch <handle>
```

Parent `accountCmd` uses `RunE: requireSubcommand`
(`account.go:28`) — calling `gt account` bare returns an error.

### Configuration store

Accounts live in an `accounts.json` file at the path returned by
`constants.MayorAccountsPath(townRoot)`
(`account.go:101`, `:174`, `:240`, `:303`, `:359`). Loaded and saved via
`config.LoadAccountsConfig` / `config.SaveAccountsConfig`. The config
is town-scoped — every command first calls `workspace.FindFromCwd()`
to resolve the town root.

Each account config directory sits under a base returned by
`config.DefaultAccountsConfigDir()` (`account.go:188`) — the only
direct reference is at `account.go:192`, where the account config dir
is built as `<baseDir>/<handle>`. The Long text at `account.go:59`
names it `~/.claude-accounts/<handle>`.

### Subcommands

#### `list` (`account.go:41-52`, `runAccountList` at `:95-164`)

1. Loads `accounts.json` (`account.go:102`). If missing, prints
   `No accounts configured.` plus an add hint and returns nil.
2. Builds `[]AccountListItem` from the map (`account.go:119-128`),
   sorts by handle (`:131-133`).
3. `--json` → `json.Encoder` output (`account.go:135-139`).
4. Text output (`account.go:142-162`): bold "Claude Code Accounts"
   header, one line per account with a `*` marker on the default,
   optional email, optional `(default)` tag, optional dim description.

#### `add <handle>` (`account.go:54-69`, `runAccountAdd` at `:166-230`)

1. Loads existing `accounts.json` or creates a new one
   (`account.go:177-180`).
2. Refuses duplicates: `account '%s' already exists`
   (`account.go:183-185`).
3. Builds the config directory path as
   `<DefaultAccountsConfigDir>/<handle>` (`account.go:188-192`).
4. `os.MkdirAll(configDir, 0755)` (`account.go:195`).
5. Calls `ensureSharedCommandsSymlink(configDir)` (`account.go:201`)
   — see below. Failure here prints a warning but does not fail.
6. Adds the account to the map with email/description/config-dir
   (`account.go:206-210`).
7. **First-account-becomes-default rule**: if `cfg.Default == ""`,
   set it to this handle (`account.go:213-215`).
8. Saves the config (`account.go:218-220`) and prints a completion
   message with the `CLAUDE_CONFIG_DIR=... claude` command to run,
   then the reminder to `/login` (`account.go:222-227`).

#### `default <handle>` (`account.go:71-84`, `runAccountDefault` at `:232-261`)

1. Loads config (`account.go:241`). Errors (not "missing" — any load
   error) are wrapped and returned.
2. Verifies the handle exists in `cfg.Accounts`, else
   `account '%s' not found` (`account.go:246-249`).
3. Sets `cfg.Default = handle` and saves (`account.go:252-258`).

#### `status` (`account.go:263-276`, `runAccountStatus` at `:297-349`)

1. Resolves the current account via
   `config.ResolveAccountConfigDir(accountsPath, "")`
   (`account.go:306`). The empty flag argument "shows the default
   resolution" (see the code comment).
2. If no handle resolves, prints `No account configured.` and
   returns nil (`account.go:311-316`).
3. Loads full config to get account details for the resolved handle
   (`account.go:322-329`).
4. Prints `Current Account` block: Handle, optional Email, optional
   Description, Config Dir (`account.go:332-340`).
5. If `GT_ACCOUNT` env var is set, appends
   `(set via GT_ACCOUNT environment variable)` (`account.go:342-343`).
   Otherwise if the resolved handle matches `cfg.Default`, appends
   `(default account)` (`account.go:344-346`).

#### `switch <handle>` (`account.go:278-295`, `runAccountSwitch` at `:351-465`)

This is the complex one. The goal is to make `~/.claude` a symlink
pointing at the target account's config dir, preserving any data
that was sitting in a non-symlinked `~/.claude`.

Step-by-step:

1. Load config, verify target exists (`account.go:360-375`). Error
   message on miss includes the list of available handles.
2. Resolve `~/.claude` path (`account.go:378-382`).
3. `os.Lstat(claudeDir)` (`account.go:385-388`). Non-`IsNotExist`
   errors are fatal.
4. If `~/.claude` is currently a symlink, read its target and try to
   match it against any registered account's `ConfigDir` to determine
   the current handle (`account.go:390-404`).
5. Short-circuit: if `currentHandle == targetHandle`, print
   `Already on account '%s'` and return (`account.go:406-410`).
6. **Real-directory migration branch** (`account.go:412-439`): if
   `~/.claude` exists as a plain directory (not a symlink), infer
   the current handle from `cfg.Default` if unknown, then
   `os.Rename(~/.claude, currentAcct.ConfigDir)` to move it into the
   current account's storage. If `~/.claude` is a dir and no default
   is known, return an error telling the user to set a default
   first.
7. **Symlink-removal branch** (`account.go:440-445`): if `~/.claude`
   is a symlink, `os.Remove` it so the next step can create a fresh
   one.
8. `os.Symlink(targetAcct.ConfigDir, claudeDir)` (`account.go:449`).
9. Updates `cfg.Default = targetHandle` and saves
   (`account.go:453-457`). **Switching also reassigns the default
   account.**
10. Prints the success message followed by the warning
    `⚠️  Restart Claude Code for the change to take effect`
    (`account.go:462`).

### `ensureSharedCommandsSymlink` (`account.go:467-504`)

Invoked by `add`. For a newly-created account config dir, it
symlinks `<configDir>/commands` → the real path of
`~/.claude/commands`, so SuperClaude-style global command files are
available regardless of which account is active via
`CLAUDE_CONFIG_DIR`.

- Resolves `filepath.EvalSymlinks(~/.claude/commands)` — if the
  global commands dir doesn't exist, silently returns nil
  (`account.go:478-483`).
- If the account already resolves to the same real directory, skip
  (`account.go:487-490`).
- If something exists at `<configDir>/commands`: if it's a symlink,
  remove and recreate; if it's a real directory, leave it alone
  (protects user's custom commands) (`account.go:492-501`).

### Flags

Declared in `init()` (`account.go:506-511`):

| Flag                | Scope                    | Type   | Default | Purpose                          |
|---------------------|--------------------------|--------|---------|----------------------------------|
| `--json`            | `account list`           | bool   | `false` | JSON output                      |
| `--email`           | `account add`            | string | `""`    | Account email address            |
| `--desc`            | `account add`            | string | `""`    | Account description              |

## Docs claim

### Source
- `/home/kimberly/repos/gastown/internal/cmd/account.go:29-38` — Cobra `Long` text on the parent `accountCmd`.

### Verbatim

> Manage multiple Claude Code accounts for Gas Town.
>
> This enables switching between accounts (e.g., personal vs work) with
> easy account selection per spawn or globally.
>
> Commands:
>   gt account list              List registered accounts
>   gt account add <handle>      Add a new account
>   gt account default <handle>  Set the default account
>   gt account status            Show current account info

## Drift

See forward-link: [../drift/README.md](../drift/README.md).

### Parent `Long` text enumerates 4 subcommands; 5 are registered

- **Claim source:** Cobra `Long` text at `/home/kimberly/repos/gastown/internal/cmd/account.go:34-38`.
- **Docs claim:** the "Commands:" list advertises exactly four subcommands — `list`, `add`, `default`, `status`. A user reading `gt account --help` would reasonably conclude those are the complete subcommand set.
- **Code does:** `init()` at `/home/kimberly/repos/gastown/internal/cmd/account.go:513-518` registers five subcommands on `accountCmd`: `accountListCmd`, `accountAddCmd`, `accountDefaultCmd`, `accountStatusCmd`, **and `accountSwitchCmd`**. `accountSwitchCmd` is declared at `account.go:278-295` with `Use: "switch <handle>"`, `Short: "Switch to a different account"`, and a multi-step `Long` describing the symlink rewrite at `~/.claude`. It is fully functional (`runAccountSwitch` at `account.go:351-465`) — the only path that actually flips the live account — but the parent `Long` text never mentions it. Users discover `switch` only by running `gt account` (which falls through `requireSubcommand`) and reading the cobra-generated `Available Commands:` block, which does include `switch`.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — extend the Commands list in `accountCmd.Long` at `account.go:34-38` to include `switch <handle>`, matching the actual init() registration order. The `Long` texts on the individual sub-command vars (`accountSwitchCmd.Long` at `account.go:281-288`) are already correct and need no change.
- **Release position:** `in-release` (parent `Long` text byte-identical at `v1.0.0:internal/cmd/account.go:29-38`; `accountSwitchCmd` already registered in the v1.0.0 init block).

## Failure modes

### Precondition violations (what does it assume?)

- **`runAccountList` treats any config load error as "no accounts":** `config.LoadAccountsConfig` at `account.go:102-109` catches all errors — not just "file not found" — and prints "No accounts configured." A corrupt JSON file produces the same user-visible output as an empty config, with no error surfaced. **Absent** — predicted bug surface: user with a malformed `accounts.json` gets a misleading "no accounts" message instead of a parse error.

### Partial completion (what doesn't it clean up?)

- **`switch` removes old symlink before creating new one:** `account.go:440-445` removes the existing `~/.claude` symlink, then `account.go:449` creates the new one. If `os.Symlink` fails (e.g., target `ConfigDir` doesn't exist), `~/.claude` is gone and Claude Code has no config directory until manually fixed. **Absent** — no rollback: the old symlink is not restored on failure.
- **`switch` real-directory migration removes target before rename:** `account.go:427-429` calls `os.RemoveAll(currentAcct.ConfigDir)` to clear the way for `os.Rename(~/.claude, ...)`. If the rename at `account.go:433` subsequently fails, the previously-existing account config dir has been destroyed. **Absent** — no rollback of the `RemoveAll`.
- **`add` creates config dir before save succeeds:** `account.go:195` calls `os.MkdirAll` to create the account config directory. If `config.SaveAccountsConfig` fails at `account.go:218-220`, the directory exists on disk but is not registered in `accounts.json`. Re-running `add` with the same handle works (the dir already exists), but the orphan dir is never cleaned up on error. **Absent** — no cleanup of the created directory if save fails.

### Silent suppression (what errors are swallowed?)

- **`ensureSharedCommandsSymlink` silently discards symlink removal error:** `account.go:496` uses `_ = os.Remove(acctCmds)` when removing a stale symlink before recreating it. If the remove fails, `os.Symlink` at `account.go:503` will also fail, and that error is returned — but the root cause (permission denied on remove) is lost. **Present** (the downstream `os.Symlink` error propagates) but the suppressed remove error obscures diagnosis.
- **`ensureSharedCommandsSymlink` failure in `add` is a warning, not an error:** `account.go:201-203` catches the symlink-setup error and prints a warning via `style.PrintWarning`. The account is still added. **Present** — intentional degradation; the warning is visible.

## Related

- [gt](../binaries/gt.md) — parent binary.
- [README.md](README.md) — command tree index.
- [whoami.md](whoami.md) — peer identity command; `whoami` reports
  the current agent's role/rig/session, `account` reports the active
  Claude login.
- [config.md](config.md) — sibling Configuration command; `account`
  uses a distinct `accounts.json` store rather than town
  `settings/config.json`.
- [status.md](status.md) — `gt status` does not currently surface
  account state as of this reading (uncited — worth verifying).
- [prime.md](prime.md) — `GT_ACCOUNT` env var is documented here as
  having highest priority for account resolution (`account.go:269`);
  `prime` or session-bring-up code likely sets it.

## Notes / open questions

- **`switch` vs `default`**: `switch` changes both the live
  `~/.claude` symlink and `cfg.Default` (`account.go:454`). `default`
  only changes `cfg.Default`. Subtle: running `gt account default X`
  while `~/.claude` still points at account Y leaves the user in a
  state where `status` says "X (default)" but Claude Code is still
  reading from Y's config dir. The asymmetry is non-obvious.
- **Accounts config dir**: `config.DefaultAccountsConfigDir()` is
  called via two paths — `add` builds `<base>/<handle>`, but the
  Long text at `account.go:59` names `~/.claude-accounts/<handle>`.
  The canonical path should be mapped in a `packages/config.md`
  page.
- **First-account-becomes-default** is a hidden invariant — a user
  adding a `work` account as their first will find it silently
  becomes the default, regardless of whether they wanted that. The
  Long text does not mention this side effect.
- **Real `~/.claude` directory migration** is destructive: if the
  current `~/.claude` is a plain dir and the inferred default is
  wrong, it will be renamed into the wrong account's config dir.
  Worth checking whether there is a safer path (copy + verify +
  remove) in a future hardening pass.
- `GT_ACCOUNT` is not yet in any env-var rollup page.
