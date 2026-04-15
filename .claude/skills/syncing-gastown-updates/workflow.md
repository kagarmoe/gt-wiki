# Release sync workflow (detailed runbook)

This is the step-by-step companion to `SKILL.md`. Read the SKILL first for the rules; this file has the exact commands, templates, and worked examples. The 11 steps below are numbered to match the SKILL's workflow section.

All commands assume working directory `~/repos/gt-wiki`. Commands touching gastown source read from `~/repos/gastown` — **read-only; never write**.

---

## Step 1: Establish the delta (tag-to-tag)

**Goal:** know exactly what has changed upstream between the previously-documented release tag and the new release tag. The wiki tracks tagged releases only — `HEAD@{1}` and "commits since date X" are **wrong** diff bases for this skill.

### Find the new release tag

```bash
cd ~/repos/gastown
git fetch --tags
git tag --sort=-creatordate | head -10           # newest 10 tags
git log --format="%H %cI %s" v<new-tag>           # confirm the tag points where you expect
```

Confirm with Kimberly that the tag you identified is the release to sync to (pre-release / RC tags don't count; see SKILL's "When to use").

### Find the previously-documented release tag

**Preferred path (once the release marker exists):**

```bash
cd ~/repos/gt-wiki
grep "gastown_release" gastown/README.md         # reads the frontmatter or body marker
```

The marker tells you the tag the wiki is currently synced to. Use that as `<prev-tag>`.

