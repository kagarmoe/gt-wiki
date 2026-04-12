---
title: gt-proxy-server
type: binary
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/cmd/gt-proxy-server/main.go
  - /home/kimberly/repos/gastown/cmd/gt-proxy-server/config.go
  - /home/kimberly/repos/gastown/internal/proxy/server.go
  - /home/kimberly/repos/gastown/internal/proxy/exec.go
  - /home/kimberly/repos/gastown/internal/proxy/git.go
  - /home/kimberly/repos/gastown/Makefile
tags: [binary, proxy, mtls, sandbox, server]
---

# gt-proxy-server

The host-side mTLS proxy server that lets sandboxed polecat containers call
`gt`/`bd` and fetch/push to host-side git repos via authenticated HTTP.
Long-lived process: binds a TLS listener on the host, accepts mTLS
connections from polecat clients, and executes a restricted set of
subcommands on their behalf. One of the three binaries produced by
[`make build`](../files/makefile.md).

**Also known as:** `gt-proxy-server` (binary on disk, produced by
`./cmd/gt-proxy-server`), `proxy-server` (docs convention — deferred
mapping), the "mTLS proxy" (in code comments at
`/home/kimberly/repos/gastown/internal/proxy/server.go:276`).

## What it actually does

### Entry point

`/home/kimberly/repos/gastown/cmd/gt-proxy-server/main.go` is 213 lines.
Unlike `cmd/gt/main.go` which is a 12-line stub
(see [gt.md](gt.md)), this `main.go` does real work: it parses flags,
loads a JSON config file, merges the two (flags override file), loads or
generates a CA, constructs a `proxy.Config` struct, and hands everything
to `proxy.New` + `proxy.Server.Start` from the `internal/proxy` package.

Package-level file comment (`cmd/gt-proxy-server/main.go:1-3`):

> gt-proxy-server is the mTLS proxy server for sandboxed polecat execution.
> It runs on the host and allows containers to call gt/bd and access git
> repos via authenticated, authorized HTTP endpoints.

### Flags

`cmd/gt-proxy-server/main.go:29-39` — six CLI flags, all with defaults:

| Flag               | Default                                       | Purpose                                                             |
|--------------------|-----------------------------------------------|---------------------------------------------------------------------|
| `-config`          | `~/gt/.runtime/proxy/config.json`             | Path to JSON config file                                            |
| `-listen`          | `0.0.0.0:9876`                                | mTLS listen address (the wire endpoint containers connect to)       |
| `-admin-listen`    | `127.0.0.1:9877`                              | Local-only admin HTTP server (no TLS); empty string disables        |
| `-ca-dir`          | `~/gt/.runtime/ca`                            | Directory holding `ca.crt` / `ca.key`                               |
| `-allowed-cmds`    | `gt,bd`                                       | Comma-separated binary names polecats may execute                   |
| `-allowed-subcmds` | auto-discovered via `gt proxy-subcmds`        | Semicolon-separated `cmd:sub1,sub2` allowlist (see below)           |
| `-town-root`       | `$GT_TOWN` or `~/gt`                          | Gas Town root directory passed to `proxy.Config`                    |

Flag/config merge rule (`cmd/gt-proxy-server/main.go:59-80`): explicitly
set flags (detected via `flag.Visit`) always win; otherwise the config
file value is used if present; otherwise the hardcoded default.

### Subcommand allowlist discovery

`cmd/gt-proxy-server/main.go:158-171` — `discoverAllowedSubcmds()` shells
out to `gt proxy-subcmds` at flag-parse time to auto-discover the allowed
subcommand list. Falls back to `defaultAllowedSubcmds`
(`main.go:24-26`) if the command is missing or returns empty:

```
gt:prime,hook,done,mail,nudge,mol,status,handoff,version,convoy,sling
bd:create,update,close,show,list,ready,dep,export,prime,stats,blocked,doctor
```

The comment at `main.go:22-23` states the intent: "Dangerous subcommands
(e.g. `gt polecat`, `gt rig`, `gt admin`, `gt nuke`) are excluded."

