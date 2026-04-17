---
title: gastown auxiliary trees inventory
type: note
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/scripts/
  - /home/kimberly/repos/gastown/templates/
  - /home/kimberly/repos/gastown/gt-model-eval/
  - /home/kimberly/repos/gastown/npm-package/
  - /home/kimberly/repos/gastown/.github/
  - /home/kimberly/repos/gastown/.githooks/
  - /home/kimberly/repos/gastown/.claude/
  - /home/kimberly/repos/gastown/.opencode/
  - /home/kimberly/repos/gastown/.runtime/
tags: [inventory, enumeration, auxiliary, layer-k]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gastown auxiliary trees inventory

A-level enumeration of the nine top-level auxiliary directories not covered
by earlier batches (binaries/commands/packages/files/plugins). Every file and
subdirectory listed, with one-line descriptions grounded in the source.

Batch 11 / Layer (k) of the gastown-map plan.

## scripts/ — build, bootstrap, CI, test helpers

Top-level: 12 files, 3 sub-directories. Shell + one Python script + Markdown
documentation.

| item | bytes | notes |
|---|---:|---|
| `bootstrap-local-rig.sh` | 4289 | `gt rig add` wrapper for creating a Gas Town rig from a local source repo. Required args: `--town-root`, `--rig`, `--local-repo`. Optional `--remote`, `--gt-bin`, `--prefix`, role_agents overrides. |
| `bump-version.sh` | 10849 | Version bump script (Makefile / VERSION / releases coordination). |
| `ci_state_classifier.py` | 5730 | Python. Classifies CI run state (used by workflows or reports). |
| `generate-newsletter.py` | 28426 | Python. Largest script. Generates newsletter content (agent-facing). |
| `gen_hanoi.py` | 3302 | Python. Likely generates Towers-of-Hanoi test data — used as synthetic work in polecat/witness tests. |
| `launch-migration-at.sh` | 4014 | Schedules a migration run at a specific time. |
| `run-boot-scraper-vm.sh` | 5782 | Runs a boot scraper inside a VM (boot-time data capture). |
| `run-hardener.sh` | 2288 | Runs the system hardener (security posture). |
| `test-gce-install.sh` | 6647 | Tests `gt` install on GCE. |
| `test-proxy-manual.sh` | 11510 | Manual proxy testing script. See also `PROXY_TESTING.md`. |
| `test-proxy-smoke.sh` | 5346 | Smoke test for `gt-proxy-server` / `gt-proxy-client`. |
| `update-nix-flake.sh` | 2182 | Updates `flake.lock` / nix inputs. |
| `PROXY_TESTING.md` | 4561 | Documentation for manual proxy testing workflows. |

Sub-directories:

| dir | contents | notes |
|---|---|---|
| `scripts/guards/` | `context-budget-guard.sh` (7995 B), `context-budget-guard_test.sh` (7274 B) | Standalone Claude Code hook guard. Reads the active session transcript, tracks token usage, enforces warn/soft/hard thresholds (0.75/0.85/0.92). Env vars: `GT_CONTEXT_BUDGET_*`. Hard-gate roles default: `mayor,deacon,witness,refinery`. Fails open on any error. Hook point: `UserPromptSubmit`. Dependency: `jq`. |
| `scripts/lib/` | `common.sh` (1203 B) | Sourced library. Colour vars (RED/GREEN/YELLOW/NC), `is_darwin_sed`, cross-platform `sed_i`, `update_file`. |
| `scripts/migration-test/` | 7 shell scripts | End-to-end migration/VM test harness: `reset-vm.sh`, `seed-data.sh`, `setup-backup.sh`, `run-test.sh`, `validate-migration.sh` (9584 B), `test-edge-cases.sh` (13187 B), `vm-integration-test.sh` (21724 B — the largest). |

**Cross-refs:** context-budget-guard binds into `gt hook` and the Claude
Code hook settings managed by [../packages/hooks.md](../packages/hooks.md).
bootstrap-local-rig invokes [../commands/rig.md](../commands/rig.md).

## templates/ — per-role agent context + agent runtime templates

Top-level:

