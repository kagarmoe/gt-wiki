---
title: gt command tree
type: note
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-13
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
| `account`      | `accountCmd`     | `internal/cmd/account.go:520` | [account.md](account.md) |
| `activity`     | `activityCmd`    | `internal/cmd/activity.go:87` | [activity.md](activity.md) |
| `agent-log`    | `agentLogCmd`    | `internal/cmd/agent_log.go:37`| [agent-log.md](agent-log.md) |
| `agents`       | `agentsCmd`      | `internal/cmd/agents.go:147`  | [agents.md](agents.md) |
| `assign`       | `assignCmd`      | `internal/cmd/assign.go:64`   | [assign.md](assign.md) |
| `audit`        | `auditCmd`       | `internal/cmd/audit.go:59`    | [audit.md](audit.md) |
| `bead`         | `beadCmd`        | `internal/cmd/bead.go:94`     | [bead.md](bead.md) |
| `boot`         | `bootCmd`        | `internal/cmd/boot.go:99`     | [boot.md](boot.md) |
| `broadcast`    | `broadcastCmd`   | `internal/cmd/broadcast.go:24`| [broadcast.md](broadcast.md) |
| `callbacks`    | `callbacksCmd`   | `internal/cmd/callbacks.go:117`| [callbacks.md](callbacks.md) |
| `cat`          | `catCmd`         | `internal/cmd/cat.go:33`      | [cat.md](cat.md) |
| `changelog`    | `changelogCmd`   | `internal/cmd/changelog.go:47`| [changelog.md](changelog.md) |
| `checkpoint`   | `checkpointCmd`  | `internal/cmd/checkpoint_cmd.go:80` | [checkpoint.md](checkpoint.md) |
| `cleanup`      | `cleanupCmd`     | `internal/cmd/cleanup.go:42`  | [cleanup.md](cleanup.md) |
| `close`        | `closeCmd`       | `internal/cmd/close.go:44`    | [close.md](close.md) |
| `commit`       | `commitCmd`      | `internal/cmd/commit.go:45`   | [commit.md](commit.md) |
| `compact`      | `compactCmd`     | `internal/cmd/compact.go:83`  | [compact.md](compact.md) |
| `config`       | `configCmd`      | `internal/cmd/config.go:1290` | [config.md](config.md) |
| `convoy`       | `convoyCmd`      | `internal/cmd/convoy.go:426`  | [convoy.md](convoy.md) |
| `costs`        | `costsCmd`       | `internal/cmd/costs.go:111`   | [costs.md](costs.md) |
| `crew`         | `crewCmd`        | `internal/cmd/crew.go:431`    | [crew.md](crew.md) |
| `cycle`        | `cycleCmd`       | `internal/cmd/cycle.go:24`    | [cycle.md](cycle.md) |
| `daemon`       | `daemonCmd`      | `internal/cmd/daemon.go:166`  | [daemon.md](daemon.md) |
| `dashboard`    | `dashboardCmd`   | `internal/cmd/dashboard.go:55`| [dashboard.md](dashboard.md) |
| `deacon`       | `deaconCmd`      | `internal/cmd/deacon.go:467`  | [deacon.md](deacon.md) |
| `directive`    | `directiveCmd`   | `internal/cmd/directive.go:39`| [directive.md](directive.md) |
| `disable`      | `disableCmd`     | `internal/cmd/disable.go:41`  | [disable.md](disable.md) |
| `dnd`          | `dndCmd`         | `internal/cmd/dnd.go:41`      | [dnd.md](dnd.md) |
| `doctor`       | `doctorCmd`      | `internal/cmd/doctor.go:131`  | [doctor.md](doctor.md) |
| `dog`          | `dogCmd`         | `internal/cmd/dog.go:298`     | [dog.md](dog.md) |
| `dolt`         | `doltCmd`        | `internal/cmd/dolt.go:389`    | [dolt.md](dolt.md) |
| `done`         | `doneCmd`        | `internal/cmd/done.go:88`     | [done.md](done.md) |
| `down`         | `downCmd`        | `internal/cmd/down.go:93`     | [down.md](down.md) |
| `enable`       | `enableCmd`      | `internal/cmd/enable.go:34`   | [enable.md](enable.md) |
| `escalate`     | `escalateCmd`    | `internal/cmd/escalate.go:171`| [escalate.md](escalate.md) |
| `estop`        | `estopCmd`       | `internal/cmd/estop.go:71`    | [estop.md](estop.md) |
| `feed`         | `feedCmd`        | `internal/cmd/feed.go:31`     | [feed.md](feed.md) |
| `forget`       | `forgetCmd`      | `internal/cmd/forget.go:13`   | [forget.md](forget.md) |
| `formula`      | `formulaCmd`     | `internal/cmd/formula.go:184` | [formula.md](formula.md) |
| `git-init`     | `gitInitCmd`     | `internal/cmd/gitinit.go:51`  | [git-init.md](git-init.md) |
| `handoff`      | `handoffCmd`     | `internal/cmd/handoff.go:95`  | [handoff.md](handoff.md) |
| `health`       | `healthCmd`      | `internal/cmd/health.go:104`  | [health.md](health.md) |
| `heartbeat`    | `heartbeatCmd`   | `internal/cmd/heartbeat.go:38`| [heartbeat.md](heartbeat.md) |
| `hook`         | `hookCmd`        | `internal/cmd/hook.go:188`    | [hook.md](hook.md) |
| `hooks`        | `hooksCmd`       | `internal/cmd/hooks.go:43`    | [hooks.md](hooks.md) |
| `info`         | `infoCmd`        | `internal/cmd/info.go:482`    | [info.md](info.md) |
| `init`         | `initCmd`        | `internal/cmd/init.go:37`     | [init.md](init.md) |
| `install`      | `installCmd`     | `internal/cmd/install.go:92`  | [install.md](install.md) |
| `issue`        | `issueCmd`       | `internal/cmd/issue.go:53`    | [issue.md](issue.md) |
| `krc`          | `krcCmd`         | `internal/cmd/krc.go:139`     | [krc.md](krc.md) |
| `log`          | `logCmd`         | `internal/cmd/log.go:88`      | [log.md](log.md) |
| `maintain`     | `maintainCmd`    | `internal/cmd/maintain.go:67` | [maintain.md](maintain.md) |
| `mail`         | `mailCmd`        | `internal/cmd/mail.go:543`    | [mail.md](mail.md) |
| `mayor`        | `mayorCmd`       | `internal/cmd/mayor.go:136`   | [mayor.md](mayor.md) |
| `memories`     | `memoriesCmd`    | `internal/cmd/memories.go:17` | [memories.md](memories.md) |
| `metrics`      | `metricsCmd`     | `internal/cmd/metrics.go:26`  | [metrics.md](metrics.md) |
| `molecule`     | `moleculeCmd`    | `internal/cmd/molecule.go:269`| [molecule.md](molecule.md) |
| `mountain`     | `mountainCmd`    | `internal/cmd/mountain.go:105`| [mountain.md](mountain.md) |
| `mq`           | `mqCmd`          | `internal/cmd/mq.go:364`      | [mq.md](mq.md) |
| `namepool`     | `namepoolCmd`    | `internal/cmd/namepool.go:111`| [namepool.md](namepool.md) |
| `notify`       | `notifyCmd`      | `internal/cmd/notify.go:38`   | [notify.md](notify.md) |
| `nudge`        | `nudgeCmd`       | `internal/cmd/nudge.go:62`    | [nudge.md](nudge.md) |
| `nudge-poller` | `nudgePollerCmd` | `internal/cmd/nudge_poller.go:23` | [nudge-poller.md](nudge-poller.md) |
| `orphans`      | `orphansCmd`     | `internal/cmd/orphans.go:181` | [orphans.md](orphans.md) |
| `patrol`       | `patrolCmd`      | `internal/cmd/patrol.go:62`   | [patrol.md](patrol.md) |
| `peek`         | `peekCmd`        | `internal/cmd/peek.go:18`     | [peek.md](peek.md) |
| `plugin`       | `pluginCmd`      | `internal/cmd/plugin.go:165`  | [plugin.md](plugin.md) |
| `polecat`      | `polecatCmd`     | `internal/cmd/polecat.go:378` | [polecat.md](polecat.md) |
| `prime`        | `primeCmd`       | `internal/cmd/prime.go:109`   | [prime.md](prime.md) |
| `proxy-subcmds`| `proxySubcmdsCmd`| `internal/cmd/proxy_subcmds.go:48`| [proxy-subcmds.md](proxy-subcmds.md) |
| `prune-branches`| `pruneBranchesCmd`| `internal/cmd/prune_branches.go:46`| [prune-branches.md](prune-branches.md) |
| `quota`        | `quotaCmd`       | `internal/cmd/quota.go:980`   | [quota.md](quota.md) |
| `ready`        | `readyCmd`       | `internal/cmd/ready.go:49`    | [ready.md](ready.md) |
| `reaper`       | `reaperCmd`      | `internal/cmd/reaper.go:573`  | [reaper.md](reaper.md) |
| `refinery`     | `refineryCmd`    | `internal/cmd/refinery.go:269`| [refinery.md](refinery.md) |
| `release`      | `releaseCmd`     | `internal/cmd/release.go:37`  | [release.md](release.md) |
| `remember`     | `rememberCmd`    | `internal/cmd/remember.go:40` | [remember.md](remember.md) |
| `repair`       | `repairCmd`      | `internal/cmd/repair.go:36`   | [repair.md](repair.md) |
| `resume`       | `resumeCmd`      | `internal/cmd/resume.go:30`   | [resume.md](resume.md) |
| `rig`          | `rigCmd`         | `internal/cmd/rig.go:343`     | [rig.md](rig.md) |
| `role`         | `roleCmd`        | `internal/cmd/role.go:139`    | [role.md](role.md) |
| `scheduler`    | `schedulerCmd`   | `internal/cmd/scheduler.go:117`| [scheduler.md](scheduler.md) |
| `seance`       | `seanceCmd`      | `internal/cmd/seance.go:74`   | [seance.md](seance.md) |
| `session`      | `sessionCmd`     | `internal/cmd/session.go:206` | [session.md](session.md) |
| `shell`        | `shellCmd`       | `internal/cmd/shell.go:64`    | [shell.md](shell.md) |
| `show`         | `showCmd`        | `internal/cmd/show.go:12`     | [show.md](show.md) |
| `shutdown`     | `shutdownCmd`    | `internal/cmd/start.go:164`   | [shutdown.md](shutdown.md) |
| `signal`       | `signalCmd`      | `internal/cmd/signal.go:8`    | [signal.md](signal.md) |
| `sling`        | `slingCmd`       | `internal/cmd/sling.go:168`   | [sling.md](sling.md) |
| `stale`        | `staleCmd`       | `internal/cmd/stale.go:40`    | [stale.md](stale.md) |
| `start`        | `startCmd`       | `internal/cmd/start.go:163`   | [start.md](start.md) |
| `status`       | `statusCmd`      | `internal/cmd/status.go:61`   | [status.md](status.md) |
| `status-line`  | `statusLineCmd`  | `internal/cmd/statusline.go:37`| [status-line.md](status-line.md) |
| `synthesis`    | `synthesisCmd`   | `internal/cmd/synthesis.go:106`| [synthesis.md](synthesis.md) |
| `tap`          | `tapCmd`         | `internal/cmd/tap.go:34`      | [tap.md](tap.md) |
| `thanks`       | `thanksCmd`      | `internal/cmd/thanks.go:238`  | [thanks.md](thanks.md) |
| `thaw`         | `thawCmd`        | `internal/cmd/estop.go:72`    | [thaw.md](thaw.md) |
| `theme`        | `themeCmd`       | `internal/cmd/theme.go:79`    | [theme.md](theme.md) |
| `town`         | `townCmd`        | `internal/cmd/town_cycle.go:33`| [town.md](town.md) |
| `trail`        | `trailCmd`       | `internal/cmd/trail.go:110`   | [trail.md](trail.md) |
| `uninstall`    | `uninstallCmd`   | `internal/cmd/uninstall.go:54`| [uninstall.md](uninstall.md) |
| `unsling`      | `unslingCmd`     | `internal/cmd/unsling.go:52`  | [unsling.md](unsling.md) |
| `up`           | `upCmd`          | `internal/cmd/up.go:158`      | [up.md](up.md) |
| `upgrade`      | `upgradeCmd`     | `internal/cmd/upgrade.go:55`  | [upgrade.md](upgrade.md) |
| `version`      | `versionCmd`     | `internal/cmd/version.go:65`  | [version.md](version.md) |
| `vitals`       | `vitalsCmd`      | `internal/cmd/vitals.go:26`   | [vitals.md](vitals.md) |
| `warrant`      | `warrantCmd`     | `internal/cmd/warrant.go:119` | [warrant.md](warrant.md) |
| `whoami`       | `whoamiCmd`      | `internal/cmd/whoami.go:33`   | [whoami.md](whoami.md) |
| `witness`      | `witnessCmd`     | `internal/cmd/witness.go:142` | [witness.md](witness.md) |
| `wl`           | `wlCmd`          | `internal/cmd/wl.go:70`       | [wl.md](wl.md) |
| `worktree`     | `worktreeCmd`    | `internal/cmd/worktree.go:94` | [worktree.md](worktree.md) |

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
5. **[`memories`](memories.md), [`remember`](remember.md), [`forget`](forget.md)** — three commands that look
   like a memory-management trio. `bd remember` is referenced in the
   auto-injected `AGENTS.md` from `bd init` as the agent memory
   primitive; these `gt` commands may be a parallel or wrapper
   system. Needs verification.
