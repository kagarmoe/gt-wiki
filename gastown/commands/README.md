---
title: gt command tree
type: note
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/root.go
  - /home/kimberly/repos/gastown/internal/cmd/
tags: [commands, cli, index, inventory]
---

# gt command tree

Authoritative index of every top-level `gt` subcommand registered in
`internal/cmd/`. Drives the per-command entity pages under this
directory.

## Methodology

Command tree extracted by grepping for `rootCmd.AddCommand` under
`/home/kimberly/repos/gastown/internal/cmd/`. Each hit registers one
cobra command as a direct child of the root. Variable names
(`xxxCmd`) map to likely CLI command names but the mapping is
**inferred** until each command's `Use:` field has been verified — see
individual entity pages for the confirmed value. A handful of guesses
are flagged with [?].

Counts as of 2026-04-11:

- **111** `rootCmd.AddCommand` invocations
- **109** files contribute at least one top-level command
- **2** files register two top-level commands each (`estop.go` adds
  `estop` + `thaw`; `start.go` adds `start` + `shutdown`)
- **495** total `cobra.Command{...}` definitions across
  `internal/cmd/` when you include nested subcommands (`rig add`,
  `mayor start`, `convoy launch`, etc.). The 111 top-level commands
  each have on average ~4 nested children.

## Counts

111 top-level `rootCmd.AddCommand` registrations, ~107 unique
top-level commands, 495 total `cobra.Command` definitions. A neutral
inventory — no claim about what any of these commands is for or
whether it's correctly documented. Per-command entity pages will
capture the answer to "what does this command actually do" in
subsequent enumeration batches.

## Top-level commands (alphabetical)

The `source` column points at the file where
`rootCmd.AddCommand(xxxCmd)` lives. Line numbers are accurate as of the
grep pass; they may drift as the code evolves. `entity page` links to
this wiki's per-command page (stub if not yet written).