### Config file shape

`cmd/gt-proxy-server/config.go:12-59` defines `ProxyConfig`, loaded from
JSON at the `-config` path:

| Field                 | JSON key                | Purpose                                                                               |
|-----------------------|-------------------------|---------------------------------------------------------------------------------------|
| `ListenAddr`          | `listen_addr`           | Overrides `-listen`                                                                   |
| `AdminListenAddr`     | `admin_listen_addr`     | Overrides `-admin-listen`                                                             |
| `CADir`               | `ca_dir`                | Overrides `-ca-dir`                                                                   |
| `TownRoot`            | `town_root`             | Overrides `-town-root`                                                                |
| `AllowedCommands`     | `allowed_commands`      | Overrides `-allowed-cmds`                                                             |
| `AllowedSubcommands`  | `allowed_subcommands`   | Overrides `-allowed-subcmds`                                                          |
| `ExtraSANIPs`         | `extra_san_ips`         | Extra IP SANs in the server TLS cert (for NAT / VPN / additional LAN IPs)             |
| `ExtraSANHosts`       | `extra_san_hosts`       | Extra DNS SANs in the server TLS cert (for /etc/hosts entries, mDNS, split-horizon)   |

Missing file is silently treated as an empty config, not an error
(`config.go:64-77`). JSON parse errors are fatal.

The `ExtraSANIPs` / `ExtraSANHosts` fields exist because the server
auto-detects local interface IPs for cert SANs
(`internal/proxy/server.go:336-390`), but external NAT IPs (the address
shown by `curl ifconfig.me`) cannot be auto-detected — they must be
declared explicitly. See the comment at `config.go:35-48`.

### Main loop / core behavior

After config loading, `main.go:94-153`:

1. **CA load/generate** — `proxy.LoadOrGenerateCA(*caDir)`
   (`main.go:94`). Reads `ca.crt` + `ca.key` from disk, creating a fresh
   CA if they don't exist.
2. **Parse extra SAN IPs/hosts** (`main.go:107-128`) — skips invalid IP
   strings with a warning, filters empty hostnames.
3. **Construct `proxy.Config`** (`main.go:130-138`) — an all-fields
   struct passed to `proxy.New`.
4. **`proxy.New(cfg, ca)`** — validates `TownRoot` is non-empty and
   absolute (`internal/proxy/server.go:92-98`), resolves each allowed
   command's absolute path via `exec.LookPath` at startup to prevent
   PATH hijacking (`internal/proxy/server.go:115-125`), builds
   subcommand allowlists, and applies defaults for
   `MaxConcurrentExec=32`, `ExecRateLimit=10`, `ExecRateBurst=20`,
   `ExecTimeout=60s` (`internal/proxy/server.go:141-156`).
5. **Signal handler** (`main.go:146-147`) — `signal.NotifyContext` on
   SIGINT/SIGTERM produces the context passed to `Start`.
6. **`srv.Start(ctx)`** (`main.go:149-152`) — blocks until the context
   is cancelled; errors cause exit code 1.

### The Start() behavior

`internal/proxy/server.go:205-328` — `Server.Start` does the heavy lifting:

- Builds a TLS config that requires client certs
  (`tls.RequireAndVerifyClientCert`), pins TLS 1.3 min, disables session
  tickets, and installs a `VerifyPeerCertificate` hook that checks the
  leaf cert serial against an in-memory deny list
  (`server.go:210-230`).
- Registers two HTTP routes on the mTLS listener
  (`server.go:232-234`):
  - `POST /v1/exec` → `handleExec` (run `gt`/`bd` subcommand)
  - `/v1/git/` → `handleGit` (git smart-HTTP bridge)
- Issues the server TLS cert from the CA with IP SANs for loopback,
  local interfaces, and `ExtraSANIPs`, and DNS SANs for
  `"gt-proxy-server"` plus `ExtraSANHosts`
  (`server.go:246-263`).