| item | bytes | notes |
|---|---:|---|
| `polecat-CLAUDE.md` | 9248 | Template for `{{rig}}/polecats/{{name}}/CLAUDE.md`. Starts with "THE IDLE POLECAT HERESY" (mandatory `gt done`) and SINGLE-TASK FOCUS rules. `{{rig}}` and `{{name}}` placeholders expanded at polecat spawn time. |
| `witness-CLAUDE.md` | 6278 | Template for `{{RIG}}/witness/CLAUDE.md`. Defines the witness role (pit boss), nudge/escalate protocol, dormant polecat recovery (`gt polecat check-recovery`), deacon-session naming (`hq-deacon`). |
| `agents/` | 2 files | Agent-runtime templates — see [../files/templates-agents.md](../files/templates-agents.md) for the entity page. |

**Cross-refs:** these templates are loaded by
[../packages/templates.md](../packages/templates.md) and surfaced by
[../commands/prime.md](../commands/prime.md).

## gt-model-eval/ — promptfoo-based patrol-agent model comparison

10 items. Node.js project. Measures whether cheaper Claude models (Sonnet,
Haiku) match Opus on patrol decisions — evidence base for downgrading
roles to reduce Opus burn.

| item | bytes | notes |
|---|---:|---|
| `README.md` | 5778 | Full usage guide. Class A (neutral, reasoning-from-evidence) vs Class B (directive, instruction-following). 94 tests total (82 Class B + 12 Class A) across 4 patrol roles (deacon, witness, refinery, dog). |
| `MAINTENANCE.md` | 5395 | Maintenance notes. |
| `package.json` | 419 | Pins `promptfoo@0.121.2`. Scripts: `eval`, `eval:repeat`, `view`, `export`. |
| `package-lock.json` | 508527 | Lockfile. |
| `promptfooconfig.yaml` | 2755 | Three providers: `claude-opus-4-6`, `claude-sonnet-4-5-20250929`, `claude-haiku-4-5-20251001` at temperature 0. Default assertions: `is-json`, action field present, reason field present, action in allowed set, cost ≤ $0.10, latency ≤ 15s. Grader: Opus 4.6. maxConcurrency: 3. |
| `.env.example` | 89 | `ANTHROPIC_API_KEY=...`. |
| `.gitignore` | 115 | Local artifacts. |
| `prompts/patrol-decision.txt` | 1728 | System prompt template with `{{role}}`, `{{role_context}}`, `{{formula_step}}`, `{{allowed_actions}}`, `{{shell_output}}`, `{{context}}` placeholders. Asserts ZFC (Zero Fixed Constants): the model decides thresholds. Required JSON fields: `action`, `reason`. |
| `scripts/results-to-discussion.sh` | 9020 | Format results as markdown and (optionally) post to a GitHub Discussion via `gh`. `--post`, `--repo` flags. |
| `tests/` (12 YAML files) | 123154 total | See below. |

`tests/` contents — 12 yaml files, 4 Class A + 8 Class B:

| file | bytes | role | class |
|---|---:|---|---|
| `class-a-deacon.yaml` | 4436 | deacon | A (3 tests) |
| `class-a-dog.yaml` | 3722 | dog | A (3 tests) |
| `class-a-refinery.yaml` | 4366 | refinery | A (3 tests) |
| `class-a-witness.yaml` | 3870 | witness | A (3 tests) |
| `deacon-dog-health.yaml` | 10073 | deacon | B (10 tests) |
| `deacon-plugin-gate.yaml` | 9995 | deacon | B (10 tests) |
| `deacon-zombie.yaml` | 14603 | deacon | B (10 tests) |
| `dog-orphan.yaml` | 10857 | dog | B (10 tests) |
| `refinery-conflict.yaml` | 19073 | refinery | B (10 tests) |
| `refinery-triage.yaml` | 16251 | refinery | B (10 tests) |
| `witness-cleanup.yaml` | 11171 | witness | B (10 tests) |
| `witness-stuck.yaml` | 14658 | witness | B (12 tests) |

