---
title: gt install
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/install.go
tags: [command, workspace, setup, install, hq, dolt, beads]
phase3_audited: 2026-04-15
phase3_findings: [cobra-drift, drift]
phase3_severities: [wrong]
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase5_audience: user
phase8_audited: 2026-04-17
phase8_findings: [partial-completion]
detail_depth: {params: 2, data_flow: 2, errors: 2, side_effects: 2}
---

# gt install

Create (or reinitialize) a Gas Town HQ — the top-level workspace
directory containing the Mayor, Deacon, rig registry, town beads
database, and configuration files. This is usually the **first** gt
command a user runs on a new machine.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWorkspace` ("Workspace") (`install.go:50`)
**Polecat-safe:** no
**Beads-exempt:** yes (in `beadsExemptCommands` on `root.go:44-77`
— has to work before a beads DB exists)
**Branch-check-exempt:** yes (in `branchCheckExemptCommands` on
`root.go:81-91` — runs before the town is even on `main`)
**SilenceUsage:** yes — the command sets `SilenceUsage: true`
(`install.go:76`) so errors don't dump cobra usage output.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/install.go`
(813 lines). Sibling tests: `install_test.go`,
`install_integration_test.go`. Registered on `rootCmd` in
`install.go:79-93`.

### Invocation

```
gt install [path]
    [--force|-f] [--name|-n <name>] [--owner <email>] [--public-name <name>]
    [--no-beads] [--git] [--github <owner>/<repo>] [--public]
    [--shell] [--wrappers] [--supervisor] [--dolt-port <port>]
```

If `path` is omitted, the current directory is used
(`install.go:97-100`).

### Preflight (before any mutation)

Ordered, in `runInstall` (`install.go:95-202`):

1. **Path expansion.** `~` is expanded; the path is resolved to
   absolute (`install.go:103-114`).
2. **Town name.** Defaults to `filepath.Base(absPath)` unless
   `--name` is set (`install.go:116-120`).
3. **Already-a-workspace guard** (`install.go:122-133`). If the
   target is already a Gas Town HQ and `--force` is not set, one
   of two things happens:
    - If **only** `--wrappers` was requested, install wrappers
      and return successfully. This is a special escape hatch.
    - Otherwise, abort with
      `"directory is already a Gas Town HQ (use --force to reinitialize)"`.
4. **Inside-existing-workspace guard** (`install.go:135-142`). If
   the current path is *nested* inside a different HQ (e.g. a
   crew worktree under `~/gt/<rig>/crew/joe`), aborts with a
   multi-line error whose remediation is `"Did you mean to update
   the binary? Run 'make install' in the gastown repo."` unless
   `--force`.
5. **Ensure `bd` (beads) is available** via `deps.EnsureBeads(true)`
   (`install.go:144-149`), skipped if `--no-beads`. See
   [deps package](../packages/deps.md) for the version pinning and
   `GOBIN=~/.local/bin` auto-install semantics.
6. **Dolt preflight** (`install.go:151-202`), skipped if
   `--no-beads`:
    - Verify `dolt` is on `PATH` (soft — if absent, the whole
      block is skipped).
    - `doltserver.EnsureDoltIdentity()` — aborts with an inline
      remediation message (`dolt config --global --add user.name …`)
      if the global Dolt identity is not configured.
    - Resolve the port: `installDoltPort` > env `GT_DOLT_PORT` >
      `doltserver.DefaultPort`. If `--dolt-port` was passed,
      `GT_DOLT_PORT` is exported for the rest of the process.
    - `doltserver.CheckPortAvailable(port)` — if the port is
      held, try to connect to it with a MySQL client
      (`install.go:173-184`). A **usable** Dolt server on the
      port is reused (prints `"Using existing Dolt server on port
      <N>"`). Otherwise, builds an enriched error message via
      `doltserver.PortHolder` and `doltserver.FindFreePort` and
      suggests `gt install <orig args> --dolt-port <free>`.

Both preflights exist specifically so a partial install is never
left behind — the comment at `install.go:151-153` is explicit
about this.