| command        | var              | source                        | entity page |
|----------------|------------------|-------------------------------|-------------|
| `account`      | `accountCmd`     | `internal/cmd/account.go:520` | — stub      |
| `activity`     | `activityCmd`    | `internal/cmd/activity.go:87` | [activity.md](activity.md) |
| `agent-log` [?]| `agentLogCmd`    | `internal/cmd/agent_log.go:37`| — stub      |
| `agents`       | `agentsCmd`      | `internal/cmd/agents.go:147`  | — stub      |
| `assign`       | `assignCmd`      | `internal/cmd/assign.go:64`   | — stub      |
| `audit`        | `auditCmd`       | `internal/cmd/audit.go:59`    | [audit.md](audit.md) |
| `bead`         | `beadCmd`        | `internal/cmd/bead.go:94`     | — stub      |
| `boot`         | `bootCmd`        | `internal/cmd/boot.go:99`     | — stub      |
| `broadcast`    | `broadcastCmd`   | `internal/cmd/broadcast.go:24`| — stub      |
| `callbacks`    | `callbacksCmd`   | `internal/cmd/callbacks.go:117`| — stub     |
| `cat`          | `catCmd`         | `internal/cmd/cat.go:33`      | — stub      |
| `changelog`    | `changelogCmd`   | `internal/cmd/changelog.go:47`| — stub      |
| `checkpoint`   | `checkpointCmd`  | `internal/cmd/checkpoint_cmd.go:80` | [checkpoint.md](checkpoint.md) |
| `cleanup`      | `cleanupCmd`     | `internal/cmd/cleanup.go:42`  | — stub      |
| `close`        | `closeCmd`       | `internal/cmd/close.go:44`    | — stub      |
| `commit`       | `commitCmd`      | `internal/cmd/commit.go:45`   | — stub      |
| `compact`      | `compactCmd`     | `internal/cmd/compact.go:83`  | — stub      |
| `config`       | `configCmd`      | `internal/cmd/config.go:1290` | — stub      |
| `convoy`       | `convoyCmd`      | `internal/cmd/convoy.go:426`  | — stub      |
| `costs`        | `costsCmd`       | `internal/cmd/costs.go:111`   | [costs.md](costs.md) |
| `crew`         | `crewCmd`        | `internal/cmd/crew.go:431`    | — stub      |
| `cycle`        | `cycleCmd`       | `internal/cmd/cycle.go:24`    | — stub      |
| `daemon`       | `daemonCmd`      | `internal/cmd/daemon.go:166`  | — stub      |
| `dashboard`    | `dashboardCmd`   | `internal/cmd/dashboard.go:55`| [dashboard.md](dashboard.md) |
| `deacon`       | `deaconCmd`      | `internal/cmd/deacon.go:467`  | — stub      |
| `directive`    | `directiveCmd`   | `internal/cmd/directive.go:39`| — stub      |
| `disable`      | `disableCmd`     | `internal/cmd/disable.go:41`  | — stub      |
| `dnd`          | `dndCmd`         | `internal/cmd/dnd.go:41`      | — stub      |
| `doctor`       | `doctorCmd`      | `internal/cmd/doctor.go:131`  | [doctor.md](doctor.md) |
| `dog`          | `dogCmd`         | `internal/cmd/dog.go:298`     | — stub      |
| `dolt`         | `doltCmd`        | `internal/cmd/dolt.go:389`    | — stub      |
| `done`         | `doneCmd`        | `internal/cmd/done.go:88`     | — stub      |
| `down`         | `downCmd`        | `internal/cmd/down.go:93`     | — stub      |
| `enable`       | `enableCmd`      | `internal/cmd/enable.go:34`   | — stub      |
| `escalate`     | `escalateCmd`    | `internal/cmd/escalate.go:171`| — stub      |
| `estop`        | `estopCmd`       | `internal/cmd/estop.go:71`    | — stub      |
| `feed`         | `feedCmd`        | `internal/cmd/feed.go:31`     | [feed.md](feed.md) |
| `forget`       | `forgetCmd`      | `internal/cmd/forget.go:13`   | — stub      |
| `formula`      | `formulaCmd`     | `internal/cmd/formula.go:184` | — stub      |
| `git-init` [?] | `gitInitCmd`     | `internal/cmd/gitinit.go:51`  | — stub      |
| `handoff`      | `handoffCmd`     | `internal/cmd/handoff.go:95`  | — stub      |
| `health`       | `healthCmd`      | `internal/cmd/health.go:104`  | — stub      |
| `heartbeat`    | `heartbeatCmd`   | `internal/cmd/heartbeat.go:38`| [heartbeat.md](heartbeat.md) |
| `hook`         | `hookCmd`        | `internal/cmd/hook.go:188`    | — stub      |
| `hooks`        | `hooksCmd`       | `internal/cmd/hooks.go:43`    | — stub      |
| `info`         | `infoCmd`        | `internal/cmd/info.go:482`    | [info.md](info.md) |
| `init`         | `initCmd`        | `internal/cmd/init.go:37`     | — stub      |
| `install`      | `installCmd`     | `internal/cmd/install.go:92`  | — stub      |
| `issue`        | `issueCmd`       | `internal/cmd/issue.go:53`    | — stub      |
| `krc`          | `krcCmd`         | `internal/cmd/krc.go:139`     | — stub      |
| `log`          | `logCmd`         | `internal/cmd/log.go:88`      | [log.md](log.md) |
| `maintain`     | `maintainCmd`    | `internal/cmd/maintain.go:67` | — stub      |
| `mail`         | `mailCmd`        | `internal/cmd/mail.go:543`    | — stub      |
| `mayor`        | `mayorCmd`       | `internal/cmd/mayor.go:136`   | — stub      |
| `memories`     | `memoriesCmd`    | `internal/cmd/memories.go:17` | — stub      |
| `metrics`      | `metricsCmd`     | `internal/cmd/metrics.go:26`  | [metrics.md](metrics.md) |
| `molecule`     | `moleculeCmd`    | `internal/cmd/molecule.go:269`| — stub      |
| `mountain`     | `mountainCmd`    | `internal/cmd/mountain.go:105`| — stub      |
| `mq`           | `mqCmd`          | `internal/cmd/mq.go:364`      | — stub      |
| `namepool`     | `namepoolCmd`    | `internal/cmd/namepool.go:111`| — stub      |
| `notify`       | `notifyCmd`      | `internal/cmd/notify.go:38`   | — stub      |
| `nudge`        | `nudgeCmd`       | `internal/cmd/nudge.go:62`    | — stub      |
| `nudge-poller` [?]| `nudgePollerCmd` | `internal/cmd/nudge_poller.go:23` | — stub |
| `orphans`      | `orphansCmd`     | `internal/cmd/orphans.go:181` | — stub      |
| `patrol`       | `patrolCmd`      | `internal/cmd/patrol.go:62`   | [patrol.md](patrol.md) |
| `peek`         | `peekCmd`        | `internal/cmd/peek.go:18`     | — stub      |
| `plugin`       | `pluginCmd`      | `internal/cmd/plugin.go:165`  | — stub      |
| `polecat`      | `polecatCmd`     | `internal/cmd/polecat.go:378` | — stub      |
| `prime`        | `primeCmd`       | `internal/cmd/prime.go:109`   | [prime.md](prime.md) |
| `proxy` [?]    | `proxySubcmdsCmd`| `internal/cmd/proxy_subcmds.go:48`| — stub  |
| `prune-branches` [?]| `pruneBranchesCmd`| `internal/cmd/prune_branches.go:46`| — stub |
| `quota`        | `quotaCmd`       | `internal/cmd/quota.go:980`   | — stub      |
| `ready`        | `readyCmd`       | `internal/cmd/ready.go:49`    | — stub      |
| `reaper`       | `reaperCmd`      | `internal/cmd/reaper.go:573`  | — stub      |
| `refinery`     | `refineryCmd`    | `internal/cmd/refinery.go:269`| — stub      |
| `release`      | `releaseCmd`     | `internal/cmd/release.go:37`  | — stub      |
| `remember`     | `rememberCmd`    | `internal/cmd/remember.go:40` | — stub      |
| `repair`       | `repairCmd`      | `internal/cmd/repair.go:36`   | [repair.md](repair.md) |
| `resume`       | `resumeCmd`      | `internal/cmd/resume.go:30`   | — stub      |
| `rig`          | `rigCmd`         | `internal/cmd/rig.go:343`     | — stub      |
| `role`         | `roleCmd`        | `internal/cmd/role.go:139`    | — stub      |
| `scheduler`    | `schedulerCmd`   | `internal/cmd/scheduler.go:117`| — stub     |
| `seance`       | `seanceCmd`      | `internal/cmd/seance.go:74`   | [seance.md](seance.md) |
| `session`      | `sessionCmd`     | `internal/cmd/session.go:206` | — stub      |
| `shell`        | `shellCmd`       | `internal/cmd/shell.go:64`    | — stub      |
| `show`         | `showCmd`        | `internal/cmd/show.go:12`     | — stub      |
| `shutdown`     | `shutdownCmd`    | `internal/cmd/start.go:164`   | — stub      |
| `signal`       | `signalCmd`      | `internal/cmd/signal.go:8`    | — stub      |
| `sling`        | `slingCmd`       | `internal/cmd/sling.go:168`   | — stub      |
| `stale`        | `staleCmd`       | `internal/cmd/stale.go:40`    | [stale.md](stale.md) |
| `start`        | `startCmd`       | `internal/cmd/start.go:163`   | — stub      |
| `status`       | `statusCmd`      | `internal/cmd/status.go:61`   | [status.md](status.md) |
| `statusline`   | `statusLineCmd`  | `internal/cmd/statusline.go:37`| — stub     |
| `synthesis`    | `synthesisCmd`   | `internal/cmd/synthesis.go:106`| — stub     |
| `tap`          | `tapCmd`         | `internal/cmd/tap.go:34`      | — stub      |
| `thanks`       | `thanksCmd`      | `internal/cmd/thanks.go:238`  | [thanks.md](thanks.md) |
| `thaw`         | `thawCmd`        | `internal/cmd/estop.go:72`    | — stub      |
| `theme`        | `themeCmd`       | `internal/cmd/theme.go:79`    | — stub      |
| `town`         | `townCmd`        | `internal/cmd/town_cycle.go:33`| — stub     |
| `trail`        | `trailCmd`       | `internal/cmd/trail.go:110`   | — stub      |
| `uninstall`    | `uninstallCmd`   | `internal/cmd/uninstall.go:54`| — stub      |
| `unsling`      | `unslingCmd`     | `internal/cmd/unsling.go:52`  | — stub      |
| `up`           | `upCmd`          | `internal/cmd/up.go:158`      | — stub      |
| `upgrade`      | `upgradeCmd`     | `internal/cmd/upgrade.go:55`  | [upgrade.md](upgrade.md) |
| `version`      | `versionCmd`     | `internal/cmd/version.go:65`  | [version.md](version.md) |
| `vitals`       | `vitalsCmd`      | `internal/cmd/vitals.go:26`   | [vitals.md](vitals.md) |
| `warrant`      | `warrantCmd`     | `internal/cmd/warrant.go:119` | — stub      |
| `whoami`       | `whoamiCmd`      | `internal/cmd/whoami.go:33`   | [whoami.md](whoami.md) |
| `witness`      | `witnessCmd`     | `internal/cmd/witness.go:142` | — stub      |
| `wl`           | `wlCmd`          | `internal/cmd/wl.go:70`       | — stub      |
| `worktree`     | `worktreeCmd`    | `internal/cmd/worktree.go:94` | — stub      |

