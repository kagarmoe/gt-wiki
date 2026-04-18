---
title: internal/proxy
type: package
status: verified
topic: gastown
created: 2026-04-16
updated: 2026-04-18
sources:
  - /home/kimberly/repos/gastown/internal/proxy/ca.go
  - /home/kimberly/repos/gastown/internal/proxy/denylist.go
  - /home/kimberly/repos/gastown/internal/proxy/exec.go
  - /home/kimberly/repos/gastown/internal/proxy/git.go
  - /home/kimberly/repos/gastown/internal/proxy/server.go
tags: [package, proxy, mtls, ca, tls, security, git, exec, polecat]
phase3_audited: 2026-04-16
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase4_audited: 2026-04-16
phase4_findings: [none]
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/proxy

mTLS CA management and HTTP proxy server for sandboxed polecat
execution. This is the largest of the gap packages (1,363 lines across
5 files) and provides the complete server-side infrastructure for the
`gt-proxy-server` binary: certificate authority operations, an mTLS HTTP
server with exec and git smart-HTTP endpoints, certificate revocation
via deny lists, per-client rate limiting, and branch-scoped git push
authorization.

**Go package path:** `github.com/steveyegge/gastown/internal/proxy`
**File count:** 5 non-test Go files (ca.go 194, denylist.go 45,
exec.go 228, git.go 363, server.go 533; 1,363 total).
**Imports (notable):** `crypto/ecdsa`, `crypto/elliptic`, `crypto/tls`,
`crypto/x509` (TLS infrastructure), `golang.org/x/time/rate`
(per-client rate limiting), `net/http` (HTTP server).
**Imported by:** `cmd/gt-proxy-server/main.go` (the binary entry
point).

## What it actually does

### Certificate authority (ca.go)

**`CA` struct** (`ca.go:22-26`): holds the parsed CA certificate,
PEM-encoded cert bytes, and ECDSA P-256 private key.

**`GenerateCA(dir string) (*CA, error)`** (`ca.go:29-90`) -- creates a
self-signed CA:
- Generates ECDSA P-256 key (`ca.go:35`).
- Creates a self-signed cert with `CommonName: "GasTown CA"`, 10-year
  validity, `KeyUsageCertSign | KeyUsageCRLSign` (`ca.go:44-52`).
- Writes `ca.crt` and `ca.key` via atomic rename (write to `.tmp`
  then `os.Rename`) to prevent inconsistent state on crash
  (`ca.go:69-82`).

**`LoadOrGenerateCA(dir string) (*CA, error)`** (`ca.go:93-131`) --
loads existing CA from `ca.crt`/`ca.key` if present, otherwise
generates. Checks certificate expiry and rejects expired CAs with an
actionable error message (`ca.go:120-122`). Validates key type is
ECDSA (`ca.go:125-127`).

**Leaf certificate issuance:**
- `IssueServer(cn string, extraIPs []net.IP, extraDNSNames []string, ttl time.Duration)`
  (`ca.go:140-143`) -- issues a server cert with DNS SANs (cn +
  extras) and IP SANs for container-to-host connectivity.
- `IssuePolecat(cn string, ttl time.Duration)` (`ca.go:149-154`) --
  issues a client-auth cert for a polecat. Validates CN format
  `gt-<rig>-<name>` via `cnToIdentity` before issuing (`ca.go:150-152`).
- `issue(cn, dnsNames, ipAddrs, ttl, eku)` (`ca.go:158-194`) --
  shared implementation: generates leaf ECDSA key, creates cert signed
  by the CA, returns PEM-encoded cert and key.

### Deny list (denylist.go)

**`DenyList`** (`denylist.go:14-17`) -- thread-safe in-memory set of
revoked certificate serial numbers, keyed by lowercase hex string.

- `Deny(serial *big.Int)` (`denylist.go:27-31`) -- adds a serial.
- `IsDenied(serial *big.Int) bool` (`denylist.go:34-38`) -- checks
  membership.
- `Len() int` (`denylist.go:41-45`) -- current deny-list size.

Entries are never removed; the comment at `denylist.go:13` notes that
TTLs are short and the CA rotates periodically.

