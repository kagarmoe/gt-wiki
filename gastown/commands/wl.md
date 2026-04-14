---
title: gt wl
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/internal/cmd/wl.go
  - /home/kimberly/repos/gastown/internal/wasteland/
  - /home/kimberly/repos/gastown/internal/doltserver/
tags: [command, work, wasteland, federation, dolthub, rig-registry]
---

# gt wl

Wasteland federation commands. **Despite `wl.go` being small, `wl`
does not stand for "worklist" — it stands for "wasteland"** — a
federation of Gas Towns via DoltHub, where each rig holds a
sovereign fork of a shared commons database containing the wanted
board, rig registry, and validated completions.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** `GroupWork` ("Work Management") (`wl.go:24`)
**Polecat-safe:** no
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/wl.go` (full
file, 154 lines). Delegates to `internal/wasteland` for the service
layer and `internal/doltserver` for DoltHub credentials.

### Invocation

```
gt wl <subcommand> [args]
```

`wlCmd.RunE = requireSubcommand` (`:26`) — bare `gt wl` errors with
help text.

### Subcommands

Only one is wired in this file (`wl.go:69-70`):

| subcommand | source | one-liner |
|---|---|---|
| `join <upstream>` | `:39-63`, `runWlJoin :73-154` | Fork a wasteland's commons database, clone locally, register the rig, push registration, save wasteland config. |

The Long text on the parent (`:27-36`) mentions the concept of
"wanted board" + "rig registry" + "validated completions" but no
other subcommands are present in this file. Other wasteland
subcommands (like the `gt wl browse` hinted at `:152`) must be
wired elsewhere, if at all.

### `join` pipeline (`runWlJoin`, `wl.go:73-154`)

1. **Parse upstream** (`:77-80`) — `wasteland.ParseUpstream(upstream)`
   validates the `<org>/<db>` shape early before touching Dolt.
2. **Require credentials** (`:82-91`):
   - `DOLTHUB_TOKEN` via `doltserver.DoltHubToken()`. Error message
     points to `https://www.dolthub.com/settings/tokens`.
   - `DOLTHUB_ORG` via `doltserver.DoltHubOrg()`. The user's own
     DoltHub org for the fork.
3. **Find town root** via `workspace.FindFromCwdOrError` (`:94-97`).
4. **Fast path: already joined?** (`:101-110`) —
   `wasteland.LoadConfig(townRoot)`:
   - If the existing config's `Upstream` matches the passed
     upstream, print "Already joined" + handle/fork/local and
     return success.
   - If it's joined to a *different* upstream, error with
     "already joined to X; run `gt wl leave` first". Note: `gt wl
     leave` is referenced but not implemented in this file.
5. **Load town config** (`:113-117`) — `config.LoadTownConfig` of
   the primary marker file. Only needed for a fresh join, so the
   fast path skips it to avoid failing on unrelated town-config
   errors for the no-op case.
6. **Handle + display name** (`:120-132`):
   - `handle` defaults to the DoltHub org.
   - `displayName` defaults to `townCfg.PublicName` or
     `townCfg.Name`.
7. **Call service** (`:137-146`):
   ```go
   svc := wasteland.NewService()
   svc.OnProgress = func(step string) { print step }
   cfg, err := svc.Join(upstream, forkOrg, token, handle,
       displayName, ownerEmail, gtVersion, townRoot)
   ```
   According to the Long text (`:43-49`), the service layer:
   1. Forks the upstream commons to your DoltHub org.
   2. Clones the fork locally.
   3. Registers your rig in the `rigs` table.
   4. Pushes the registration to your fork.
   5. Saves wasteland configuration locally.
8. **Report success** (`:148-152`) — handle, fork path, local
   path, and a hint: `Next: gt wl browse — browse the wanted
   board`.

### `gtVersion` is hardcoded

`wl.go:135` — `gtVersion := "dev"`. The registration in the rigs
table will always say `"dev"` regardless of the actual built
version. Likely a TODO.

### Flags (on `wlJoinCmd`, `:66-67`)

| flag | default | notes |
|---|---|---|
| `--handle <name>` | `""` | Rig handle for registration. Defaults to DoltHub org. |
| `--display-name <text>` | `""` | Display name for the rig registry. Defaults to town's `PublicName`/`Name`. |

## Notes / open questions

- **`wl` ≠ worklist.** This is important for wiki readers: the
  name is short for **wasteland** (as in Gas Town's post-apoc
  aesthetic, see [../binaries/gt.md](../binaries/gt.md)). There
  is no "worklist" command — the equivalent surface is
  [ready](ready.md) for town-wide ready work or
  [scheduler](scheduler.md) for deferred dispatch.
- **Thin surface, thick service layer.** Almost all logic lives
  in `internal/wasteland/Service` — see
  [../packages/wasteland.md](../packages/wasteland.md) for the
  federation model, Spider Protocol fraud detection, trust tiers,
  the `rigs` table schema, and the fork-and-push flow.
- **`gt wl browse` and `gt wl leave` are referenced but not
  wired here.** Expect other wl subcommands in sibling files or
  planned but unimplemented.
- **Required env vars** (`DOLTHUB_TOKEN`, `DOLTHUB_ORG`) turn
  this into an externally-dependent command. Offline use is
  impossible.
- **`--display-name` / `--handle` flags** take precedence over
  the town config defaults, so this is how a rig advertises a
  "public persona" distinct from its internal identity.
- **Related.** [config](config.md) (reads town config + writes
  wasteland config), [doctor](doctor.md) (may check wasteland
  health), [convoy](convoy.md) (the in-town tracking unit;
  wasteland is cross-town).
