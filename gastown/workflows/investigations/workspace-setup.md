---
title: "Investigating: workspace setup failures"
type: investigation
status: verified
topic: gastown
created: 2026-04-17
updated: 2026-04-17
sources:
  - gastown/commands/install.md
  - gastown/commands/rig.md
  - gastown/commands/crew.md
  - gastown/packages/deps.md
  - gastown/packages/workspace.md
  - /home/kimberly/repos/gastown/internal/cmd/install.go
  - /home/kimberly/repos/gastown/internal/cmd/rig.go
  - /home/kimberly/repos/gastown/internal/cmd/crew_add.go
  - /home/kimberly/repos/gastown/internal/deps/beads.go
  - /home/kimberly/repos/gastown/internal/deps/dolt.go
  - /home/kimberly/repos/gastown/internal/workspace/find.go
tags: [investigation, workspace-setup, diagnostic, install, rig, crew]
---

# Investigating: workspace setup failures

**Symptom:** `gt install` fails or produces a broken workspace.
`gt rig add` fails to clone or register a rig. `gt crew add` creates
an incomplete workspace. Commands report "not in a Gas Town workspace"
despite being inside one. Dependency checks fail for `bd` or `dolt`.

Gas Town workspace setup has three layers:

- **Town installation** (`gt install`): creates the HQ directory with
  `mayor/town.json`, beads database, Dolt server config, and shell
  wrappers. See [gt install](../../commands/install.md).
- **Rig registration** (`gt rig add`): clones a source repo into the
  town, registers it in `mayor/rigs.json`, and sets up beads tracking.
  See [gt rig](../../commands/rig.md).
- **Crew workspace** (`gt crew add`): creates a full git clone under
  `<rig>/crew/<name>/` with mail, settings, and optional feature
  branch. See [gt crew](../../commands/crew.md).

## Decision tree

### 1. Is this a fresh install or an existing workspace?

**Check:** Does `mayor/town.json` exist in the target directory?

```bash
ls ~/gt/mayor/town.json 2>/dev/null && echo "exists" || echo "missing"
```