- Starts a second plain-HTTP admin server on `AdminListenAddr`
  (default `127.0.0.1:9877`), serving two routes
  (`server.go:286-314`):
  - `POST /v1/admin/issue-cert` → `handleIssueCert` (mint a polecat
    client cert signed by this CA)
  - `POST /v1/admin/deny-cert` → `handleDenyCert` (revoke a cert by
    serial; any live or future TLS handshake presenting that cert is
    rejected)
- On `ctx.Done()`, drains with a 30-second shutdown deadline
  (`server.go:316-327`).

### The /v1/exec handler

`internal/proxy/exec.go:26-125` — what each authenticated polecat call
actually hits:

1. Method check: POST only.
2. 1 MiB max request body (`exec.go:33`).
3. Extract identity from client cert CN
   (`extractIdentity(r)` at `exec.go:36`; parses `gt-<rig>-<name>` →
   `<rig>/<name>`).
4. Decode JSON body → `execRequest{Argv []string}`.
5. Validate `argv[0]` is in the allowlist (`exec.go:51-54`).
6. If the command has a subcommand allowlist, require `argv[1]` to be
   in it (`exec.go:57-67`).
7. Substitute `argv[0]` with the absolute resolved path to prevent PATH
   hijacking (`exec.go:70-74`).
8. Per-client rate limiting via `golang.org/x/time/rate` keyed on cert
   CN (`exec.go:81-85`).
9. Global concurrency cap via semaphore — excess requests return 503
   immediately (`exec.go:88-95`).
10. Per-command deadline from `Config.ExecTimeout` via
    `context.WithTimeout` (`exec.go:97-102`).
11. `runCommand(ctx, argv, identity)` — exec the subprocess with
    minimal env (HOME + PATH only; see
    `internal/proxy/server.go:525-533`).