### Scaffolding steps

All under the same `runInstall` function (`install.go:204-499`).
Many steps are soft-failure: on error they print a warning with
`style.Dim.Render("⚠")` and continue.

1. `os.MkdirAll(absPath, 0755)` — create the HQ root.
2. Create `mayor/` (`install.go:213-217`).
3. Resolve **owner** from `--owner` or `git config user.email`
   (`install.go:220-226`).
4. Resolve **public name** from `--public-name` or the town name
   (`install.go:228-232`).
5. Create or preserve `mayor/town.json` (`install.go:234-256`). If
   the file already exists, it is left alone — re-running install
   must not clobber existing town config.
6. Create or preserve `mayor/rigs.json` (`install.go:257-275`).
7. Create `CLAUDE.md` and `AGENTS.md` (symlink to `CLAUDE.md`) at
   the town root via `createTownRootAgentMDs`
   (`install.go:277-289, 514-549`). This is a **minimal
   non-role-specific identity anchor** — the real role context
   comes from `gt prime`. The file's explicit purpose is to
   survive compaction for Mayor and Deacon (crew and polecats
   have their own nested git trees and won't inherit it).
8. Create `mayor/.claude/settings.json` via
   `runtime.EnsureSettingsForRole` (`install.go:292-305`). **Must
   be in `mayor/.claude/`, NOT `<town>/.claude/`** — a comment at
   `install.go:292-294` explains why: settings at the town root
   would be found by all agents through directory traversal and
   break their CWD assumptions.
9. Create `deacon/` and `deacon/.claude/settings.json`
   (`install.go:307-318`).
10. Create `deacon/dogs/boot/` directory to preempt a doctor
    warning (`install.go:320-325`).
11. Create `plugins/` (town-level) (`install.go:327-334`).
12. Create `mayor/daemon.json` via
    `config.EnsureDaemonPatrolConfig` (`install.go:336-342`).
13. **Initialize git BEFORE beads** (`install.go:344-351`) if
    `--git` or `--github` was passed. Calls `InitGitForHarness`
    from [git-init.md](git-init.md). Comment:
    "The fingerprint is required for the daemon to start properly."
14. **Town beads init** (`install.go:353-407`, skipped if
    `--no-beads`):
    - If `dolt` is on `PATH`: `doltserver.InitRig(absPath, "hq")`
      to create the HQ Dolt database, then `doltserver.Start` to
      bring the server up. The server is intentionally left
      running after install (comment at `install.go:367-369`).
    - `initTownBeads(absPath)` — runs `bd init --prefix hq
      --server --server-port <N>` after waiting up to 10 seconds
      for the Dolt server to accept MySQL queries (`install.go:570-591`
      in the helper).
    - `formula.ProvisionFormulas(absPath)` — provisions
      embedded formulas into `.beads/formulas/`.
    - `initTownAgentBeads(absPath)` — creates town-level Mayor
      and Deacon agent beads with `hq-` prefix.
    - `bd config set routing.mode explicit` — required by
      [doctor.md](doctor.md).
15. **Overseer detection** via `config.DetectOverseer`
    (`install.go:409-420`). Saves to
    `config.OverseerConfigPath(absPath)`.
16. **Escalation config** via
    `config.SaveEscalationConfig(..., NewEscalationConfig())`
    at `config.EscalationConfigPath(absPath)`
    (`install.go:422-428`).
17. **Slash commands** via `templates.ProvisionCommands(absPath)`
    (`install.go:430-436`) — populates `.claude/commands/` for
    all agents. The comment at `install.go:431-432` notes that
    all agents inherit these via Claude's directory traversal,
    so no per-workspace copy is needed.
18. **Hook sync** via `hooks.DiscoverTargets(absPath)` +
    `syncTarget` in a loop (`install.go:438-449`). Generates
    `.claude/settings.json` files for all hook targets.
19. **Optional: shell integration** (`install.go:451-463`), if
    `--shell` was passed. Calls `shell.Install()` and
    `state.Enable(Version)`. See [shell.md](shell.md).
20. **Optional: wrappers** (`install.go:465-472`), if
    `--wrappers` was passed. Installs `gt-codex`, `gt-gemini`,
    `gt-opencode` via `wrappers.Install()`.
21. **Optional: supervisor** (`install.go:474-482`), if
    `--supervisor` was passed. Calls
    `templates.ProvisionSupervisor` to install a launchd /
    systemd unit for daemon auto-restart.
22. Print "HQ created successfully!" and a numbered "Next
    steps" list (`install.go:484-499`).

### Subcommands

None. `gt install` is a leaf command.

### Flags

| flag | type | default | source |
|---|---|---|---|
| `--force` / `-f` | bool | `false` | `install.go:80` |
| `--name` / `-n` | string | `""` (basename of path) | `install.go:81` |
| `--owner` | string | `""` (git config user.email) | `install.go:82` |
| `--public-name` | string | `""` (town name) | `install.go:83` |
| `--no-beads` | bool | `false` | `install.go:84` |
| `--git` | bool | `false` | `install.go:85` |
| `--github <owner/repo>` | string | `""` | `install.go:86` |
| `--public` | bool | `false` (repos private by default) | `install.go:87` |
| `--shell` | bool | `false` | `install.go:88` |
| `--wrappers` | bool | `false` | `install.go:89` |
| `--supervisor` | bool | `false` | `install.go:90` |
| `--dolt-port <port>` | int | `0` (→ `doltserver.DefaultPort`) | `install.go:91` |

## Related commands

- [uninstall.md](uninstall.md) — inverse; tears down an HQ.
- [upgrade.md](upgrade.md) — updates the gt binary; install does
  not manage the binary itself.
- [doctor.md](doctor.md) — post-install health check. Many of
  the steps above exist specifically to preempt doctor warnings
  (`plugins/`, `daemon.json`, `routing.mode`, `deacon/dogs/boot/`,
  etc. — all have matching comments at their call sites).
- [shell.md](shell.md) — `gt shell install` is called by
  `gt install --shell`.
- [git-init.md](git-init.md) — `gt install --git` uses the
  same `InitGitForHarness` helper. Running `gt install` without
  `--git` then retrofit-ing `gt git-init` is a supported path,
  and it's the first "Next steps" hint.
- [dolt.md](dolt.md) — the Dolt server that install leaves
  running. Install uses the same `doltserver` package internally.
- [config.md](config.md) — for post-install configuration.
- [mayor.md](mayor.md) — final "Next steps" hint is `gt mayor
  attach`.
- [rig.md](rig.md) — second "Next steps" hint is `gt rig add
  <name> <git-url>`; the install command itself does not create
  any rigs.

## Docs claim

### Source
- `internal/cmd/install.go:52-62` — Cobra `Long` text, HQ structure description and `docs/hq.md` reference

### Verbatim
> The HQ (headquarters) is the top-level directory where Gas Town is installed -
> the root of your workspace where all rigs and agents live. It contains:
>   - CLAUDE.md            Mayor role context (Mayor runs from HQ root)
>   - mayor/               Mayor config, state, and rig registry
>   - .beads/              Town-level beads DB (hq-* prefix for mayor mail)
>
> [...]
>
> See docs/hq.md for advanced HQ configurations including beads
> redirects, multi-system setups, and HQ templates.

## Drift

### installCmd.Long HQ structure lists 3 items; install creates at least 5 directories
- **Claim source:** Cobra `Long` text at `internal/cmd/install.go:55-58`
- **Docs claim:** HQ "contains: CLAUDE.md, mayor/, .beads/" (3 items).
- **Code does:** `runInstall` (`install.go:204-499`) also creates `deacon/` (`install.go:307-318`), `plugins/` (`install.go:327-334`), and `mayor/daemon.json` (`install.go:336-342`). The Deacon is a major agent (daemon lifecycle management); `plugins/` is the town-level plugin directory. Both are unconditionally created by install.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — add `deacon/` and `plugins/` to the `installCmd.Long` "It contains:" list
- **Release position:** `in-release` — both directories created at v1.0.0

### installCmd.Long references non-existent docs/hq.md
- **Claim source:** Cobra `Long` text at `internal/cmd/install.go:61-62`
- **Docs claim:** "See docs/hq.md for advanced HQ configurations including beads redirects, multi-system setups, and HQ templates."
- **Code does:** `docs/hq.md` does not exist in the gastown repo at HEAD or at v1.0.0. The Long text references a non-existent document.
- **Category:** `cobra drift`
- **Severity:** `wrong`
- **Fix tier:** `code` — either create `docs/hq.md` or remove the reference from `installCmd.Long`
- **Release position:** `in-release` — reference present at v1.0.0

### docs/INSTALLING.md shows `rigs/` in the gt install tree, but rigs are top-level directories
- **Claim source:** `/home/kimberly/repos/gastown/docs/INSTALLING.md:112-117`
- **Docs claim:** The install tree diagram shows `~/gt/` containing `├── rigs/  # Project containers (initially empty)`. This implies `gt install` creates a `rigs/` subdirectory.
- **Code does:** `runInstall` (`install.go:204-499`) does not create a `rigs/` directory. Rigs are added later via `gt rig add <name> <url>` which creates `~/gt/<rigname>/` as a top-level directory under the town root. The rig registry is `mayor/rigs.json` (a JSON file, not a directory). grep of `install.go` for "rigs/" returns zero hits outside `rigs.json`.
- **Category:** `drift`
- **Severity:** `wrong`
- **Fix tier:** `docs` — remove `rigs/` from the INSTALLING.md tree diagram; replace with a note that rigs are created at the top level via `gt rig add`
- **Release position:** `in-release` — the `rigs/` directory has never existed at any release tag

See [gastown/drift/README.md](../drift/README.md) for the consolidated corrections list.

## Failure modes

### Partial completion

- **Multi-step workspace creation:** `runInstall` at `install.go:94+` creates directories, writes config files, initializes beads, sets up git, creates GitHub repos, installs shell integration, and provisions supervisors — all in sequence. If any middle step fails (e.g., `--github` repo creation fails after directories and config are written), the workspace is left in a partial state. **Absent** — no rollback of previously created directories/files. A re-run with `--force` is the recovery path.

## Troubleshooting

- [Investigating: workspace setup](../workflows/investigations/workspace-setup.md) — Step 2 covers `gt install` failures including dependency checks, port conflicts, and partial scaffolding.

## Notes / open questions

- **Install does a lot of things a healthy fresh system already
  has.** The comments at many steps (`install.go:322-325, 327-330,
  336-342, etc.`) explicitly call out that the step exists to
  preempt a `gt doctor` warning. This means doctor is the de-facto
  acceptance test for install. Worth a dedicated drift / workflow
  page on that relationship.
- **Dolt must be running post-install.** Install intentionally
  leaves the Dolt server running (`install.go:367-369`). A user
  who wants a "quiet" install after the fact has to run `gt dolt
  stop`. See [dolt.md](dolt.md).
- **The `--wrappers`-only escape hatch** (`install.go:125-131`)
  makes `gt install ~/gt --wrappers` idempotent-and-safe even when
  `~/gt` is already a full HQ. No other flag has this
  re-run-on-existing-HQ behavior.
- **Windows support.** Unlike [worktree.md](worktree.md), this
  file has no `runtime.GOOS == "windows"` branches. Most of the
  supervisor / shell / wrappers logic is delegated to helper
  packages that may or may not be Windows-aware. Probably not a
  supported path on Windows yet.
- **`SilenceUsage: true`** is set on the command itself
  (`install.go:76`). None of the other 6 commands in this batch
  do this. It means `gt install --bad-flag` gets the cobra usage
  line, but `gt install /path/that/fails/preflight` does not. If
  error ergonomics ever become a sticking point, worth
  revisiting.
- **`buildBdInitArgs`** (`install.go:561-565`) is a small helper
  that formats the `bd init` invocation with
  `--server-port`. It exists so test code can reconstruct the
  exact argv without duplicating config resolution.
