---
title: gt role
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/cmd/role.go
tags: [command, agents, role, identity, env, detection, cwd]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gt role

Show or inspect the current agent role. Role is the primary identity
signal for every Gas Town agent — it determines the home directory,
the actor string used by beads/mail attribution, and which role
definition (session shape, health thresholds, prompt template)
applies.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupAgents` ("Agent Management") (`role.go:43`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/role.go`
(767 lines, not counting tests). Registration at `role.go:138-150`
attaches six subcommands to `roleCmd` and wires two flags on
`role home`.

The parent `roleCmd` (`role.go:41-53`) has `RunE: runRoleShow`, so
`gt role` with no arguments is equivalent to `gt role show`. This
is unusual inside the Agents group — `mayor`, `deacon`, `refinery`,
`witness`, `polecat`, and `session` all use `requireSubcommand`
and refuse to do anything without an explicit subcommand.

### Invocation

```
gt role [show]
gt role home [ROLE] [--rig <r>] [--polecat <p>]
gt role detect
gt role list
gt role env
gt role def <role>
```

### Subcommands

| subcommand | var | source | run-fn |
|---|---|---|---|
| (default) / `show` | `roleShowCmd` | `role.go:55-64` | `runRoleShow` (`role.go:474-508`) |
| `home`    | `roleHomeCmd`   | `role.go:66-79`  | `runRoleHome`   (`role.go:510-561`) |
| `detect`  | `roleDetectCmd` | `role.go:81-88`  | `runRoleDetect` (`role.go:563-602`) |
| `list`    | `roleListCmd`   | `role.go:90-99`  | `runRoleList`   (`role.go:604-623`) |
| `env`     | `roleEnvCmd`    | `role.go:101-113`| `runRoleEnv`    (`role.go:625-684`) |
| `def`     | `roleDefCmd`    | `role.go:115-130`| `runRoleDef`    (`role.go:686-767`) |

### Role detection cascade

The canonical entry point is `GetRole()` at `role.go:154-169`,
which gets the cwd and town root then calls
`GetRoleWithContext(cwd, townRoot)` at `role.go:172-240`. The
cascade:

1. **`GT_ROLE` env var is authoritative** if set (`role.go:179`,
   `role.go:187-227`).
   - `parseRoleString` (`role.go:339-394`) parses strings like
     `"mayor"`, `"deacon"`, `"gastown/witness"`,
     `"gastown/polecats/Toast"`, `"gastown/crew/max"`.
   - If `parseRoleString` didn't populate `rig` or `polecat` but
     they are required for the role (`needsRig`, `needsPolecat`
     at `role.go:212-213`), the code reads `GT_RIG` and
     `GT_CREW`/`GT_POLECAT` env vars as supplements
     (`role.go:197-207`).
   - If still missing after env vars, fills from cwd detection
     and sets `EnvIncomplete=true`. This gets a stderr warning
     from `runRoleEnv` at `role.go:651-653`.
   - If cwd detection suggests a different role than GT_ROLE,
     sets `Mismatch=true` (`role.go:224-227`) — `runRoleShow`
     prints a warning block at `role.go:497-505`.
2. **Cwd-based detection** if `GT_ROLE` is unset. `detectRole`
   (`role.go:244-336`) takes the cwd relative to town root and
   walks directory prefixes in a fixed order:
   - `mayor/...` → RoleMayor (town-level mayor).
   - `deacon/dogs/boot/...` → RoleBoot (checked before deacon).
   - `deacon/dogs/<name>/...` → RoleDog with `Polecat=<name>`.
   - `deacon/...` → RoleDeacon.
   - `<rig>/mayor/...` → RoleMayor with `Rig=<rig>`.
   - `<rig>/witness/...` → RoleWitness with `Rig=<rig>`.
   - `<rig>/refinery/...` → RoleRefinery with `Rig=<rig>`.
   - `<rig>/polecats/<name>/...` → RolePolecat with
     `Polecat=<name>`.
   - `<rig>/crew/<name>/...` → RoleCrew with `Polecat=<name>`
     (the crew member name is stored in the `Polecat` field —
     see comments at `role.go:285,330`).
   - Otherwise: RoleUnknown. Town root itself (`relPath == "."`)
     is explicitly not treated as Mayor (`role.go:262-266`):
     "Town root is a neutral location — don't infer any role
     from it."

### `ActorString` — the canonical beads attribution string

`RoleInfo.ActorString()` (`role.go:402-433`) formats the detected
role into the actor string used by beads `created_by`:

- Mayor → `"mayor"`
- Deacon → `"deacon"`
- Witness → `"<rig>/witness"` (or `"witness"` if rig missing)
- Refinery → `"<rig>/refinery"`
- Polecat → `"<rig>/polecats/<name>"`
- Crew → `"<rig>/crew/<name>"`
- Boot → `"deacon-boot"` (hyphenated, matching `BD_ACTOR`)
- Other → `string(info.Role)` fallback

This string is the identity the rest of Gas Town sees for any
command that creates beads or sends mail.

### `getRoleHome`

`getRoleHome(role, rig, polecat, townRoot)` at `role.go:436-472`
returns the canonical home directory per role. Used by both
`roleEnvCmd` (to set `GT_ROLE_HOME`) and `roleHomeCmd` (to print
it).

### `show` / default

`runRoleShow` (`role.go:474-508`) prints role, source
(`"env"` or `"cwd"`), home, rig, polecat, and a mismatch warning
block if `GT_ROLE` and cwd disagree. Read-only.

### `home`

