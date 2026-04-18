---
title: flake.nix
type: file
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/flake.nix
  - /home/kimberly/repos/gastown/flake.lock
tags: [file, nix, flake, build, devshell]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# flake.nix

Nix flake definition for building and developing the gastown project
on systems with Nix installed. Lives at
`/home/kimberly/repos/gastown/flake.nix`. Alternative build path to
[Makefile](makefile.md), [Dockerfile](dockerfile.md), and
[.goreleaser.yml](goreleaser-yml.md) — each produces a
[gt](../binaries/gt.md) binary via a different route with different
ldflag semantics.

Companion file: `/home/kimberly/repos/gastown/flake.lock`. Not
transcribed as its own page; pinned input SHAs are captured in the
"Locked inputs" section below.

## What it actually does

### Description

`flake.nix:2`:

```nix
description = "Multi-agent orchestration system for Claude Code with persistent work tracking";
```

### Inputs

`flake.nix:4-8`:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  flake-utils.url = "github:numtide/flake-utils";
  beads.url = "github:steveyegge/beads";
};
```

Three inputs, none pinned to a specific tag in the flake itself —
`flake.lock` provides the SHAs.

### Outputs function

`flake.nix:10-17` — opens with:

```nix
outputs = { self, nixpkgs, flake-utils, beads }:
  flake-utils.lib.eachDefaultSystem (system: ...)
