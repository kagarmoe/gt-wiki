---
title: internal/bitbucket
type: package
status: verified
topic: gastown
created: 2026-05-20
updated: 2026-05-20
sources:
  - /home/kimberly/repos/gastown/internal/bitbucket/client.go
  - /home/kimberly/repos/gastown/internal/bitbucket/pr.go
  - /home/kimberly/repos/gastown/internal/bitbucket/remote.go
tags: [package, integration, bitbucket, pull-request, merge-queue, http-client, post-v1.0.0]
detail_depth: {params: 2, data_flow: 2, errors: 1, side_effects: 1}
---

# internal/bitbucket

A Bitbucket Cloud REST API 2.0 client covering the pull-request lifecycle the
Gas Town merge queue needs: create draft PRs, read approval state and
comments, reply to comments, and merge. New in v1.0.1.

**Go package path:** `github.com/steveyegge/gastown/internal/bitbucket`
**File count:** 3 go files — `client.go` (125), `pr.go` (206), `remote.go` (30)
**Imports (notable):** stdlib only (`net/http`, `encoding/json`, `context`, `os`)
**Imported by:** [internal/refinery](refinery.md) only, via
`internal/refinery/pr_provider_bitbucket.go` — the Bitbucket implementation of
the refinery's PR-provider abstraction.
**Authentication:** `BITBUCKET_TOKEN` env var (API token or Repository Access
Token), sent as `Authorization: Bearer <token>`.

## What it actually does

### Client construction and transport (`client.go`)

- `bitbucket.NewClient(opts ...Option) (*Client, error)`
  (`client.go:50-64`) — builds a client defaulting to `http.DefaultClient`,
  `BITBUCKET_TOKEN` from the environment, and REST base
  `https://api.bitbucket.org/2.0` (`client.go:21`). Returns an error if no
  token is resolved. Functional options `WithHTTPClient`, `WithToken`,
  `WithRESTBase` (`client.go:34-48`) override these, primarily for testing.
- `(*Client).restRequest(ctx, method, path, body, result)`
  (`client.go:66-114`) — the single transport path. Marshals the request
  body, sets Bearer auth + JSON headers, executes, and on a non-2xx status
  returns an `*APIError`; otherwise decodes the JSON response into `result`.
- `APIError` (`client.go:116-125`) — captures method, path, status code, and
  raw body; its `Error()` renders all four.

### Pull-request operations (`pr.go`)

All methods are thin wrappers over `restRequest` against
`/repositories/{workspace}/{repoSlug}/...`:

- `CreateDraftPR(...)` (`pr.go:35-61`) — POSTs a PR with `"draft": true`,
  returning a `PRResult{ID, URL}` (`pr.go:29-32`).
- `UpdatePRDescription(...)` (`pr.go:63-72`) — PUT the description.
- `GetPRApprovalStatus(...) (ReviewState, error)` (`pr.go:74-111`) — GETs the
  participants list and reduces it to a `ReviewState` (`pr.go:9-16`:
  `PENDING` / `APPROVED` / `CHANGES_REQUESTED`). A reviewer in
  `changes_requested` short-circuits to `ReviewChangesRequired`; otherwise any
  approving reviewer yields `ReviewApproved`, else `ReviewPending`.
- `GetPRComments(...) ([]PRComment, error)` (`pr.go:113-157`) — GETs comments
  and flattens them into `PRComment` (`pr.go:18-27`), carrying inline
  path/line when present.
- `ReplyToPRComment(...)` (`pr.go:159-171`) — POSTs a threaded reply via the
  comment's `parent.id`.
- `MergePR(ctx, ..., strategy)` (`pr.go:173-186`) — POSTs a merge with
  `close_source_branch: true`; valid strategies `merge_commit`, `squash`,
  `fast_forward`.
- `GetRepoMergeStrategies(...) (string, error)` (`pr.go:188-205`) — reads the
  repo branching model and returns its development merge strategy, **defaulting
  to `squash`** if unset or unrecognised (and on request failure).

### Remote URL parsing (`remote.go`)

- `ParseBitbucketRemote(remoteURL) (workspace, repoSlug, err)`
  (`remote.go:11-29`) — extracts workspace + repo slug from HTTPS
  (`https://bitbucket.org/ws/repo.git`) or SSH
  (`git@bitbucket.org:ws/repo.git`) remotes; errors on non-Bitbucket URLs.

## Outgoing calls

All network egress is HTTPS to the Bitbucket Cloud REST API 2.0
(`https://api.bitbucket.org/2.0`, overridable via `WithRESTBase`):

| Operation | Method + path |
|---|---|
| Create draft PR | `POST /repositories/{ws}/{repo}/pullrequests` |
| Update description | `PUT  …/pullrequests/{id}` |
| Approval status | `GET  …/pullrequests/{id}` |
| Comments | `GET  …/pullrequests/{id}/comments` |
| Reply to comment | `POST …/pullrequests/{id}/comments` |
| Merge | `POST …/pullrequests/{id}/merge` |
| Merge strategy | `GET  …/branching-model` |

## Failure modes

- **No token:** `NewClient` fails fast with
  `"bitbucket: BITBUCKET_TOKEN is required (...)"` (`client.go:60-62`).
- **Non-2xx response:** surfaced as `*APIError` with the raw body
  (`client.go:104-111`), so callers can inspect status and message.
- **Missing branching model:** `GetRepoMergeStrategies` swallows the request
  error and falls back to `squash` (`pr.go:196-199`) — a non-configured repo
  is treated as merge-strategy `squash` rather than an error.

## Related wiki pages

- [internal/refinery](refinery.md) — sole consumer; the merge queue selects
  this client through its `PRProvider` abstraction
  (`pr_provider_bitbucket.go`).
- [go-packages inventory](../inventory/go-packages.md) — package enumeration.

## Notes / open questions

- New in v1.0.1; not yet reached by a Phase 3 drift audit (no `phase3_*`
  frontmatter). Created during the v1.0.0 → v1.1.0 release sync (Stage A).
- The client is REST-2.0 only; there is no GraphQL or webhook surface here.
- Approval reduction only counts participants with `role == "REVIEWER"`
  (`pr.go:97-106`); author or non-reviewer approvals are ignored.
