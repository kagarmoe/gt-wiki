---
title: internal/github
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-15
sources:
  - /home/kimberly/repos/gastown/internal/github/client.go
  - /home/kimberly/repos/gastown/internal/github/pr.go
tags: [package, github, api, rest, graphql, pr, merge-queue]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
---

# internal/github

Thin HTTP client for the GitHub REST and GraphQL APIs, scoped to the PR
lifecycle operations the Gas Town merge queue actually needs: create a
draft PR, update its description, flip it from draft to ready, fetch
review status + comments, reply to review comments, and merge. Also
detects the repo's preferred merge method. Authentication is a bearer
token read from `GITHUB_TOKEN` at construction time.

This package is **orthogonal** to [`internal/git`](git.md)'s `gh`-CLI
bridges (`HasOpenPR`, `GhPrMerge`, etc.). Those shell out to the `gh`
binary for quick probes; `internal/github` is the authenticated HTTP
client used on the hot path of the merge queue where subprocess fork
cost and `gh` dependency are undesirable.

**Go package path:** `github.com/steveyegge/gastown/internal/github`
**File count:** 2 go files (`client.go`, `pr.go`), 1 test file.
**Imports (notable):** stdlib only (`bytes`, `context`, `encoding/json`,
`fmt`, `io`, `net/http`, `os`). No external HTTP libraries.
**Imported by (notable):** [`internal/refinery`](refinery.md) and the
[`gt mq`](../commands/mq.md) / [`gt release`](../commands/release.md)
command paths.

## What it actually does

### Client plumbing (`client.go`)

- `Client` struct (`client.go:26-31`): `{httpClient, token, restBase,
  graphqlBase}`. Defaults: `http.DefaultClient`, `GITHUB_TOKEN` env
  var, `https://api.github.com`, `https://api.github.com/graphql`.
- `NewClient(opts ...Option)` (`client.go:58-72`) — returns an error
  if no token is resolved (either from env var or `WithToken(...)`).
  The four `Option`s are `WithHTTPClient`, `WithToken`, `WithRESTBase`,
  `WithGraphQLBase`. The last two exist for test injection.
- `restRequest(ctx, method, path, body, result)`
  (`client.go:75-123`) — marshal → authorize → execute → read →
  unmarshal. Adds three headers: `Authorization: Bearer <token>`,
  `Accept: application/vnd.github+json`,
  `X-GitHub-Api-Version: 2022-11-28`, plus `Content-Type:
  application/json` when there's a body. Non-2xx responses become
  `*APIError`.
- `graphqlRequest(ctx, query, variables, result)`
  (`client.go:126-177`) — wraps the request in `{"query", "variables"}`,
  posts to the GraphQL endpoint, decodes into a `graphQLResponse`
  holding raw `data` + typed `errors`. The first entry of `errors`
  short-circuits into a Go error; otherwise `data` is re-unmarshalled
  into `result`.
- `APIError` struct (`client.go:187-196`): `{Method, Path, StatusCode,
  Body}`. Error string is
  `"github: <METHOD> <path> returned <status>: <body>"`.

### PR operations (`pr.go`)

All methods are on `*Client` and take a `ctx` as first arg.

- `ReviewState` type with constants `ReviewPending`, `ReviewApproved`,
  `ReviewChangesRequired`, `ReviewCommented`, `ReviewDismissed`
  (`pr.go:9-17`).
- `ReviewComment` and `PRResult` structs (`pr.go:20-34`) — the PR
  creation return value is just `{Number, URL}`.
- `CreateDraftPR(ctx, owner, repo, head, base, title, body)`
  (`pr.go:38-55`) — POSTs `/repos/{o}/{r}/pulls` with `draft: true`.
  Comment explicitly notes that cross-repo (fork) PRs use
  `head = "forkOwner:branchName"` format.
- `UpdatePRDescription(ctx, owner, repo, prNumber, body)`
  (`pr.go:58-65`) — PATCH `/repos/{o}/{r}/pulls/{n}` with
  `{"body": body}`.
- `ConvertDraftToReady(ctx, owner, repo, prNumber)`
  (`pr.go:69-86`) — two-step: first `getPRNodeID` (REST GET for
  `node_id`), then the GraphQL `markPullRequestReadyForReview`
  mutation. REST doesn't expose this, hence the GraphQL hop.
- `GetPRReviewStatus(ctx, owner, repo, prNumber) (ReviewState, error)`
  (`pr.go:90-127`) — GETs `/pulls/{n}/reviews`, collapses to the
  latest review per reviewer, then returns:
  - `ReviewChangesRequired` if any latest review is
    `CHANGES_REQUESTED` (highest priority).
  - `ReviewApproved` if any is `APPROVED`.
  - Otherwise `ReviewPending`.
  Empty review list → `ReviewPending`.
- `GetPRReviewComments(ctx, owner, repo, prNumber)`
  (`pr.go:130-160`) — GET `/pulls/{n}/comments`, decodes into a
  parallel raw struct, then copies into `[]ReviewComment` flattening
  the nested `user.login` into a top-level `User` string.
- `ReplyToPRComment(ctx, owner, repo, prNumber, commentID, body)`
  (`pr.go:163-173`) — POST `/pulls/{n}/comments` with
  `{"body", "in_reply_to": commentID}`.
- `MergePR(ctx, owner, repo, prNumber, method)` (`pr.go:177-184`)
  — PUT `/pulls/{n}/merge` with `{"merge_method": method}`. Valid
  methods: `"merge"`, `"squash"`, `"rebase"`.
- `GetRepoMergeMethod(ctx, owner, repo)` (`pr.go:189-210`) — GET
  `/repos/{o}/{r}`, inspects `allow_squash_merge`, `allow_rebase_merge`,
  `allow_merge_commit`, and returns the preferred method in the order
  **squash > rebase > merge**. Errors out if none are enabled.
- `getPRNodeID(ctx, owner, repo, prNumber)` (`pr.go:213-225`) —
  internal helper for GraphQL mutations: GETs the PR's `node_id`.

## Related wiki pages

- [internal/refinery](refinery.md) — the merge-queue pipeline that
  drives this client.
- [internal/git](git.md) — sibling, subprocess-level git wrapper.
  Overlaps in purpose for PR existence probes; the git package's
  `HasOpenPR`/`GhPrMerge` shell out to `gh`, this package uses the
  HTTP API directly.
- [gt mq](../commands/mq.md), [gt release](../commands/release.md)
  — command-level consumers.
- [gt](../binaries/gt.md).
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- `GITHUB_TOKEN` is read once at `NewClient` time and cached on the
  struct; later env changes don't take effect. Rotate by building a
  new client.
- `restRequest` decodes responses into the passed `result` only when
  the response body is non-empty. DELETE-style endpoints that return
  204 with no body work correctly, but the function will silently leave
  `result` unmodified if a 200 happens to have an empty body.
- `GetPRReviewStatus`'s "latest per reviewer" logic assumes the API
  returns reviews in chronological order; the GitHub REST docs do
  guarantee this (ascending by submitted_at), so the last iteration
  wins. Worth keeping in mind if pagination is ever added — currently
  only the first page is fetched.
- `GetRepoMergeMethod` errors out hard if no merge methods are
  enabled. The merge queue caller should handle that as a permanent
  per-repo failure, not a transient one.
- The comma in `reqBody` in `ReplyToPRComment` (`pr.go:165-167`) uses
  the key `"in_reply_to"` — this matches the GitHub REST schema; a
  quick sanity check for future maintainers who may reach for the
  newer `in_reply_to_id` field name.