### Server (server.go)

**`Config` struct** (`server.go:25-61`): server configuration including
`ListenAddr`, `AllowedCommands`, `AllowedSubcommands` (per-command
subcommand allowlists), `TownRoot`, `Logger`, `ExtraSANIPs`,
`ExtraSANHosts`, `AdminListenAddr`, `MaxConcurrentExec` (default 32),
`ExecRateLimit` (default 10 req/s), `ExecRateBurst` (default 20),
`ExecTimeout` (default 60s).

**`New(cfg Config, ca *CA) (*Server, error)`** (`server.go:92-171`) --
validates `TownRoot` is absolute, resolves binary paths at startup via
`exec.LookPath` to prevent PATH hijacking (`server.go:116-125`),
builds subcommand allowlists, initializes the concurrency semaphore
and rate limiter defaults.

**`Start(ctx context.Context) error`** (`server.go:205-328`) --
the main server loop:
1. Configures mTLS with `RequireAndVerifyClientCert`, TLS 1.3 minimum,
   disabled session tickets, and a `VerifyPeerCertificate` callback
   that checks the deny list (`server.go:210-230`).
2. Registers HTTP routes: `POST /v1/exec` and `/v1/git/*`
   (`server.go:232-234`).
3. Issues a server cert from the CA with auto-detected IP SANs from
   local interfaces plus configured extras (`server.go:254-263`).
4. Binds and starts the mTLS listener (`server.go:266-280`).
5. Optionally starts a local admin HTTP server (no TLS, 127.0.0.1
   only) with `POST /v1/admin/deny-cert` and
   `POST /v1/admin/issue-cert` endpoints (`server.go:287-314`).
6. Blocks on context cancellation, then gracefully shuts down with a
   30-second deadline (`server.go:316-328`).

**Admin endpoints:**
- `handleIssueCert` (`server.go:414-478`) -- issues polecat client
  certs via `ca.IssuePolecat`. Takes `rig`, `name`, optional `ttl`
  (default 30 days). Returns cert, key, CA PEM, serial, and expiry.
- `handleDenyCert` (`server.go:492-518`) -- revokes a cert by serial
  number (hex string). Adds to the deny list for immediate TLS
  rejection.

**IP SAN detection** (`server.go:336-390`):
`serverListenIPs` enumerates local interface IPs when listening on
`0.0.0.0`/`::`, always includes loopback, filters out link-local
addresses. Notes that external NAT IPs require explicit configuration.

### Exec handler (exec.go)

**`handleExec`** (`exec.go:26-125`) -- `POST /v1/exec` endpoint:
1. Validates `argv[0]` against the command allowlist (`exec.go:51-54`).
2. Validates `argv[1]` against per-command subcommand allowlists if
   configured (`exec.go:57-67`).
3. Uses resolved absolute binary paths to prevent PATH hijacking
   (`exec.go:72-74`).
4. Per-client rate limiting by cert CN (`exec.go:77-85`).
5. Global concurrency cap via semaphore; rejects with 503 if all
   slots busy (`exec.go:88-95`).
6. Runs subprocess with timeout, minimal environment (HOME + PATH
   only + `GT_PROXY_IDENTITY`), returns stdout/stderr/exitCode as
   JSON (`exec.go:98-124`).
7. HTTP 200 always returned; exit code is in the JSON body
   (`exec.go:119` comment explains the design choice).

**Identity extraction** (`exec.go:144-183`):
- `extractIdentity` -- parses cert CN `gt-<rig>-<name>` to
  `<rig>/<name>`.
- `cnToIdentity` / `polecatName` -- the CN parsing uses
  `strings.LastIndex` for the rig/name separator, correctly handling
  hyphenated rig names like `gas-town` (`exec.go:155-166`).

**`runCommand`** (`exec.go:206-228`) -- subprocess execution with
`minimalEnv()` (HOME + PATH only, no server credentials leaked).

### Git smart-HTTP proxy (git.go)

Implements the git smart-HTTP protocol for bridging sandboxed
containers to host-side bare repositories at
`<townRoot>/<rig>/.repo.git`.