- **Missing** -> go to [Step 2: `gt install` failures](#2-gt-install-failures)
- **Exists but commands say "not in a workspace"** -> go to
  [Step 6: Workspace discovery failures](#6-workspace-discovery-failures)
- **Exists and working** -> go to
  [Step 3: Rig add failures](#3-rig-add-failures) or
  [Step 5: Crew add failures](#5-crew-add-failures)

### 2. `gt install` failures

`runInstall` at `install.go:95-499` runs a sequence of preflight
checks before any mutation, then scaffolds the workspace.

**a. Dependency: beads (`bd`) not available:**
`install.go:144-149` calls `deps.EnsureBeads(true)` unless
`--no-beads`. `EnsureBeads` at `beads.go:105-150` checks `PATH` for
`bd`, runs `bd version` with a 10-second timeout, compares against
`MinBeadsVersion = "0.57.0"` (`beads.go:19`), and if missing or
outdated, attempts `go install` into `~/.local/bin`.

**Check:**

```bash
bd version          # Is it on PATH?
which bd            # Where is it?
go version          # Is Go available for auto-install?
```

**Fixes:**
- If `bd` is missing and Go is available: `go install github.com/steveyegge/beads/cmd/bd@latest`
- If `bd` is too old: same command to upgrade
- If Go is not available: install Go first, or use `--no-beads` for a
  beads-free workspace (limited functionality)
- If `bd version` hangs (CI load): the 10-second timeout at
  `beads.go:45-47` covers this case. Retry when load is lower.

**b. Dependency: Dolt not available or not configured:**
`install.go:151-202` checks for `dolt` on PATH, then verifies global
Dolt identity via `doltserver.EnsureDoltIdentity()`. Missing identity
produces a remediation message:

```bash
dolt config --global --add user.name "Your Name"
dolt config --global --add user.email "you@example.com"
```

**c. Port conflict during Dolt setup:**
`install.go:173-184` checks if the Dolt port (default 3307) is
available. If held by another process, it tries to connect with a
MySQL client. A usable Dolt server on the port is reused; otherwise,
`doltserver.FindFreePort` suggests an alternative.

**Fix:** `gt install <args> --dolt-port <free-port>`

**d. Already-a-workspace guard:**
`install.go:122-133` — if the target is already an HQ and `--force`
is not set, the command aborts. Exception: if only `--wrappers` was
requested, wrappers are installed and the command succeeds.

**Fix:** `gt install --force` to reinitialize.

**e. Inside-existing-workspace guard:**
`install.go:135-142` — if the target is nested inside a different HQ
(e.g., `~/gt/some-rig/crew/joe`), the command aborts with a
remediation hint.

**f. Partial scaffolding:**
Both preflights (beads check, Dolt preflight) run before any
mutations. The comment at `install.go:151-153` is explicit: "a partial
install is never left behind." If a mutation step fails mid-sequence
(e.g., disk full during `os.MkdirAll`), subsequent steps may fail
with confusing errors.

**Fix:** Remove the partial directory and re-run `gt install`.

### 3. Rig add failures

`runRigAdd` at `rig.go:478-576+` validates inputs, resolves the
workspace, then calls `mgr.AddRig(opts)`.

**a. Not in a workspace:**
`rig.go:501-505` calls `workspace.FindFromCwdOrError()`. If you're
not inside a Gas Town HQ, this fails.

**Fix:** `cd ~/gt` (or wherever your HQ is) before running `gt rig add`.

**b. Beads not available:**
`rig.go:497-499` calls `deps.EnsureBeads(true)` before proceeding.
Same dependency check as `gt install` — see Step 2a.

**c. Invalid git URL:**
`rig.go:492-494` validates that the URL is a remote URL (https://,
git@, ssh://, s3://, file:///). Local paths need `file://` prefix or
use `--adopt` for existing directories.

**Fix:** Use the full remote URL, or `gt rig add <name> --adopt` for
local directories.

**d. Git clone failure:**
`mgr.AddRig` performs `git clone` of the source repo. Network
failures, authentication issues, or invalid URLs cause clone failures.

**Check:** Can you clone the repo manually?

```bash
git clone <url> /tmp/test-clone
```

**e. Invalid clone filter:**
`rig.go:541-554` only accepts `blob:none` or `tree:0` for
`--filter`. Other values are rejected.

**f. Rig already exists:**
If `mayor/rigs.json` already has an entry for this name, use
`--force` to overwrite.

### 4. Rig add post-clone setup

After the git clone succeeds, `mgr.AddRig` performs several setup
steps. Failures here leave a cloned-but-not-fully-configured rig.

**a. Beads database initialization:**
Each rig gets its own beads database. If Dolt is unreachable during
rig add, the beads setup may fail.

**Check:** `gt dolt status` — is the Dolt server running?

**b. Rigs config save failure:**
After adding the rig, `config.SaveRigsConfig` writes
`mayor/rigs.json`. If the save fails (disk full, permissions), the
rig directory exists but isn't registered.

**Fix:** Manually verify `mayor/rigs.json` includes the new rig, or
re-run `gt rig add --adopt` to register the existing directory.

### 5. Crew add failures

`gt crew add <name>` creates a full git clone under
`<rig>/crew/<name>/`.

**a. Invalid crew name:**
`validateCrewName` at `crew/manager.go:106-126` rejects names with
hyphens, dots, spaces, path separators, or traversal sequences. These
characters break agent ID parsing.

**Check:** The error message suggests an underscore-substituted
alternative. Use it.

**b. Crew already exists:**
`ErrCrewExists` — the directory already exists.

**Fix:** Use a different name, or `gt crew remove <name>` first.

**c. Clone failure:**
`crew add` performs a full git clone from the rig's source repo. If
the clone is interrupted (network, disk space), the crew workspace is
left in a partial state — the directory exists but `.git` may be
incomplete.

**Check:**

```bash
ls -la <rig>/crew/<name>/.git/
git -C <rig>/crew/<name>/ status
```

**Fix:** `gt crew remove <name>` to clean up, then retry.

**d. Remote sync failure:**
After cloning, the crew manager syncs the remote from the rig's
`mayor/rig` configuration. If the rig's remotes are misconfigured,
the sync fails.

**e. Setup hook failure:**
Post-clone setup hooks (overlay file copies, beads redirect setup)
are run after the clone. Failures are typically non-fatal but may
leave the workspace without proper integration.

### 6. Workspace discovery failures

Commands report "not in a Gas Town workspace" despite being inside
one.

**a. Missing `mayor/town.json`:**
`workspace.Find` at `find.go:33-64` walks up from the current
directory looking for `mayor/town.json` (primary marker) or `mayor/`
(secondary marker). If neither exists, the workspace isn't found.

**Check:** `ls -la ~/gt/mayor/town.json` — does the marker exist?

**Fix:** If the file was deleted, `gt install --force` in the town
root directory recreates it.

**b. Symlink confusion:**
`workspace.Find` does NOT resolve symlinks (`find.go:32`). If you're
inside a symlinked directory, the walk-up may not find the workspace
markers.

**Fix:** Use the canonical (non-symlinked) path.

**c. Environment variable fallback:**
`FindFromCwdOrError` at `find.go:90-113` falls back to `GT_TOWN_ROOT`
and `GT_ROOT` env vars if the CWD walk fails. The fallback verifies
the env-var path actually contains a workspace marker. If the env var
points to a stale or deleted directory, the fallback fails silently
and the command reports "not in a workspace."

**Check:**

```bash
echo $GT_TOWN_ROOT
echo $GT_ROOT
```

**Fix:** Update or unset the stale env var.

**d. Nuked worktree edge case:**
`FindFromCwdWithFallback` at `find.go:119-137` is the most tolerant
variant, designed for `gt done` which runs after the polecat's
worktree is nuked by the Witness. If `os.Getwd()` itself errors
(working directory deleted), it falls back to `GT_TOWN_ROOT`.

### 7. Dependency version mismatches

**a. `bd` version too old:**
`deps.CheckBeads()` at `beads.go:36-67` compares the installed
version against `MinBeadsVersion`. `BeadsTooOld` status is returned
with the current and required versions.

**b. `dolt` version too old:**
`deps.CheckDolt()` compares against `MinDoltVersion = "1.82.4"`
(`dolt.go:16`). Unlike beads, dolt cannot be auto-installed — the
error message points to the Dolt installation URL.

**c. `dolt version` exec failure:**
`DoltExecFailed` (binary present but non-zero exit from `dolt version`)
is distinct from `DoltUnknown` (zero exit but unparseable output).
Broken dolt installs are a real operational hazard.

**Check:** `dolt version` — does it run cleanly?

## Related investigation workflows

- [Investigating: data-plane failures](data-plane.md) -- Dolt server
  issues that affect `gt install` Dolt preflight and rig add beads
  setup.
- [Investigating: agent lifecycle](agent-lifecycle.md) -- agent
  start failures after workspace setup is complete.