**[?] = inferred command name; `Use:` field not yet verified.**

## Command groups

See [../binaries/gt.md](../binaries/gt.md) for the 7 group IDs defined
on the root command. Each top-level command assigns itself to a
`GroupID` in its own file (e.g., `version.go:33` uses `GroupDiag`).
Mapping each command to its group is a follow-up pass — a dedicated
bead will track this.

## Observations

1. **Two commands sharing a root letter.** `hook` and `hooks` are both
   top-level commands, each in its own file. Likely one is
   singular/single-item management and the other bulk, or one is
   deprecated. Needs disambiguation.
2. **Compound command names guessed from var names.** Several `xxxCmd`
   variables use CamelCase that could map to either hyphenated,
   space-separated, or concatenated command names
   (`agentLogCmd` → `agent-log` vs `agent log` vs `agentlog`). All
   flagged `[?]` until a per-command verification pass reads the
   `Use:` field.
3. **`statusCmd` vs `statusLineCmd`.** Two separate top-level commands:
   `status` and `statusline`. The names are too close for accident —
   likely one gives overall status, the other formats a shell
   status-line integration (similar to `gt statusline` in Claude Code
   integrations).
4. **Named after Gas Town lore.** `deacon`, `mountain`, `thanks`,
   `peek`, `warrant`, `witness`, `refinery`, `reaper`, `seance`,
   `convoy`, `patrol`, `mayor`, `rig`, `crew`, `polecat`, `formula` —
   none self-explanatory from the name. A follow-up pass will
   reverse-map the domain vocabulary.