6. **bd-wrapper patterns (resolved in Batch 3c, extended in Batch 3h).** Multiple top-level commands are alternate entry points to bd: [`cat`](cat.md) is a thin `bd show` wrapper with prefix-based rig routing; [`show`](show.md) is also a thin `bd show` wrapper (`DisableFlagParsing`) with auto-routing across hq/gt/mo beads databases via bead-ID prefix — `cat` and `show` are both wrappers around `bd show`; [`close`](close.md) wraps `bd close` with gt-specific `--cascade` child closing and native-SDK convoy propagation; [`done`](done.md) is a polecat-only session-end primitive with extensive side effects (not merely `bd done`); [`ready`](ready.md) is a town-wide aggregated ready view; [`bead`](bead.md) is a parent command for cross-repo bead operations (`move`/`show`/`read`). `gt issue` is NOT a bd-wrapper despite the name — it sets `GT_ISSUE` tmux env var; see [`issue.md`](issue.md).

## What's NOT yet in this index

- **Mapped so far:** **111 of 111** top-level commands have entity pages. Full cobra-group breakdown: `GroupDiag` (22), `GroupConfig` (11), `GroupWork` (26), `GroupAgents` (12), `GroupComm` (7), `GroupServices` (11), `GroupWorkspace` (7) = 96 grouped. The remaining 15 are ungrouped (no `GroupID` set): agent-log, commit, cycle, forget, health, krc, memories, nudge-poller, proxy-subcmds, remember, show, status-line, tap, town, warrant — all mapped in Batch 3h.
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
