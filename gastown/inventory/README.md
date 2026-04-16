---
title: gastown inventory
type: note
status: partial
topic: gastown
created: 2026-04-11
updated: 2026-04-11
sources:
  - /home/kimberly/repos/gastown/
tags: [inventory, enumeration, index]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# gastown inventory

Neutral, flat enumeration of everything in `/home/kimberly/repos/gastown/`.
No interpretation, no classification, no drift claims — just a
comprehensive list of what's in the repo with factual metadata (file
sizes, line counts, package file counts). Drives subsequent batches of
per-entity wiki pages.

## Pages

- [repo-root.md](repo-root.md) — every file and directory at the
  top level of the gastown repo, with size and immediate-child count.
- [docs-tree.md](docs-tree.md) — every file under `docs/` with line
  count and path.
- [go-packages.md](go-packages.md) — every Go package under `cmd/`,
  `internal/`, and `plugins/` with Go file and test file counts.
- [auxiliary.md](auxiliary.md) — the 9 auxiliary top-level directories
  (`scripts/`, `templates/`, `gt-model-eval/`, `npm-package/`,
  `.github/`, `.githooks/`, `.claude/`, `.opencode/`, `.runtime/`).
  Added in Batch 11.

## What is NOT here

- **Nested subcommand tree.** The 111 top-level commands are listed
  in [../commands/README.md](../commands/README.md). The ~384 nested
  subcommands that make up the remaining 495 − 111 ≈ 384 total
  `cobra.Command` definitions are a separate enumeration pass.
- **Per-file prose content.** This inventory lists files and counts
  only. Reading each file and writing an entity page with "what it
  actually does" is the next batch after enumeration.

## Method

Counts captured in one session on 2026-04-11 via
`ls`, `find`, and `wc -l` from inside `/home/kimberly/repos/gastown/`.
Reproducible commands are documented at the top of each inventory
page. The counts are a snapshot and will drift as upstream evolves.