5. **`memories`, `remember`, `forget`** — three commands that look
   like a memory-management trio. `bd remember` is referenced in the
   auto-injected `AGENTS.md` from `bd init` as the agent memory
   primitive; these `gt` commands may be a parallel or wrapper
   system. Needs verification.
6. **`bead`, `issue`, `ready`, `close`, `done`** — five commands that
   look like alternate-syntax wrappers around `bd` operations. If so,
   `gt` may be re-exposing `bd` under its own noun (`gt issue ready`
   ≈ `bd ready`). Tracked as a follow-up.

## What's NOT yet in this index

- **Mapped so far:** 22 of 111 top-level commands have entity pages — the full `GroupDiag` set (version, activity, audit, checkpoint, costs, dashboard, doctor, feed, heartbeat, info, log, metrics, patrol, prime, repair, seance, stale, status, thanks, upgrade, vitals, whoami). Remaining groups pending: Work, Agents, Comm, Services, Workspace, Config.
- **`Use:` field verification per command.** Many entries have inferred
  command names flagged `[?]`.
- **`GroupID` assignment per command.** Each command needs a quick read
  to extract its `GroupID` constant.
- **Nested subcommand tree.** Each top-level has on average ~4 nested
  subcommands (e.g., `rig add`, `rig list`, `mayor start`). The 495
  total cobra.Command count minus 111 top-level ≈ 384 nested commands.
  A full subcommand inventory is a separate pass.
- **Flag enumeration per command.** This index does not list flags.
- **Polecat-safe annotations.** `internal/cmd/proxy_subcmds.go:15`
  defines `AnnotationPolecatSafe = "polecatSafe"` and `version.go:34`
  uses it. Which commands carry the annotation?

## See also

- [../binaries/gt.md](../binaries/gt.md) — the binary, entry point,
  ldflags, runtime state, persistentPreRun sequence, command groups,
  and the exempt-command maps.
- [../files/makefile.md](../files/makefile.md) — build recipes and
  binary outputs.