```

`eachDefaultSystem` generates per-system outputs (typically
`x86_64-linux`, `aarch64-linux`, `x86_64-darwin`, `aarch64-darwin`).

### Go overlay (upstream-version workaround)

`flake.nix:20-35` — a `let`-bound overlay that bumps Go to 1.25.8:

```nix
goOverlay = final: prev: {
  go_1_25 = prev.go_1_25.overrideAttrs {
    version = "1.25.8";
    src = prev.fetchurl {
      url = "https://go.dev/dl/go1.25.8.src.tar.gz";
      hash = "sha256-6YjUokRqx/4/baoImljpk2pSo4E1Wt7ByJgyMKjWxZ4=";
    };
  };
};
```

Comment at `flake.nix:20-21` explains the reason:

> Go 1.25.8: beads v0.60.0 deps (charm.land/huh/v2) require it,
> nixpkgs has 1.25.7. Remove when nixpkgs ships Go >= 1.25.8

Note mismatch: the comment says "beads v0.60.0 deps" but
[go.mod](go-mod.md) at line 18 requires `github.com/steveyegge/beads
v0.63.3`. The workaround was added for v0.60.0 and hasn't been
removed.

`pkgs` is instantiated with this overlay at `flake.nix:32-35`, and
`beadsPkg = beads.packages.${system}.default` at `flake.nix:36`.

### Packages output

`flake.nix:39-61`:

```nix
packages = {
  gt = pkgs.buildGoModule {
    pname = "gt";
    version = "0.8.0";
    src = ./.;
    vendorHash = "sha256-XWv/slFm796AO928eqzVHms0uUX4ZMJk0I4mZz+kp54=";

    ldflags = [
      "-X github.com/steveyegge/gastown/internal/cmd.Build=nix"
      "-X github.com/steveyegge/gastown/internal/cmd.BuiltProperly=1"
    ];

    subPackages = [ "cmd/gt" ];

    meta = with pkgs.lib; {
      description = "Multi-agent orchestration system for Claude Code with persistent work tracking";
      homepage = "https://github.com/steveyegge/gastown";
      license = licenses.mit;
      mainProgram = "gt";
    };
  };
  default = self.packages.${system}.gt;
};
```

Key points:

- **Hardcoded `version = "0.8.0"`** — the Nix package version doesn't
  track git tags.
- **`subPackages = [ "cmd/gt" ]`** — Nix builds only `gt`, not the
  proxy binaries that [Makefile](makefile.md):35-38 produces.
- **LDFLAGS** set only two variables:
  - `Build=nix`
  - `BuiltProperly=1`

  Both bypass the self-kill check at
  `/home/kimberly/repos/gastown/internal/cmd/root.go:96-106`.
  `Build=nix` would work alone (since `Build != "dev"` satisfies the
  gate per [../binaries/gt.md](../binaries/gt.md)); `BuiltProperly=1`
  is redundant but harmless. `Version`, `Commit`, `Branch` are left as
  their source defaults.
- **`vendorHash`** pins a cryptographic hash of the vendored module
  set. Must be updated whenever [go.mod](go-mod.md) or `go.sum` changes.
- **`default = self.packages.${system}.gt`** — `nix build` with no
  argument builds `gt`.

### Apps output

`flake.nix:63-68`:

```nix
apps = {
  gt = flake-utils.lib.mkApp {
    drv = self.packages.${system}.gt;
  };
  default = self.apps.${system}.gt;
};
```

Makes `gt` runnable via `nix run github:...`. `default = gt`, so
`nix run` with no app name runs `gt`.

### Dev shell

`flake.nix:70-78`:

```nix
devShells.default = pkgs.mkShell {
  buildInputs = [
    beadsPkg
    pkgs.go_1_25
    pkgs.gopls
    pkgs.gotools
    pkgs.go-tools
  ];
};
```

`nix develop` drops you into a shell with:

- `beads` (the daemon, from the `beads` flake input)
- Go 1.25.8 (via the overlay above)
- `gopls` — the Go language server
- `gotools` — `gorename`, `guru`, etc. (the upstream `golang.org/x/tools`
  binaries)
- `go-tools` — the `honnef.co/go/tools` suite including `staticcheck`

No `golangci-lint` in the dev shell (see [.golangci.yml](golangci-yml.md)
for the project's lint configuration — users need to install it
separately).

### No `checks` output

The flake does not define a `checks` output, so `nix flake check` has
nothing to verify beyond the package building successfully.

## Locked inputs

From `/home/kimberly/repos/gastown/flake.lock`:

| input          | owner/repo                 | ref                | locked SHA                                 | lastModified |
|----------------|----------------------------|--------------------|--------------------------------------------|--------------|
| `nixpkgs`      | `NixOS/nixpkgs`            | `nixos-unstable`   | `2fc6539b481e1d2569f25f8799236694180c0993` | 1771848320   |
| `flake-utils`  | `numtide/flake-utils`      | (default branch)   | `11707dc2f618dd54ca8739b309ec4fc024de578b` | 1731533236   |
| `beads`        | `steveyegge/beads`         | (default branch)   | `1ce765eef881a2a7efcc005c789729242da19f8b` | 1773453770   |
| `systems`      | `nix-systems/default`      | (default branch)   | `da67096a3b9bf56a91d16901293e51ba5b49a27e` | 1681028828   |

Secondary node: `nixpkgs_2` and an intermediate `nixpkgs` entry (both
from `NixOS/nixpkgs`) show the lockfile's nested input graph — `beads`
pulls its own `nixpkgs` at `nixos-25.11` (rev
`c581273b8d5bdf1c6ce7e0a54da9841e6a763913`), separate from the top-level
`nixos-unstable` pin.

## Cross-references

- [../binaries/gt.md](../binaries/gt.md) — the binary this flake
  builds; the `Build=nix` ldflag is one of the gate-bypass paths
  documented there.
- [Makefile](makefile.md) — sibling build path. Produces all three
  binaries ([gt](../binaries/gt.md),
  [gt-proxy-server](../binaries/gt-proxy-server.md),
  [gt-proxy-client](../binaries/gt-proxy-client.md)); flake
  produces only `gt`.
- [Dockerfile](dockerfile.md) — sibling build path via container.
- [.goreleaser.yml](goreleaser-yml.md) — sibling build path for
  release artifacts.
- [.golangci.yml](golangci-yml.md) — lint config not wired into the
  dev shell.
- [go.mod](go-mod.md) — the `vendorHash` in this file is derived from
  the module set declared there.
- [../inventory/repo-root.md](../inventory/repo-root.md) — inventory
  row for this file (and `flake.lock`).

## Notes / open questions

- `version = "0.8.0"` is hardcoded. The Nix package does not track
  `git describe` or `.goreleaser.yml`'s version template. A Nix user
  running `gt version` sees `"0.8.0"` regardless of the actual
  checkout.
- The Go overlay comment references "beads v0.60.0" but `go.mod`
  requires `v0.63.3`. Stale comment — the overlay may or may not
  still be necessary depending on what nixpkgs currently ships.
- `vendorHash` must be updated by hand when `go.mod` changes. If it
  drifts, `nix build` fails with a hash mismatch. There's no CI
  check in the flake to catch this.
- `golangci-lint` is not in the dev shell. Users who `nix develop`
  and then try to run lint need to install it externally.
- `flake.lock` pins a nested `nixpkgs` for beads (at `nixos-25.11`)
  separate from the top-level `nixos-unstable` — normal flake input
  graph behavior but worth noting.
- No Darwin-specific handling visible in the flake (unlike
  [Makefile](makefile.md):27-33's ICU detection). Either
  `buildGoModule` handles the ICU dependency transparently under Nix,
  or Darwin nix builds currently don't work and this hasn't been
  exercised.