`runRoleHome` (`role.go:510-561`). Takes an optional `[ROLE]`
positional and `--rig`/`--polecat` flag overrides. The flag
`--polecat` requires `--rig` to be set (`role.go:525-527`) — the
check exists to prevent "strange merges" where you supply a
worker name without a rig context.

Starts from the current detected role, applies argument/flag
overrides, computes the home with `getRoleHome`, errors out if
computation returns empty, and otherwise prints the path. A
stderr warning is emitted if the computed home is outside the
cwd (`role.go:555-557`).

### `detect`

`runRoleDetect` (`role.go:563-602`) forces cwd-only detection,
bypassing `GT_ROLE`. Used for debugging the detection logic.
Prints the detected role, the cwd, rig/worker if set, and a
mismatch block if `GT_ROLE` disagrees.

### `list`

`runRoleList` (`role.go:604-623`) prints a hardcoded table of the
six known roles and one-line descriptions:

- `mayor`    — Global coordinator at `mayor/`
- `deacon`   — Background supervisor daemon
- `witness`  — Per-rig polecat lifecycle manager
- `refinery` — Per-rig merge queue processor
- `polecat`  — Worker with persistent identity, ephemeral sessions
- `crew`     — Persistent worker with own worktree

`boot` and `dog` are NOT in this list (they are detected but not
advertised here). The `Unknown` sentinel is also omitted.

### `env`

`runRoleEnv` (`role.go:625-684`). Prints shell `export` statements
(PowerShell `$env:...=...` on Windows per `role.go:676-679`) for
the current role's env vars, suitable for `eval $(gt role env)`.

The env map comes from `config.AgentEnv` (`role.go:661-666`) —
the canonical source of agent-level env vars. `GT_ROLE_HOME` is
added explicitly (`role.go:667`). Keys are sorted so output is
reproducible. This is the read-only export counterpart of
whatever wrote the env to begin with.

Stderr warnings fire on two conditions:
- `info.EnvIncomplete` → `"env vars incomplete, filled from cwd"`
- `home != cwd && !strings.HasPrefix(cwd, home)` → `"cwd (...)
  is not within role home (...)"`

### `def`

`runRoleDef <role>` (`role.go:686-767`). Displays the effective
role definition after layered overrides:

1. Built-in defaults embedded in the binary.
2. Town-level overrides at `<town>/roles/<role>.toml`.
3. Rig-level overrides at `<rig>/roles/<role>.toml`.

Implementation: validates the role name against `config.AllRoles()`,
resolves the rig path from the current detected role, and calls
`config.LoadRoleDefinition(townRoot, rigPath, roleName)`. Prints
the scope, session config (pattern, work_dir, needs_pre_sync,
start_command), env block, health config (ping_timeout,
consecutive_failures, kill_cooldown, stuck_threshold,
hung_session_threshold), and optional nudge/prompt fields.

### Flags

| flag | type | default | scope | source |
|---|---|---|---|---|
| `--rig <name>`    | string | `""` | `home` | `role.go:148` |
| `--polecat <name>`| string | `""` | `home` | `role.go:149` |

No flags on the other subcommands.

## Related commands

- [../binaries/gt.md](../binaries/gt.md) — root.
- [README.md](README.md) — command index.
- [prime.md](prime.md) — `gt prime` uses `GetRole()` to brief the
  agent on identity at session start; role detection is the
  foundation of the primed context.
- [whoami.md](whoami.md) — prints the actor string (from
  `ActorString`) without the surrounding mismatch/home noise.
- [mayor.md](mayor.md), [deacon.md](deacon.md),
  [polecat.md](polecat.md), [refinery.md](refinery.md),
  [witness.md](witness.md) — the six CLI commands that manage the
  six roles `gt role list` enumerates. The role page documents
  the role *as a persona*; these command pages document the
  lifecycle commands.
- [boot.md](boot.md), [dog.md](dog.md) — the two additional roles
  that `detectRole` recognizes but `roleListCmd` does not
  advertise.
- [agents.md](agents.md) — `gt agents list` ranks agents by role;
  mismatches reported here may show up as "unknown" entries in
  `gt agents list`.
- [session.md](session.md) — polecat session commands that
  depend on `parseAddress` / `ActorString` conventions defined
  in this file.
- [config.md](config.md) — `config.LoadRoleDefinition` and
  `config.AgentEnv` are implemented in
  [internal/config](../packages/config.md); the shape of
  `<town>/roles/<role>.toml` lives there.

## Notes / open questions

- **Role pages (personas)** for mayor, deacon, witness, refinery,
  polecat, crew, boot, dog are pending Batch 6 — not yet created.
  Link to plain text.
- **`gt role` default behavior is atypical.** Every other command
  in the Agents group errors on bare invocation; `gt role` prints
  role info. This is a good UX for the human interactive case but
  means scripts cannot rely on "no args → usage".
- **`parseRoleString` fallback is odd.** At `role.go:390-393`,
  if a compound role string's second segment isn't one of the
  recognized keywords, it assumes the second segment is a polecat
  name and returns `RolePolecat`. This means `"gastown/madeup"`
  silently becomes a polecat named `"madeup"`.
- **Rename of `Polecat` field for crew/dog storage.** The
  `RoleInfo.Polecat` field doubles as the "worker name" slot for
  crew members (`role.go:330`) and dogs (`role.go:285`). Any
  downstream code reading `info.Polecat` on a crew or dog role
  must know this overload.
- **`roleListCmd` is hand-maintained** and does not consult
  `config.AllRoles()`. If a new role (e.g. boot, dog) ever
  qualifies for the human-readable list, both places need
  updating.
- **`GT_ROLE_HOME` is only emitted by `gt role env`.** The
  env var is consumed elsewhere but there is no single place
  that documents where — a follow-up grep target.
