---
title: gastown Go packages inventory
type: note
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/cmd/
  - /home/kimberly/repos/gastown/internal/
  - /home/kimberly/repos/gastown/plugins/
tags: [inventory, enumeration, go-packages]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gastown Go packages inventory

Every Go package directory under `cmd/`, `internal/`, and `plugins/`
as of 2026-04-11 with file counts. Neutral enumeration.

**Capture method:** `find <dir> -maxdepth 1 -name '*.go'` split into
`*_test.go` and non-test files, with `find <dir> -mindepth 1 -maxdepth 1
-type d` for nested package count.

**Totals:**

- **3 `cmd/` binaries** (one Go module main each)
- **67 `internal/` packages** with Go files, counted at immediate
  subdirectory depth only. Nested packages exist but are not listed
  here.
- **14 `plugins/` subdirectories**, only 1 of which currently contains
  a Go source file.
- **~1088 total `.go` files** in the module (including tests), from
  `find . -name "*.go" -not -path "./vendor/*" -not -path "./.git/*"
  | wc -l` run at repo root.

## cmd/

| package              | go files | test files |
|----------------------|---------:|-----------:|
| `gt`                 |        1 |          1 |
| `gt-proxy-client`    |        1 |          0 |
| `gt-proxy-server`    |        2 |          1 |

See also: [../binaries/gt.md](../binaries/gt.md),
[../files/makefile.md](../files/makefile.md).

## internal/ (67 packages)

| package         | go files | test files | nested subdirs |
|-----------------|---------:|-----------:|---------------:|
| `acp`           |        4 |          8 |              0 |
| `activity`      |        1 |          1 |              0 |
| `agent`         |        1 |          1 |              1 |
| `agentlog`      |        3 |          1 |              0 |
| `beads`         |       28 |         21 |              0 |
| `boot`          |        1 |          1 |              0 |
| `channelevents` |        1 |          1 |              0 |
| `checkpoint`    |        2 |          2 |              0 |
| `cli`           |        1 |          1 |              0 |
| `cmd`           |      239 |        164 |              0 |
| `config`        |        9 |         12 |              1 |
| `connection`    |        4 |          1 |              0 |
| `constants`     |        1 |          1 |              0 |
| `convoy`        |        2 |          4 |              0 |
| `crew`          |        2 |          2 |              0 |
| `daemon`        |       33 |         29 |              0 |
| `deacon`        |        7 |          7 |              0 |
| `deps`          |        3 |          2 |              0 |
| `doctor`        |       70 |         50 |              0 |
| `dog`           |        4 |          5 |              0 |
| `doltserver`    |        9 |         12 |              0 |
| `estop`         |        1 |          1 |              0 |
| `events`        |        1 |          1 |              0 |
| `feed`          |        1 |          1 |              0 |
| `formula`       |        6 |          9 |              1 |
| `git`           |        3 |          3 |              0 |
| `github`        |        2 |          1 |              0 |
| `health`        |        1 |          0 |              0 |
| `hooks`         |        3 |          4 |              1 |
| `hookutil`      |        1 |          1 |              0 |
| `keepalive`     |        1 |          1 |              0 |
| `krc`           |        3 |          3 |              0 |
| `lock`          |        5 |          1 |              0 |
| `mail`          |        7 |          7 |              0 |
| `mayor`         |        4 |          2 |              0 |
| `mq`            |        1 |          1 |              0 |
| `nudge`         |        6 |          2 |              0 |
| `plugin`        |        4 |          3 |              0 |
| `polecat`       |        5 |          7 |              0 |
| `protocol`      |        5 |          1 |              0 |
| `proxy`         |        5 |          6 |              0 |
| `quota`         |        6 |          5 |              0 |
| `reaper`        |        1 |          1 |              0 |
| `refinery`      |        5 |         10 |              0 |
| `rig`           |        5 |          4 |              0 |
| `runtime`       |        1 |          1 |              0 |
| `scheduler`     |        0 |          0 |              1 |
| `session`       |       11 |         10 |              0 |
| `shell`         |        1 |          1 |              0 |
| `state`         |        1 |          1 |              0 |
| `style`         |        2 |          1 |              0 |
| `suggest`       |        1 |          1 |              0 |
| `telemetry`     |        3 |          3 |              0 |
| `templates`     |        2 |          1 |              6 |
| `testutil`      |        4 |          2 |              0 |
| `tmux`          |       11 |         10 |              0 |
| `townlog`       |        1 |          1 |              0 |
| `tui`           |        0 |          0 |              2 |
| `ui`            |        4 |          2 |              0 |
| `util`          |        9 |          6 |              0 |
| `version`       |        1 |          1 |              0 |
| `wasteland`     |        3 |          6 |              0 |
| `web`           |        7 |          7 |              2 |
| `wisp`          |        4 |          4 |              0 |
| `witness`       |        7 |          7 |              0 |
| `workspace`     |        1 |          1 |              0 |
| `wrappers`      |        1 |          1 |              1 |

**Notes about the counts above:**

- `internal/cmd` is by far the largest package at **239 Go files +
  164 tests = 403 files**. This is the package that holds all cobra
  command definitions. It is mapped separately in
  [../commands/README.md](../commands/README.md) and
  [../binaries/gt.md](../binaries/gt.md).
- `internal/doctor` (70 go + 50 test = 120 files) is the
  second-largest.
- `internal/daemon` (33 go + 29 test = 62) is third.
- `internal/beads` (28 go + 21 test = 49) is fourth.
- Several packages have 0 go files at the immediate level but have
  nested subdirectories (`scheduler`, `tui`). Their content lives at a
  nested level — enumerating nested packages is a follow-up pass.
- `internal/templates` has 6 nested subdirectories (not enumerated
  here).

## plugins/ (14 directories)

| package                 | go files | test files |
|-------------------------|---------:|-----------:|
| `compactor-dog`         |        0 |          0 |
| `dolt-archive`          |        0 |          0 |
| `dolt-backup`           |        0 |          0 |
| `dolt-log-rotate`       |        0 |          0 |
| `dolt-snapshots`        |        1 |          1 |
| `github-sheriff`        |        0 |          0 |
| `git-hygiene`           |        0 |          0 |
| `gitignore-reconcile`   |        0 |          0 |
| `quality-review`        |        0 |          0 |
| `rate-limit-watchdog`   |        0 |          0 |
| `rebuild-gt`            |        0 |          0 |
| `stuck-agent-dog`       |        0 |          0 |
| `submodule-commit`      |        0 |          0 |
| `tool-updater`          |        0 |          0 |

Only `dolt-snapshots` contains a Go source file. The other 13 are
either stubs, non-Go content (shell scripts, config files), or empty
at the immediate-child level. Investigating whether these are
placeholders or use a different file layout is a follow-up pass.
