---
title: gt-proxy-client
type: binary
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/cmd/gt-proxy-client/main.go
  - /home/kimberly/repos/gastown/internal/proxy/server.go
  - /home/kimberly/repos/gastown/internal/proxy/exec.go
  - /home/kimberly/repos/gastown/Makefile
tags: [binary, proxy, mtls, sandbox, client, shim]
---

# gt-proxy-client

A drop-in shim installed inside polecat containers as both `gt` and `bd`.
When proxy environment variables are set, it forwards `os.Args[1:]` to
[gt-proxy-server](gt-proxy-server.md) over mTLS and proxies the response.
Otherwise it `execve`s the real `gt`/`bd` binary and disappears.
One of the three binaries produced by [`make build`](../files/makefile.md).

**Also known as:** `gt-proxy-client` (binary on disk, produced by
`./cmd/gt-proxy-client`), `gt` / `bd` (names it is installed as inside
polecat containers via symlink or copy — the binary inspects `argv[0]`
to decide which tool it is).

## What it actually does

### Entry point

`/home/kimberly/repos/gastown/cmd/gt-proxy-client/main.go` is 137 lines
and is the **entire** binary — there is no accompanying package code.
Unlike `cmd/gt/main.go` (a 12-line stub that delegates to
`internal/cmd/`), this is a self-contained one-file program.

Package-level file comment (`cmd/gt-proxy-client/main.go:1-4`):

> gt-proxy-client is the pass-through binary installed in containers as
> both `gt` and `bd`. When `GT_PROXY_URL`, `GT_PROXY_CERT`,
> `GT_PROXY_KEY`, and `GT_PROXY_CA` are all set, it forwards
> `os.Args[1:]` to the proxy server over mTLS and proxies the response.
> Otherwise it execs the real binary at `/usr/local/bin/gt.real` (or the
> path in `GT_REAL_BIN`).

### Environment variable contract

`main.go:31-51` — four env vars are required for proxy mode:

| Env var          | Purpose                                                              |
|------------------|----------------------------------------------------------------------|
| `GT_PROXY_URL`   | Proxy base URL, e.g. `https://172.17.0.1:9876`                       |
| `GT_PROXY_CERT`  | Path to PEM client cert (issued by the proxy's CA)                   |
| `GT_PROXY_KEY`   | Path to PEM client private key                                       |
| `GT_PROXY_CA`    | Path to PEM proxy CA cert, used to verify the server TLS cert        |

Optional:

| Env var        | Default                   | Purpose                                     |
|----------------|---------------------------|---------------------------------------------|
| `GT_REAL_BIN`  | `/usr/local/bin/gt.real`  | Fallback binary used when proxy vars unset  |

If any of the four required vars is unset, the binary silently calls
`execReal()` (`main.go:47-51`) — it is a pure pass-through in that mode,
replacing its own process with the real `gt`/`bd` via `syscall.Exec`
(`main.go:127-136`).

The comment at `main.go:42-45` notes that `GT_PROXY_CA` is the same
CA cert as `GIT_SSL_CAINFO` (which git uses to trust the proxy's
server cert), but is passed separately so the Go HTTP client can also
verify the server cert.

### Proxy-mode behavior

When all four env vars are set, `main.go:53-118`:

1. **Build mTLS client**: load the client cert+key via
   `tls.LoadX509KeyPair` (`main.go:54-58`), read the CA PEM and build a
   cert pool (`main.go:60-69`), construct a `tls.Config` with the client
   cert as identity and the pool as `RootCAs` (`main.go:71-74`), wrap
   in an `http.Client` with a 5-minute timeout (`main.go:76-79`).
2. **Determine the tool name from `argv[0]`**
   (`main.go:81-85`): `toolName := filepath.Base(os.Args[0])`. This is
   what lets a single binary installed as both `gt` and `bd` serve
   both — the server's subcommand allowlist keys on `argv[0]`, so the
   name matters.
3. **Build the request**: rewrite `argv[0]` to the tool name,
   `json.Marshal({argv: [...]})` (`main.go:85-91`).
4. **`POST $GT_PROXY_URL/v1/exec`** with `application/json` body
   (`main.go:93`). See the matching server handler at
   `/home/kimberly/repos/gastown/internal/proxy/exec.go:26`.
5. **Error handling**: network failure → exit 1 with message on stderr.
   Non-200 HTTP → exit 1 with server body on stderr
   (`main.go:94-104`).
6. **Decode the `execResponse` JSON body** (`main.go:106-110`):
   `{stdout, stderr, exitCode}`.
7. **Replay the response to local tty**: write `result.Stdout` to
   os.Stdout, `result.Stderr` to os.Stderr (`main.go:112-117`).
8. **Exit with the subprocess's exit code**: `os.Exit(result.ExitCode)`
   (`main.go:118`). From the caller's perspective this is
   indistinguishable from a local exec.

### Wire protocol

`cmd/gt-proxy-client/main.go:21-29` defines the request and response
shapes, which exactly match the server's types at
`/home/kimberly/repos/gastown/internal/proxy/exec.go:14-24`:

```go
type execRequest struct {
    Argv []string `json:"argv"`
}

type execResponse struct {
    Stdout   string `json:"stdout"`
    Stderr   string `json:"stderr"`
    ExitCode int    `json:"exitCode"`
}
```

Stdout and stderr are returned as complete strings (not streamed).
Subprocess exit code is carried in the JSON body — the HTTP status is
always 200 for successful RPC, regardless of subprocess exit.
See [gt-proxy-server](gt-proxy-server.md) for the matching handler
semantics.

### Dependencies

`cmd/gt-proxy-client/main.go:7-19` — stdlib only:

```go
"bytes", "crypto/tls", "crypto/x509", "encoding/json", "errors",
"fmt", "io", "net/http", "os", "path/filepath", "syscall", "time"
```

**No internal imports.** No `internal/proxy`, no `internal/cmd`, no
`internal/util`. This binary is intentionally self-contained so it can
be a thin tool shipped in minimal container images without dragging in
the rest of gastown's Go dependency tree.

### Build

Produced by `make build` via this line in
[../files/makefile.md](../files/makefile.md) (`Makefile:35-38`):

```make
go build -ldflags "$(LDFLAGS)" -o $(BUILD_DIR)/$(BINARY)-proxy-client ./cmd/gt-proxy-client
```

Like [gt-proxy-server](gt-proxy-server.md), this binary does NOT import
`internal/cmd`, so the `BuiltProperly` self-kill gate in
`/home/kimberly/repos/gastown/internal/cmd/root.go:94-107` never runs
here — the `-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1`
ldflag is silently a no-op for this binary. `go build ./cmd/gt-proxy-client`
works without `make build`.

### Runtime relationship

Short-lived; runs once per `gt` or `bd` invocation inside a polecat
container:

1. Container image install places this binary at the canonical `gt`
   and `bd` locations (e.g. `/usr/local/bin/gt`, `/usr/local/bin/bd`).
2. The real `gt`/`bd` (if any) is moved aside to `/usr/local/bin/gt.real`
   or whatever path `GT_REAL_BIN` points to.
3. On each invocation, the binary checks the four proxy env vars and
   either forwards the call over mTLS or `syscall.Exec`s the real
   binary.
4. If it forwards, it prints the server's captured stdout/stderr
   locally and exits with the same code as the remote subprocess.

The pairing is: one [gt-proxy-server](gt-proxy-server.md) on the host,
and one `gt-proxy-client` inside each polecat container. The server
terminates mTLS, enforces allowlists, and runs the real `gt`/`bd` on
the host's filesystem. The client is purely a transport shim.

## Cross-references

- [gt](gt.md) — the main CLI binary. Sibling under `cmd/`. Note that
  `gt-proxy-client` is **the binary that gets installed as `gt`**
  inside polecat containers, intercepting every `gt` call so it can
  be routed through the proxy. The "real" `gt` is moved to
  `gt.real` (or `$GT_REAL_BIN`).
- [gt-proxy-server](gt-proxy-server.md) — the host-side counterpart
  this client talks to over mTLS.
- [Makefile](../files/makefile.md) — canonical build recipe; the
  `build` target produces this binary at `./gt-proxy-client`.
- [Dockerfile](../files/dockerfile.md) — the container image build.
- [../inventory/go-packages.md](../inventory/go-packages.md) —
  enumerates `cmd/gt-proxy-client` as 1 Go file + 0 tests.
- [../inventory/repo-root.md](../inventory/repo-root.md) — lists
  `cmd/` as holding the three built binaries.

The wire-protocol peer (`internal/proxy/exec.go`) is part of the
`internal/proxy/` package, not yet mapped as a wiki entity.

## Notes / open questions

- This binary has no test file (inventory says 1 Go file + 0 tests at
  `/home/kimberly/repos/gt-wiki/gastown/inventory/go-packages.md:41`).
  All test coverage of the client/server exchange must live in
  `internal/proxy/e2e_test.go` or the `cmd/gt-proxy-server/config_test.go`
  — not in `cmd/gt-proxy-client/`.
- How does the container image arrange for this binary to appear as
  `gt` AND `bd`? Two copies? Two symlinks? Out of scope for this layer
  — check the Dockerfile mapping or container build script.
- How do `GT_PROXY_CERT` / `GT_PROXY_KEY` end up inside the container?
  Presumably issued via the proxy's admin `POST /v1/admin/issue-cert`
  endpoint (`internal/proxy/server.go:414`) and mounted into the
  container at launch by the polecat supervisor. Not mapped yet.
- The HTTP client timeout is a hardcoded 5 minutes (`main.go:77`) — no
  env var to override. This caps any single proxied `gt`/`bd` command
  at 5 minutes end-to-end, which is tighter than the server's
  `ExecTimeout` default of 60s but looser than the server's
  `WriteTimeout` of 5 minutes (also hardcoded at
  `/home/kimberly/repos/gastown/internal/proxy/server.go:242`). These
  three timeouts are not obviously coordinated.
- Stdout/stderr are returned as complete strings, not streamed. Any
  long-running `gt` or `bd` subcommand (e.g. interactive TUI, progress
  bars, tail output) will buffer until exit. This fundamentally shapes
  what commands can usefully be called through the proxy — hence the
  conservative default allowlist
  (`cmd/gt-proxy-server/main.go:24-26`).
- `GT_REAL_BIN` default is `/usr/local/bin/gt.real`. The same binary is
  also installed as `bd`, but the fallback path is hardcoded to
  `gt.real` — when invoked as `bd`, an unset-env fallback would `exec`
  `gt.real` rather than `bd.real`. Whether this matters depends on
  whether the outer `bd.real` binary is even needed in sandboxed
  containers.
- Out of scope for this layer: `docs/proxy-server.md` comparison —
  deferred to phase 3.