**First-run path (no marker yet — see SKILL's "First-run caveat"):**

The wiki was mapped mid-release-cycle during Phase 2 (2026-04-11). There is no release tag corresponding to the baseline. Recover it manually:

```bash
cd ~/repos/gt-wiki
grep "^## \[" log.md | grep ingest | tail -5     # last 5 ingest entries
# Phase 2 Batch 12 (the final mapping batch) was 2026-04-11.
# Find the gastown commit that was current on that date:
cd ~/repos/gastown
git log --until="2026-04-11 23:59" --format="%H %cI" -1
```

Use that SHA as `<prev-base>`. Note explicitly in the sync's scope proposal that this is an untagged base — an artifact of Phase 2 running mid-cycle — and that establishing the release marker is a deliverable of this first sync.

### Run the tag-to-tag diff

```bash
cd ~/repos/gastown
git log --oneline <prev-tag>..v<new-tag>          # commits between releases
git log --format="%H %cI %s" v<new-tag>^..v<new-tag>   # the new release's own commit metadata
git diff --stat <prev-tag>..v<new-tag>            # files touched + insertions/deletions
git diff --name-status <prev-tag>..v<new-tag>     # A/M/D/R status per file
```

On the first-run path, substitute `<prev-base>` for `<prev-tag>` and the newly-identified release tag for `v<new-tag>`.

### Capture

- New release tag and its commit SHA (record in the log entry).
- Previous release tag (or untagged base SHA on first run).
- List of A (added), M (modified), D (deleted), R (renamed) files.
- Release notes if available: `git show v<new-tag>` plus the upstream `CHANGELOG.md` entry for the release if one exists.
- Commit messages scanning for keywords: "rename", "remove", "refactor", "breaking", "deprecate", "new command".

### Sanity checks

- The new tag is actually newer than the previous tag (`git log <prev-tag>..v<new-tag>` produces commits, not "no commits").
- You are not syncing against an untagged HEAD. If `git describe --tags HEAD` does not return a bare tag (`v<new-tag>` with no `-N-g<sha>` suffix), the branch is ahead of the tag and you should `git checkout v<new-tag>` before any source reads to avoid capturing between-release drift.

---

## Step 2: Read the active plan and recent log

**Goal:** know which phase is active and what the most recent decisions constrain.

```bash
cd ~/repos/gt-wiki
ls -la .claude/plans/
cat .claude/plans/$(ls -t .claude/plans/ | head -1)   # most recent plan
grep "^## \[" log.md | tail -10                        # last 10 log timeline entries
```

Read fully:
- The active plan file end-to-end. Note the current batch, the phase's discipline rules, the drift taxonomy if Phase 3+.
- The last 5 `log.md` decision entries (`grep "decision |"` for all decisions, then read the latest 5).
- Any `drift-found` or `ingest` entries from the last 7 days.

**Output of this step:** a one-paragraph understanding of "where the wiki is right now" — written down in your working memory, not yet in a file.

---

## Step 3: Triage the diff into buckets

**Goal:** classify every changed file into one of four buckets so the next steps know what pattern to apply.

| Bucket | Definition | Next step |
|--------|------------|-----------|
| New entity | File is `A` (added) AND represents a new binary / command / package / file / role / concept per the wiki's load-bearing test in CLAUDE.md | Step 8 (create new page) |
| Modified entity | File is `M` (modified) AND the modification changes observable behavior cited on an existing wiki page | Step 7 (refresh page) |
| Removed entity | File is `D` (deleted) OR `R` (renamed) AND has a wiki page | Step 9 (mark vestigial or rename) |
| Internal refactor | File is `M` but the change is non-semantic (formatting, comment, test-only) OR the file has no wiki page and isn't load-bearing | No wiki action; note in the log entry |

**How to classify "observable behavior changes":**

Read the diff of the file (`git show <commit> -- <path>`). If the changed lines include any of:
- function signatures (exported or cited)
- cobra `Long` help text
- error messages
- constants used in citations
- struct fields
- `AnnotationPolecatSafe` / beads-exempt registrations

...the modification is observable and needs a Step 7 refresh. If the changes are only internal helpers or tests or comments, it's a refactor (no wiki action).

**Output:** a classification table. Example shape:

```markdown
| Bucket | File | Wiki page(s) affected |
|--------|------|-----------------------|
| New entity | internal/cmd/swarm.go | (none — to create: gastown/commands/swarm.md) |
| Modified entity | internal/cmd/polecat.go | gastown/commands/polecat.md, gastown/roles/polecat.md |
| Modified entity | internal/daemon/daemon.go | gastown/packages/daemon.md |
| Removed entity | internal/cmd/legacy_sling.go | (none — file had no wiki page) |
| Internal refactor | internal/util/atomic.go | none |
```

---

## Step 4: Cross-reference diff against existing wiki

**Goal:** find every wiki page that cites a changed file, even when the page is for a different entity.

```bash
cd ~/repos/gt-wiki

# For each changed file between the two release tags, find wiki pages that cite it
for path in $(cd ~/repos/gastown && git diff --name-only <prev-tag>..v<new-tag>); do
  echo "=== $path ==="
  grep -rln "$path" gastown/ 2>/dev/null
done
```

This catches transitive references — e.g., `gastown/packages/beads.md` may cite `internal/cmd/sling.go:123` as an example of native-SDK dispatch; if `sling.go` changed, the `beads.md` page also needs a review.

Add every page found to the Step 3 classification table.

---

## Step 5: Check open beads

**Goal:** the release may have resolved or invalidated existing work items.

```bash
cd ~/repos/gt-wiki
bd list --status=open
bd list -l wants-wiki-entry
bd list -l drift
bd list -l wiki-investigation
```

For each open bead:
- Does the release resolve it (e.g., a drift finding was fixed upstream)? → flag for closing in the sync batch.
- Does the release change its scope (e.g., a `wants-wiki-entry` for a feature that's now been renamed)? → flag for updating in the sync batch.
- Is it still valid? → leave it alone.

---

## Step 6: Propose scope to Kimberly BEFORE touching files

**Goal:** phase-discipline gate. Release sync is not autonomous — Kimberly approves the batch list before work begins.

Present to Kimberly:

```markdown
## Release sync scope proposal

**Gastown delta:** `v<new-tag>` (SHA `<new-sha>`) vs `<prev-tag>` (SHA `<prev-sha>`). N commits between releases, M files changed.

**Active phase:** <phase from active plan file>.

**Classification:**
- N new entities (list)
- M modified entities (list with affected wiki pages)
- K removed entities (list with pages to mark vestigial)
- L internal refactors (no wiki action)

**Transitive wiki pages cited:** <list, deduplicated with Step 3 output>

**Beads affected:**
- <bead-id> — <resolution or scope change>
- ...

**Proposed batches** (one per commit, in order):
1. Batch A: <cluster> — <rationale>
2. Batch B: <cluster> — <rationale>
...

**Drift findings the release may resolve (Phase 3+ only):**
- <wiki page> — <current finding> — <proposed resolution>

**Wiki-stale findings discovered in cross-reference:**
- <wiki page> — <stale citation> — <proposed fix>

**Plan-file delta:**
- [fold into active phase plan as ad-hoc batch] OR [new release-sync plan file at .claude/plans/YYYY-MM-DD-release-sync-v<N>.md]

Awaiting approval before touching files.
```

Wait for Kimberly's verdict. Do NOT proceed to Step 7 without explicit approval.

---

## Step 7: Per affected page — refresh source, citations, frontmatter, drift

For each page in the "Modified entity" bucket:

### 7.1 Re-read source at the new release tag

First, make sure `~/repos/gastown` is checked out at the new release tag (not an arbitrary HEAD):

```bash
cd ~/repos/gastown
git describe --tags HEAD                              # must equal v<new-tag> exactly
# If not, check out the tag before reading:
# git checkout v<new-tag>
```

Then read the file in full:

```bash
cat ~/repos/gastown/<cited-path>                      # full file, no cheats
grep -n "<symbol-to-verify>" ~/repos/gastown/<cited-path>
```

**Load-bearing rule:** you must read the file at the new release tag, not at an arbitrary HEAD. Do NOT substitute "grep for the symbol" for "read the file" — citations depend on context, and grep misses renames and relocations.

### 7.2 Update `file:line` citations in the wiki page

Open `gastown/<page>.md`. For every citation that points to the re-read file:
- If the line number is still accurate → leave unchanged.
- If the line number moved → update to the new line.
- If the symbol was renamed → update the cited identifier AND the line number, and note the rename in the Notes / open questions section if it's notable.
- If the symbol is gone → this is a wiki-stale finding. Fix the surrounding page body to match current behavior (rewrite the claim, cite the new source). Log under `lint` verb.

### 7.3 Update frontmatter

```yaml
updated: 2026-04-14              # always bump
sources:                          # add new files if the page now cites them
  - /home/kimberly/repos/gastown/<path>
  - ...
phase3_audited: 2026-04-14        # Phase 3+ only, when the audit re-verified against the new release tag
phase3_findings: [...]            # Phase 3+ only, update the category tag list if findings changed
phase3_severities: [...]          # Phase 3+ only, update the severity tag list if findings changed (wrong / incomplete / ambiguous)
phase3_findings_post_release: ... # Phase 3+ only, recompute iff finding release-positions changed
```

If the release sync resolved a previously-recorded finding (upstream PR merged), update that finding's `**PR reference:**` field inline from `gastown#<N> (open)` to `gastown#<N> (merged in v<new-tag>)` and move the finding to the archived section of the corrections list.

### 7.4 Update `## Drift` and `## Implementation status` sections (Phase 3+)

- If a previously-captured drift is now fixed upstream → move it to a "Resolved drift (archived)" subsection of the Drift section with the commit SHA and date. Do NOT silently delete it.
- If new drift emerged → add a new entry under `## Drift` with the new `Claim → Code → Resolution` structure.
- If aspirational content became real → move the finding from `## Implementation status` to the `## What it actually does` body with a "(implemented in <commit>)" note; archive the old Implementation status entry.
- If vestigial content became truly removed → update the Implementation status tag from `vestigial` to `removed in <commit>` and mark the status in frontmatter.

### 7.5 Cross-link check (per page)

Before saving, run:

```bash
# Every markdown link in the page resolves
for link in $(grep -oE '\]\([^)]+\)' gastown/<page>.md | sed -E 's/\]\((.*)\)/\1/'); do
  if [[ "$link" == *.md ]]; then
    test -f "gastown/$(dirname <page>)/$link" 2>/dev/null || echo "BROKEN: $link"
  fi
done

# Every existing mention of this entity links to this page
entity="$(grep -oE '^title: .*' gastown/<page>.md | head -1 | sed 's/title: //')"
grep -rln "$entity" gastown/ | grep -v "^gastown/<page>.md$"
# For each result, check if it already links; if not, add a backlink in a follow-up commit.
```

---

## Step 8: Per new entity — create a full page

Use the Phase 2 template:

```markdown
---
title: <entity-canonical-name>
type: <binary|command|package|file|role|concept|workflow|service|...>
status: verified
topic: gastown
created: 2026-04-14
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/<path>
tags: [<relevant>, <tags>]
---

# <Canonical Name>

**Also known as:** <aliases, if any>

## What it actually does

<code-grounded body with file:line references>

## Docs claim

<verbatim quotes from docs + source refs — Phase 3+ only; empty if no docs yet exist for the new entity>

## Drift

<leave empty for new entities until Phase 3 audit touches them; new entities start with no known drift>

## Notes / open questions

<anything that's worth flagging but not yet drift>
```

After writing:

- Add the page to `index.md` in the appropriate category section.
- Update `gastown/README.md`'s sub-index for the category.
- Add to `gastown/inventory/` if the category has an inventory page (e.g., `go-packages.md`, `docs-tree.md`).
- Run the cross-link check in 7.5 against this new page.

---

## Step 9: Per removed entity — mark vestigial

**Never delete the page.** Update its frontmatter and add an `## Implementation status` section:

```yaml
---
title: <entity-name>
type: <original-type>
status: vestigial
topic: gastown
created: <original>
updated: 2026-04-14
sources:
  - /home/kimberly/repos/gastown/<path>  # may be gone at the new release tag — preserve the historical citation
tags: [<tags>, vestigial]
---
```

Add at the top of the body:

```markdown
## Implementation status

- **Status:** vestigial (removed from upstream)
- **Removed in:** commit `<SHA>`, <date>
- **Reason (from commit message or PR):** <reason>
- **Last verified present at:** commit `<prior-SHA>`
- **Replacement (if any):** <link to the entity that replaces it>
```

The body below this section — the original `## What it actually does`, etc. — is preserved as a historical map. Future readers can see what the entity used to do and why it was removed.

Add the page to `gastown/drift/vestigial.md` (a new per-category aggregation page, created lazily when the first vestigial page lands).

---

## Step 10: Cross-link checks on every touched page

Already performed per-page in Step 7.5. This step is the batch-level confirmation:

```bash
cd ~/repos/gt-wiki

# List every page touched in this batch
touched_pages="$(git diff --name-only --staged | grep '\.md$' | grep gastown/)"

# Count outbound links per page
for p in $touched_pages; do
  count=$(grep -oE '\]\([^)]+\.md\)' "$p" | wc -l)
  echo "$p: $count outbound links"
done

# Any page with zero outbound links is suspect unless it's a very small entity
```

Expected: every touched page has at least one outbound markdown link (entity pages always reference at least the parent category inventory or the related role/concept).

---

## Step 11: Batch entry in log.md + commit + push

### 11.1 Write the batch entry

Append to `log.md`. Use the phase-appropriate verb:

- **Phase 2 style (`ingest`):** when new entities are added or existing pages are refreshed against current source without drift findings.
- **Phase 3 style (`drift-found`):** when the sync surfaces new drift, resolves existing drift, or touches Phase 3 audit state.
- **Phase 3 style (`lint`):** when the sync fixes wiki-stale findings in-place without touching docs-vs-code drift.

**Template (ingest style for release syncs):**

```markdown
## [YYYY-MM-DD] ingest | Release sync to gastown v<new-tag>

**Scope:** Release sync from gastown `<prev-tag>` (SHA `<prev-sha>`) to `v<new-tag>` (SHA `<new-sha>`). N commits between releases, M files changed. (On the first run of this skill: substitute `<untagged-base-sha>` for `<prev-tag>` and note it as the Phase 2 mapping baseline.)

**Source files re-read at the new release tag:**
- `/home/kimberly/repos/gastown/<path>` (full re-read)
- ...

**Wiki pages updated:**
- [page](path/to/page.md) — refreshed `file:line` citations, `updated: YYYY-MM-DD` bumped
- ...

**Wiki pages created:**
- [new-entity](path/to/new-entity.md) — new in commit `<SHA>`
- ...

**Wiki pages marked vestigial:**
- [removed-entity](path/to/removed-entity.md) — removed in commit `<SHA>`, reason: <reason>
- ...

**Drift findings resolved (Phase 3+ only):**
- [page](path/to/page.md) — previous drift "<summary>" fixed upstream in commit `<SHA>`; archived in the page's Drift section.

**Wiki-stale findings fixed (lint-style inline):**
- [page](path/to/page.md) — Phase 2 body said "<old>"; current source at `file:line` does "<new>". Fixed in place.

**Beads updated:**
- Closed: <bead-id> (reason: resolved by release)
- Opened: <bead-id> (new entity needs follow-up)

**Cross-link discipline:** N new outbound links added. All touched pages pass the backlink check. All `file:line` citations re-verified against gastown `v<new-tag>`.

**Next batch:** <name or "none — sync complete">. Tracked in bead <id>.

→ <list of pages touched>
```

### 11.2 Stage and commit

```bash
cd ~/repos/gt-wiki
git add log.md gastown/ index.md
git status                                         # review staged files before commit
git diff --cached --stat                           # visual sanity check

git commit -m "$(cat <<'EOF'
Release sync to gastown v<new-tag>: <short summary>

<2-3 sentence description of the scope, using the log entry as reference.>

<Note any Phase N context, any notable drift resolutions, any notable
new entities or vestigial markings.>

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### 11.3 Push

```bash
cd ~/repos/gt-wiki
git pull --rebase
bd dolt push 2>&1 || echo "bd dolt push failed — investigate"
git push
git status                                         # must show "up to date with origin"
```

### 11.4 Brief Kimberly

One-paragraph summary of what landed, citing commit SHA, the number of pages touched, and the next batch (if any).

---

## Worked example (abbreviated)

**Scenario:** gastown releases a minor update adding `gt swarm` (new command), refactoring `internal/polecat/` (method rename), and removing `internal/cmd/legacy_sling.go` (no wiki page).

**Step 1 output:**
```
Previously-documented release (from gastown/README.md marker): v0.6.3 (SHA 9a1d8fe, 2026-03-12)
New release tag: v0.7.0 (SHA 4a8f2c1, 2026-05-01)
Delta v0.6.3..v0.7.0: 14 commits, 23 files
A: internal/cmd/swarm.go
A: internal/cmd/swarm_test.go
M: internal/polecat/polecat.go (Spawn → StartSandbox rename)
M: internal/polecat/polecat_spawn.go (internal refactor)
D: internal/cmd/legacy_sling.go
M: README.md (new swarm command mentioned)
M: docs/CLEANUP.md (new entry for gt swarm cleanup semantics)
...
```

**Step 2:** active plan is Phase 3 (`.claude/plans/2026-04-14-phase3-drift.md`); last batch was Batch 3 Sweep 1 files/inventory. Release sync lands as an ad-hoc `lint`-style batch between Sweep 1 and Sweep 2 batches.

**Step 3 classification:**
| Bucket | File | Wiki page |
|--------|------|-----------|
| New entity | internal/cmd/swarm.go | (create gastown/commands/swarm.md) |
| Modified entity | internal/polecat/polecat.go | gastown/packages/polecat.md, gastown/roles/polecat.md |
| Removed entity | internal/cmd/legacy_sling.go | (none — no wiki page existed) |
| Internal refactor | internal/polecat/polecat_spawn.go | none (wiki cites polecat.go, not spawn.go, for the affected claim) |

**Step 6 proposal:** 3 batches:
1. Batch A: new `gt swarm` page + inventory update + README sub-index.
2. Batch B: polecat rename refresh (Spawn → StartSandbox) on `packages/polecat.md` and `roles/polecat.md`, including a `wiki-stale` finding on the roles page (the old Phase 2 body said "`Spawn` allocates a worktree"; current source says `StartSandbox`).
3. Batch C: `docs/CLEANUP.md` one-row check — the new swarm row — folded into the Phase 3 Batch 9 scope when it happens.

**Step 11 log entry** (abbreviated):

```markdown
## [2026-05-01] ingest | Release sync to gastown 4a8f2c1 (Batch A: gt swarm new command)

**Scope:** New top-level command `gt swarm` landed upstream in commit `4a8f2c1`. This batch produces the wiki entity page and integrates the new command into the index.

**Source files re-read at the new release tag:**
- `/home/kimberly/repos/gastown/internal/cmd/swarm.go` (full read, 247 lines)
- `/home/kimberly/repos/gastown/internal/cmd/swarm_test.go` (skim for exported test helpers)

**Wiki pages created:**
- [gastown/commands/swarm.md](../../gastown/commands/swarm.md) — new page; `phase3_findings: [none]`; cites `file:line` 1-247 from swarm.go

**Wiki pages updated:**
- [gastown/commands/README.md](../../gastown/commands/README.md) — added swarm to the Work Management group row + incremented progress counter to 112/112
- [index.md](../../index.md) — added swarm bullet to the Commands section
- [gastown/README.md](../../gastown/README.md) — added swarm to Work Management group sub-index

**Cross-link discipline:** 7 new outbound links on swarm.md (parent command group, adjacent commands, role pages). Backlink check passed. All `file:line` citations fresh from `v0.7.0`.

**Next batch:** Batch B — polecat Spawn→StartSandbox rename refresh + wiki-stale finding on roles/polecat.md. Tracked in bead wiki-<new>.

→ gastown/commands/swarm.md, gastown/commands/README.md, index.md, gastown/README.md
```

Each batch gets its own entry, its own commit, its own push, its own brief.

---

## Quick reference: verbs → shapes

| Log verb | When to use | Frontmatter impact |
|----------|-------------|---------------------|
| `ingest` | New source mapped; existing pages refreshed without drift change | `updated:` |
| `drift-found` | New drift captured OR existing drift resolved/updated (Phase 3+) | `updated:`, `phase3_audited:`, `phase3_findings:` |
| `lint` | Wiki-stale fix; self-correction against current source | `updated:` |
| `decision` | Schema or process change | no page changes, CLAUDE.md + log.md only |
| `experiment` | Build / test / run captured | `updated:` on any page touched |
| `query` | Synthesis answer filed back | new page or updated page, `updated:` bumped |

Release syncs almost always use `ingest`, sometimes `drift-found` (if Phase 3+) or `lint` (for wiki-stale fixes). If in doubt, use `ingest` and note the drift or lint findings in the entry body.