Linked issues: [Discussion #1542], [Issue #1545] (downgrade evidence tracking).

## npm-package/ — @gastown/gt npm installer

6 items. Installer wrapper for the Go-built `gt` binary so it can be installed
via `npm install -g @gastown/gt`.

| item | bytes | notes |
|---|---:|---|
| `package.json` | 980 | `@gastown/gt` v1.0.0. `bin.gt` → `bin/gt.js`. Scripts: `postinstall` (downloads platform binary), `test`. Node ≥14. Platforms: darwin/linux/win32 × x64/arm64. |
| `README.md` | 630 | Installation/usage doc (5 commands: `version`, `init`, `status`, `rig list`, manual install URL). |
| `LICENSE` | 1068 | MIT (per README). |
| `.npmignore` | 169 | Publish exclusions. |
| `bin/gt.js` | 1345 | Shim. Resolves platform-specific binary name (`gt` or `gt.exe`), spawns it with `stdio: 'inherit'`, forwards `process.argv.slice(2)`, propagates exit code. |
| `scripts/postinstall.js` | 6217 | Downloads `gastown_<version>_<platform>_<arch>.(tar.gz|zip)` from the GitHub release at `https://github.com/steveyegge/gastown/releases/download/v<version>/<archive>`. Follows 301/302. Extracts via `tar -xzf` (Unix) or `Expand-Archive` (Windows PowerShell) or `unzip`. Chmod 0755. Verifies with `gt version`. **Skips download if `process.env.CI` is set.** |
| `scripts/test.js` | 1502 | Post-install smoke test. |

**Cross-refs:** ships the binaries built by
[../files/makefile.md](../files/makefile.md) (`gt`, `gt-proxy-server`,
`gt-proxy-client`). Release artefacts produced by `.github/workflows/release.yml`.

## .github/ — GitHub repo metadata (CI, issue/PR templates, scripts)

Sub-directories and top-level:

| item | notes |
|---|---|
| `PULL_REQUEST_TEMPLATE.md` (431 B) | Sections: Summary, Related Issue, Changes, Testing (unit tests + manual), Checklist (style, docs, breaking changes). |
| `ISSUE_TEMPLATE/bug_report.md` (570 B) | Standard bug report form. |
| `ISSUE_TEMPLATE/feature_request.md` (467 B) | Standard feature request form. |
| `scripts/junit-report.py` (4802 B) | Python. Generates JUnit-formatted XML test reports for workflows. |

`workflows/` — 10 YAML files:

| workflow | bytes | purpose |
|---|---:|---|
| `ci.yml` | 6468 | Main CI. Pin-hash actions. Includes guards: rejects `go.mod` replace directives (issue #2230), rejects `issues.jsonl` (Dolt is the only backend now). |
| `release.yml` | 2500 | Triggered on `v*` tags. Runs GoReleaser via `.goreleaser.yml`. `permissions: contents: write`. Concurrency-grouped per ref. |
| `e2e.yml` | 481 | End-to-end test job. |
| `nightly-integration.yml` | 1921 | Nightly integration suite. |
| `windows-ci.yml` | 1147 | Windows-specific CI. |
| `block-internal-prs.yml` | 1722 | Blocks PRs from same-repo branches (Gas Town pushes to main directly; PRs are contributor-only). |
| `close-stale-needs.yml` | 3638 | Closes stale `needs-*` issues. |
| `remove-needs-info.yml` | 1952 | Auto-removes `needs-info` label on comment activity. |
| `remove-needs-triage.yml` | 1427 | Auto-removes `needs-triage` on labelling. |
| `triage-label.yml` | 713 | Auto-labels new issues for triage. |

## .githooks/ — repo-local pre-push guardrails

| item | bytes | notes |
|---|---:|---|
| `pre-push` | 4239 | Bash. Blocks pushes to invalid branches — Gas Town agents push to `main` (crew) or `polecat/*` / `integration/*` branches. Also enforces integration-branch-landing guardrail (blocks pushes to default branch that introduce integration branch content unless `GT_INTEGRATION_LAND=1` is set, which only `gt mq integration land` sets). Upstream-remote escape hatch for external contributors. Default-branch detection walks `origin/HEAD` → `origin/master` → `origin/main`. |
| `pre-push_test.sh` | 8046 | Bash test suite for the pre-push hook. Creates temp git repos to simulate push scenarios. |

## .claude/ — Claude Code agent-facing surface (slash commands + skills)

Full entity page: [../files/claude-dir.md](../files/claude-dir.md).

| item | notes |
|---|---|
| `commands/backup.md` (3250 B) | Slash command: `/backup`. Runs the dolt-snapshots-like database backup cycle. Mirrors the `dolt-backup` plugin. |
| `commands/patrol.md` (4804 B) | Slash command: `/patrol [witness|deacon|refinery]`. Runs one patrol cycle. Detects role from `$GT_ROLE` if no arg. |
| `commands/reaper.md` (3281 B) | Slash command: `/reaper [--dry-run]`. Runs the wisp reaper (scan/reap/purge/auto-close stale issues) directly without Dog dispatch. |
| `skills/crew-commit/SKILL.md` | Canonical 8-step commit workflow for crew (pre-flight → branch → submodule check → stage → `gt commit` → push → PR → notify). |
| `skills/ghi-list/SKILL.md` | Formats `gh issue list` output as an ASCII box-drawing table. |
| `skills/pr-list/SKILL.md` | Formats `gh pr list` output as an ASCII box-drawing table. Filters `CHANGES_REQUESTED` by default. |
| `skills/pr-sheriff/skill.md` | PR triage workflow. Delegates to the `mol-pr-sheriff-patrol` formula. Categories: EASY-WIN, NEEDS-CREW, NEEDS-HUMAN, SKIP. Uses ephemeral beads (wisps) for dispatched fix-merge work. |

## .opencode/ — opencode.ai agent-facing surface

Full entity page: [../files/opencode-dir.md](../files/opencode-dir.md).

| item | bytes | notes |
|---|---:|---|
| `commands/handoff.md` | 581 | Slash command: `/handoff [message]`. Runs `gt handoff` (optionally with `-s` + `-m`). Note: new session auto-primes via the SessionStart hook. |
| `plugins/gastown.js` | 2720 | OpenCode plugin. Hooks `session.created` / `session.compacted` / `session.deleted` events. On session create/compact: runs `gt prime` (and `gt mail check --inject` for autonomous roles: polecat/witness/refinery/deacon). Injects captured output into `experimental.chat.system.transform`. On `session.deleted`: records cost via `gt costs record --session <id>`. |

## .runtime/ — per-rig runtime setup hooks

| item | bytes | notes |
|---|---:|---|
| `setup-hooks/01-git-config.sh` | 430 | Bash. Propagates global git identity (`user.name` / `user.email`) into polecat worktrees via `git -C "$GT_WORKTREE_PATH" config ...`. Prevents commits from falling back to system defaults and being misattributed. |

**Cross-refs:** runs inside [../packages/runtime.md](../packages/runtime.md)
flow when a polecat worktree is created. Consumes `$GT_WORKTREE_PATH`
(set by runtime spawn logic).

## Cross-topic references

- Polecat/witness role context templates → [../roles/polecat.md](../roles/polecat.md), [../roles/witness.md](../roles/witness.md)
- Agent runtime integration for opencode hook provider → [../packages/runtime.md](../packages/runtime.md)
- Role templates + prime-time injection → [../packages/templates.md](../packages/templates.md), [../commands/prime.md](../commands/prime.md)
- Hook settings installer (consumed by context-budget-guard.sh) → [../packages/hooks.md](../packages/hooks.md)
- Main CLI → [../binaries/gt.md](../binaries/gt.md)

## Totals

- **9 directories**, enumerated in full.
- **~50 top-level files + 30+ files in sub-directories** across scripts/
  (16 items + subdirs with 10 additional files), templates/ (3), gt-model-eval/
  (10 + 12 tests), npm-package/ (6), .github/ (14 workflow/template files +
  scripts/), .githooks/ (2), .claude/ (3 commands + 4 skills), .opencode/
  (1 command + 1 plugin), .runtime/ (1 hook script).
- **3 directories get dedicated C-level entity pages** (.claude/, .opencode/,
  templates/agents/) because they define agent-facing behaviour the rest of
  the system depends on.
