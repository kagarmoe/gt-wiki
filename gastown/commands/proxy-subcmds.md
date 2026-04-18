---
title: gt proxy-subcmds
type: command
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-16
sources:
  - /home/kimberly/repos/gastown/internal/cmd/proxy_subcmds.go
tags: [command, ungrouped, hidden, polecat-safe, proxy, allowlist]
phase3_audited: 2026-04-15
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase5_audience: internal
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# gt proxy-subcmds

Hidden command that prints the allowlist of gt/bd subcommands polecats
may invoke through the mTLS proxy. Also the source file that defines
the `AnnotationPolecatSafe` constant consumed by every polecat-safe
command in the tree.

**Parent:** [gt](../binaries/gt.md) (root command)
**Group:** (none — no `GroupID` set on the cobra.Command definition)
**Hidden:** yes (`Hidden: true` on `proxy_subcmds.go:24`)
**Polecat-safe:** yes (its output *is* the polecat-safe list; also
itself would be counted if it set the annotation, which it does not —
so polecats cannot call this command through the proxy, only
`gt-proxy-server` reads it out-of-band at startup)
**Beads-exempt:** no
**Branch-check-exempt:** no

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/cmd/proxy_subcmds.go:22-49`.

### Invocation

```
gt proxy-subcmds
```

`Use: "proxy-subcmds"` (`proxy_subcmds.go:23`). No positional args, no
flags. Produces one line of output and exits.

### Output format

A single semicolon-separated line:

```
gt:<sorted,comma,separated>;bd:<fixed,comma,separated>
```

The run function at `proxy_subcmds.go:35-44`:

```go
Run: func(cmd *cobra.Command, args []string) {
    var gtSubs []string
    for _, c := range rootCmd.Commands() {
        if c.Annotations[AnnotationPolecatSafe] == "true" {
            gtSubs = append(gtSubs, c.Name())
        }
    }
    sort.Strings(gtSubs)
    fmt.Printf("gt:%s;bd:%s\n", strings.Join(gtSubs, ","), bdSafeSubcmds)
},
```

### `AnnotationPolecatSafe` (`proxy_subcmds.go:11-15`)

```go
const AnnotationPolecatSafe = "polecatSafe"
```

The comment on `proxy_subcmds.go:11-14` is the canonical spec:

> AnnotationPolecatSafe marks a cobra command as safe for polecat
> sandbox execution. Commands with this annotation are included in the
> output of "gt proxy-subcmds". Add `Annotations:
> map[string]string{AnnotationPolecatSafe: "true"}` to any gt
> subcommand that polecats should be permitted to run through the
> proxy.

This constant is imported by every command that sets the annotation —
e.g. [hook](hook.md) at `hook.go:26`, [version](version.md) at
`version.go:34`. It is not exported through a package — it lives in
`package cmd` alongside every command that uses it.

### `bdSafeSubcmds` (`proxy_subcmds.go:17-20`)

```go
const bdSafeSubcmds = "create,update,close,show,list,ready,dep,export,prime,stats,blocked,doctor"
```

Unlike the `gt` half (auto-discovered via annotations), the `bd` half
is a fixed hard-coded list. `bd` commands are maintained in a separate
binary that does not embed cobra annotations, so the allowlist has to
live here.

### Subcommands

None.

### Flags

None.

## Who calls it

Per the `Long` help on `proxy_subcmds.go:26-34`:

> Prints a semicolon-separated "cmd:sub1,sub2,..." string listing which
> subcommands polecats may invoke through the mTLS proxy. The gt
> portion is discovered automatically by scanning commands annotated
> with the polecatSafe annotation; the bd portion is a fixed list
> embedded here.
>
> The proxy server calls this command at startup and falls back to its
> built-in default if discovery fails.

The consumer is [../binaries/gt-proxy-server.md](../binaries/gt-proxy-server.md),
which references this command via its `discoverAllowedSubcmds` helper
(per the batch brief). The proxy server runs `gt proxy-subcmds`, parses
the output, and uses it to build its mTLS request allowlist. On
discovery failure (missing binary, parse error, timeout), it falls back
to a compiled-in default.

## Related commands

- [../binaries/gt-proxy-server.md](../binaries/gt-proxy-server.md) —
  the caller. Reads this command's stdout at startup.
- [../binaries/gt-proxy-client.md](../binaries/gt-proxy-client.md) —
  the client side of the mTLS channel; subject to the allowlist this
  command emits.
- [polecat](polecat.md) — the role that runs under the allowlist.
- [hook](hook.md), [version](version.md), [status](status.md),
  [prime](prime.md), [nudge](nudge.md), [mail](mail.md),
  [handoff](handoff.md), [done](done.md), [convoy](convoy.md),
  [sling](sling.md), [mountain](mountain.md), [molecule](molecule.md) —
  the current polecat-safe set (per the batch brief). Each sets
  `Annotations: map[string]string{AnnotationPolecatSafe: "true"}` in
  its own init.

## Failure modes

No failure modes discovered. Registers proxy subcommands for `bd` passthrough. Pure command registration — no runtime error paths.

## Notes / open questions

- **Fragile allowlist discovery.** If `gt proxy-subcmds` crashes or
  times out, the proxy server silently falls back to a built-in
  default that may lag the source tree. Worth flagging in the
  `gt-proxy-server` page.
- **Command is not itself polecat-safe.** It has `Hidden: true` but
  lacks the `AnnotationPolecatSafe` annotation — which is correct
  (polecats should not be able to enumerate the allowlist through the
  proxy), but means `gt-proxy-server` must invoke this command
  locally (not through the proxy) to discover the allowlist.
- **bd list can drift.** The `bd` half is hard-coded. If the `bd`
  binary grows or drops subcommands, this string needs a manual
  update. No test enforces parity.
- **Cobra iteration order.** `rootCmd.Commands()` returns in
  registration order; `sort.Strings(gtSubs)` canonicalizes the `gt`
  half. The `bd` half order is preserved as written. Consumers should
  tokenize on both `;` and `,`.