12. Audit log (does NOT log full argv — "it may contain tokens or
    secrets" per `exec.go:105`).
13. Always returns HTTP 200 with `{stdout, stderr, exitCode}` JSON —
    subprocess exit code is in the body, NOT the HTTP status.

### The /v1/git/ handler

`internal/proxy/git.go:1-50` — a full git smart-HTTP bridge to bare
repos at `~/gt/<rig>/.repo.git`. Containers never talk to GitHub;
fetch/push flow through the proxy. Salient behaviors per the package
comment:

- `rig` name validated against `^[a-zA-Z0-9_-]+$` before path construction.
- `git-receive-pack` (push) requires a valid mTLS client cert; the CN
  polecat name is extracted and every pushed ref must match
  `refs/heads/polecat/<name>-*`. Any other ref rejected with 403
  before git sees the body.
- `git-upload-pack` (fetch) is unrestricted for any authenticated client.
- Subprocesses inherit only HOME + PATH (minimal env).

Full treatment of the `internal/proxy/` package is future work (not
mapped in this layer).

### Dependencies

`cmd/gt-proxy-server/main.go:6-20` — stdlib plus two internal imports:

```go
"github.com/steveyegge/gastown/internal/proxy"
"github.com/steveyegge/gastown/internal/util"
```

The `util` import is used only for `util.SetDetachedProcessGroup` on the
`gt proxy-subcmds` discovery exec (`main.go:160`). Everything else is
stdlib: `flag`, `log/slog`, `net`, `os`, `os/exec`, `os/signal`,
`path/filepath`, `strings`, `syscall`, `context`.

No cobra. No `internal/cmd`. No `internal/cli.Name()` dynamic renaming.
This binary is **not** affected by the `GT_COMMAND` env var that
rewrites the `gt` root command name.

### Build

Produced by `make build` via this line in
[../files/makefile.md](../files/makefile.md) (`Makefile:35-38`):

```make
go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY)-proxy-server ./cmd/gt-proxy-server
```

The same `LDFLAGS` used for `gt` (including `BuiltProperly=1`) are
applied here. However, because this binary does NOT import
`internal/cmd`, the `BuiltProperly` self-kill gate in
`/home/kimberly/repos/gastown/internal/cmd/root.go:94-107` never runs
for `gt-proxy-server`. The `-X` flag sets a variable in a package that
this binary doesn't link in — silently a no-op. Version metadata
injected by `make build` (`Version`, `Commit`, `BuildTime`) also has no
consumer here.

This binary therefore runs fine after `go build ./cmd/gt-proxy-server`
without `make build`, unlike `gt` which will self-kill. See
[gt.md](gt.md) for the contrasting `gt` behavior.

### Runtime relationship

Long-lived server. Started how? The binary has no `daemonize` flag; it
runs in the foreground and exits on SIGINT/SIGTERM. The expected
supervision mechanism (systemd unit, `gt daemon`, a shell wrapper,
manual `nohup`) is not visible from the sources read in this layer and
is deferred to a later mapping pass.

The binary it pairs with is [gt-proxy-client](gt-proxy-client.md),
which runs inside each polecat container and forwards `gt`/`bd`
invocations over mTLS to this server.

## Cross-references

- [gt](gt.md) — the main CLI binary. Sibling under `cmd/`. Unlike
  `gt-proxy-server`, `gt` is a 12-line cobra stub that delegates to
  `internal/cmd/`.
- [gt-proxy-client](gt-proxy-client.md) — the container-side counterpart.
  Connects to this server over mTLS.
- [Makefile](../files/makefile.md) — canonical build recipe; the
  `build` target produces this binary at `./gt-proxy-server`.
- [Dockerfile](../files/dockerfile.md) — the container image build.
- [../inventory/go-packages.md](../inventory/go-packages.md) —
  enumerates `cmd/gt-proxy-server` as 2 Go files + 1 test.
- [../inventory/repo-root.md](../inventory/repo-root.md) — lists
  `cmd/` as holding the three built binaries.

The backing implementation lives in `/home/kimberly/repos/gastown/internal/proxy/`
(`ca.go`, `denylist.go`, `exec.go`, `git.go`, `server.go`); that package
is not yet mapped as a wiki entity page.

## Notes / open questions

- The default config path is `~/gt/.runtime/proxy/config.json`. That
  directory shape (and who writes the file) is not mapped yet — likely
  provisioned by [`gt polecat`](../commands/polecat.md) / `gt rig` / a setup workflow.
- `discoverAllowedSubcmds` runs `gt proxy-subcmds` at startup. That
  subcommand isn't yet mapped — lives at
  `/home/kimberly/repos/gastown/internal/cmd/proxy_subcmds.go`.
- The CA files in `~/gt/.runtime/ca/` are the root of trust for both
  server and client certs. `proxy.LoadOrGenerateCA` will create them on
  first run — silent first-time bootstrap. Rotation strategy not
  documented in the code read here.
- `MaxConcurrentExec=32`, `ExecRateLimit=10 req/s`,
  `ExecRateBurst=20`, `ExecTimeout=60s` — these defaults are set in
  `Server.New` (`internal/proxy/server.go:141-156`) and are NOT
  surfaced as CLI flags or config file fields. Operators cannot override
  them without code changes.
- `rateLimiters` is a `sync.Map` keyed on client cert CN with no
  eviction (`internal/proxy/server.go:197-198`, comment at lines
  193-196). Each unique CN leaks ~200 bytes. Documented as acceptable
  for dozens of polecats; not for thousands.
- Admin HTTP server is plain HTTP bound to `127.0.0.1` — any local
  process can mint or revoke polecat certs. Stated security model at
  `internal/proxy/server.go:283-286`: "any process on the same host
  can reach it, which is the intended access model."
- Subprocess env is HOME + PATH only. No `GIT_EXEC_PATH`, no `GT_TOWN`,
  no OTEL exporters. This is intentional — see
  `/home/kimberly/repos/gastown/internal/proxy/server.go:520-533`.
- Out of scope for this layer: `docs/proxy-server.md` comparison —
  deferred to phase 3.