**Routes:**
- `GET /v1/git/<rig>/info/refs?service=git-upload-pack` -- ref
  advertisement for fetch.
- `POST /v1/git/<rig>/git-upload-pack` -- pack stream for fetch.
- `GET /v1/git/<rig>/info/refs?service=git-receive-pack` -- ref
  advertisement for push.
- `POST /v1/git/<rig>/git-receive-pack` -- pack stream for push.

**Security model** (documented in `git.go:25-39`):
- Rig name validated against `rigNameRe` (`^[a-zA-Z0-9_-]+$`) to
  prevent path traversal (`git.go:66, 91-93`).
- Repository existence checked before any response body is written
  (`git.go:98-106`).
- Push authorization: cert CN parsed to extract polecat name; every
  pushed ref must match `refs/heads/polecat/<name>-*`
  (`git.go:207-268`). This prevents polecats from pushing to branches
  outside their namespace.
- Upload-pack (fetch) is unrestricted for any authenticated client.
- Subprocesses get minimal environment (HOME + PATH only).

**`authorizeReceivePack`** (`git.go:211-268`) -- reads the pkt-line
ref-update section (up to 256 KiB, `git.go:222`) without loading the
entire pack into memory. Reconstructs the request body via
`io.MultiReader` so git receives the complete stream after
authorization (`git.go:266`).

**`validateReceivePackRefs`** (`git.go:311-363`) -- parses each
pkt-line record, strips capabilities after NUL byte, extracts the ref
name, and validates it starts with the allowed prefix.

## Related

- [gt-proxy-server](../binaries/gt-proxy-server.md) -- the binary
  that wraps this package.
- [gt-proxy-client](../binaries/gt-proxy-client.md) -- the
  container-side client that connects to this server.
- [internal/tmux](tmux.md) -- not directly imported, but the proxy
  server runs alongside tmux-managed agent sessions.
- [polecat role](../roles/polecat.md) -- polecats are the primary
  clients of the proxy, executing `gt`/`bd` commands through it.
- [rig concept](../concepts/rig.md) -- the rig name in addresses
  determines the bare repository path.
- [go-packages inventory](../inventory/go-packages.md)

## Outgoing calls

### Subprocess invocations
| Called binary | Command | Flags | Flag source | `file:line` |
|---|---|---|---|---|
| (dynamic) | (dynamic) | `argv[0] argv[1:]...` | API request | `exec.go:207` |
| `git-upload-pack` / `git-receive-pack` | `--stateless-rpc --advertise-refs` | `<repoPath>` | runtime | `git.go:158` |
| `git-upload-pack` / `git-receive-pack` | `--stateless-rpc` | `<repoPath>` | runtime | `git.go:190` |

### File writes
| Target | What is written | Purpose | `file:line` |
|---|---|---|---|
| CA cert temp file | PEM cert data | TLS certificate | `ca.go:70` |
| CA key temp file | PEM key data | TLS private key | `ca.go:77` |

## Notes / open questions

- The deny list is purely in-memory with no persistence. Server
  restart clears all revocations. The comment at `denylist.go:13`
  justifies this: cert TTLs are short and the CA rotates periodically.
- Rate limiters are never evicted from the `sync.Map`
  (`exec.go:196-197` comment). Each unique CN accumulates ~200 bytes.
  Acceptable for typical deployments but noted as a potential concern
  for thousands of unique certs.
- The git push authorization uses prefix matching on branch names
  (`refs/heads/polecat/<name>-*`). This means a polecat named `foo`
  can push to any branch starting with `polecat/foo-` but not to
  `polecat/foo` exactly (the trailing `-` is required). This prevents
  namespace collisions between polecats whose names are prefixes of
  each other.
- `handleExec` always returns HTTP 200 even for failed subprocesses.
  The rationale at `exec.go:118-124`: the RPC call succeeded; the
  subprocess outcome is in the JSON body. Clients must check `exitCode`.
- `serverListenIPs` cannot auto-detect external NAT IPs (they live on
  the router, not local interfaces). Operators must declare them via
  `ExtraSANIPs` in the config.
