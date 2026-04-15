---
name: compact-recovery
description: Use at the start of any session that has lost context — after auto-compact, `/clear`, `/resume`, a fresh session, or whenever you realize you don't know where you are in the wiki's work. Walks through a minimal reading checklist (log.md timeline, active plan file, git state, bd state, plan file drafts, memory files) and produces a one-paragraph situational summary so you can resume where the prior session left off without guessing. This skill is for ORIENTATION only — do not start doing work until you have the summary.
---

# Compact Recovery

## Overview

When context is lost (auto-compact, `/clear`, `/resume`, cold-start, partial transcript), this skill walks you through the minimum reads needed to re-orient to the project's current state. The goal is a one-paragraph situational summary in ~5 minutes — what phase, what batch, what's in flight, what's next — so you can resume work instead of guessing.

**Core principle: do not start doing anything until you know where you are.** A confused session making a commit is worse than a 5-minute read that produces a clean resume. If you catch yourself about to act without orientation, STOP and run the checklist.

## When to use

Triggers (any one is sufficient):

- You just woke up from auto-compact and have a partial conversation tail
- Kimberly typed `/clear`, `/resume`, or started a new session
- You don't remember what batch / task / sub-batch you were in
- You see unfamiliar in-progress beads in `bd list` and don't know why they're there
- The working tree has uncommitted changes and you can't remember what they're for
- Kimberly says "where are we?" or "resume where you left off" or "pick up where the last session stopped"
- You're about to act on a user message but don't recognize the context it references

When NOT to use:

- Mid-batch when you already know where you are — just keep working.
- Fresh session where Kimberly has already given you a concrete next instruction (that's not orientation, that's task execution).
- Debugging a specific bug or answering a specific query — use `systematic-debugging` or the relevant task-specific skill instead.

## The read-in-order checklist

Do these in order. Each step informs the next. Total time: ~5 minutes for a clean resume, ~10 minutes if the state is complicated.

### 1. log.md timeline (1 min)

```bash
cd ~/repos/gt-wiki
grep "^## \[" log.md | tail -20
```

This gives you the last 20 events. The most recent `decision` / `ingest` / `drift-found` / `lint` entry is where the prior session's work-thread was. Note the date of the most recent entry — if it's older than today, work has been idle.

### 2. log.md recent entries (2 min)

Read the last 3-5 entries in full. Use the Read tool starting from the line number of the 5th-to-last entry (from step 1). The prose tells you WHAT was done and WHY. Look especially for:

- The most recent `decision` entry (schema changes, scope decisions, phase transitions)
- The most recent batch-entry log (which batch / sub-batch is complete)
- Any "Next batch" references (tells you what was supposed to come next)

### 3. Active plan file (2 min)

```bash
cd ~/repos/gt-wiki
ls -t .claude/plans/*.md | head -1
```

Read the most recent plan file in full. It's the authoritative "what are we doing right now" document. Pay special attention to:

- **Phase roadmap table** (if present) — tells you which phase you're in and where it fits in the overall sequence.
- **Current batch roadmap entry** — the most recent batch that's marked "DETAILED BELOW" or "in planning" is your active batch.
- **"Drafts for Kimberly review" section** — if populated with content (not placeholder text), that's in-flight work awaiting a review gate. You are MID-BATCH.
- **Task-level steps** — if task checkboxes are mixed (some `[ ]`, some `[x]`), that tells you which task is next.

### 4. Git state (30 sec)

```bash
cd ~/repos/gt-wiki
git log --oneline -10
git status --short
git branch -v
```

Check:

- **Commits ahead of origin** — if large (>3), multiple batches landed but haven't pushed. Session may have been interrupted mid-batch sequence.
- **Uncommitted changes** — in-flight work that needs to be resolved before proceeding. Do NOT blindly `git checkout` or `git stash` — investigate first.
- **Untracked files** — may be new skills / new plan files being created. Check each before deciding what to do.

### 5. Bead state (30 sec)

```bash
bd list --status=in_progress
bd ready
bd list -l wants-wiki-entry
```

Check:

- **In-progress beads** — these are the claimed work the prior session was doing. They're your active tasks.
- **Ready beads** — next actionable items once in-progress beads close.
- **`wants-wiki-entry` beads** — queued handoffs to Kimberly awaiting her attention.

### 6. Memory files (automatic — already loaded)

The memory files under `~/.claude/projects/-home-kimberly-repos-gt-wiki/memory/` are auto-loaded into every session via `MEMORY.md`. Trust them for who Kimberly is, what the directory layout means, and persistent context. If they contradict current state, current state wins and the memory should be updated via the `update-memory` protocol.

### 7. (Optional) Partial conversation tail

If this session has a partial transcript from before the compact, scan it for:

- Kimberly's most recent instruction (usually the last user message)
- Any pending decisions she was asking you to rule on
- In-flight draft text or edits that were being made

Post-compact you often have a partial conversation tail — use it as context but verify against the log + plan + git state, not the other way around.

## The synthesis

After running the checklist, produce a one-paragraph situational summary in this shape:

```
Situational state: [current phase, e.g. "Phase 3 Batch 0 Task 5 (Review gate)"].
Last committed work: [most recent commit SHA + one-line summary].
Commits ahead of origin: N (pushed | not pushed).
In-flight: [uncommitted changes / populated plan drafts / in-progress beads / etc.].
Next action: [specific task or "awaiting Kimberly's verdict on <artifact>"].
Blockers: [any / none].
```

**Hand this summary to Kimberly as your first message.** Wait for confirmation or redirect before doing anything else.

If the checklist reveals something surprising (a commit you don't recognize, an uncommitted change you can't explain, a plan file in a state that doesn't match the log), FLAG IT in the summary. "Something unexpected: `<description>`" — let Kimberly resolve before you proceed.

## Common recovery patterns

**Pattern 1: Mid-batch, drafts populated, awaiting review.**
- Signals: plan file's drafts section has content (not placeholder text); a review-gate task is the current step; working tree may be dirty with uncommitted working-tree edits on a tracked schema file (e.g. `CLAUDE.md`).
- Action: present the situational summary and ask Kimberly for verdict on the review gate.

**Pattern 2: Mid-batch, task-level steps partially done.**
- Signals: plan file task checkboxes are mixed; some commits on the current batch have landed; no drafts pending.
- Action: identify the next unchecked task step and propose executing it.

**Pattern 3: Just committed a batch, next batch not started.**
- Signals: git log shows a recent batch-completion commit; working tree clean; plan file roadmap shows the next batch as "not yet detailed" or similar.
- Action: propose starting the next batch and await approval. Do NOT dispatch subagents without explicit go-ahead.

**Pattern 4: Between phases, awaiting plan approval.**
- Signals: plan file has draft subsections ready for review; no new commits since the plan file was last edited; no in-progress beads.
- Action: ask Kimberly for plan approval / revisions. Do NOT start executing tasks from an unapproved plan.

**Pattern 5: Untracked files + uncommitted changes, state unclear.**
- Signals: `git status` shows a mix of modified, staged, and untracked files; no recent commits corresponding to them.
- Action: STOP. Run `git diff` and `git diff --cached` and `git status`. Present the full working tree state to Kimberly and ask what to do. Do NOT `git stash`, `git checkout`, or `git clean` without explicit instruction.

**Pattern 6: Nothing obviously in flight.**
- Signals: working tree clean, no in-progress beads, recent commits pushed.
- Action: `bd ready` for next available work, then present options to Kimberly.

**Pattern 7: bd dolt trouble detected.**
- Signals: `bd` commands hang, return errors, or time out; `bd list` returns empty when you expected content.
- Action: this is the gastown Dolt server issue (see `~/repos/CLAUDE.md` "Dolt Server — Operational Awareness"). Capture diagnostics with `kill -QUIT $(cat ~/gt/.dolt-data/dolt.pid)` before restarting. Escalate via `gt escalate -s HIGH "Dolt: <symptom>"` if this is the broader Gas Town environment, or present the issue to Kimberly if it's the wiki's embedded bd. Do NOT restart bd blindly — collect diagnostics first.

## Red flags — STOP and re-orient

These mean you're about to act on guesses instead of evidence:

- You're about to run a command without reading `log.md` first → STOP.
- You're about to commit without knowing how many commits you're ahead of origin → STOP.
- You're about to reply "done" / "complete" / "ready to push" without running `git status` → STOP.
- You think you know what batch you're in but haven't read the plan file → STOP.
- You're about to say "continuing where we left off" without a concrete next step from the synthesis → STOP.
- Kimberly asked a specific question referencing context you don't have (e.g. "does that work?" where "that" is ambiguous) → do the checklist FIRST, then answer with a concrete reference.
- You're about to modify a gitignored file like a plan draft without confirming its current state → STOP.

## Common mistakes

| Mistake | Fix |
|---|---|
| Skipping the checklist because "I remember" | You don't remember. Do it anyway — it takes 5 minutes and prevents wrong actions. |
| Reading the plan file before `log.md` | Wrong order. `log.md` is the ground truth for state; the plan is intent. Intent matters, but state is authoritative. |
| Making a commit before reading `git log` | You might commit to the wrong branch, overwrite in-progress work, or duplicate a just-landed commit. |
| Assuming the next task number | Re-read the most recent batch's log entry to confirm which sub-batch is next. |
| Skipping the "Drafts for Kimberly review" check | Populated drafts mean work is ready for review, not ready to commit. The drafts ARE the in-flight work. |
| Asking Kimberly "what's next?" without synthesis | Do the checklist first, then present your synthesis for confirmation/redirect. Asking without synthesis wastes her time. |
| Running `git stash` / `git checkout` / `git clean` to "clean up" uncommitted changes | Never. Investigate first. The uncommitted changes may be critical in-flight work. |
| Dispatching a subagent immediately post-recovery | Subagents inherit limited context. Run the checklist yourself first, then dispatch with a specific task scope. |

## Self-check — did you actually recover?

After producing the synthesis, run this self-evaluation before handing it to Kimberly. If any answer is "no" or "I'm not sure," go back and re-read the source. **A wrong synthesis is worse than admitting you need another pass.**

### Coverage checklist

Your synthesis must include all of these. Tick each one before sending:

- [ ] **Current phase** — named explicitly (e.g. "Phase 3 Batch 0 Task 5"), not implied ("drift work")
- [ ] **Most recent committed work** — cited by SHA or at least by log.md entry title
- [ ] **Commits-ahead-of-origin count** — a specific number from `git status` / `git log origin/main..HEAD`
- [ ] **Working tree state** — clean / modified / staged / untracked, specifically
- [ ] **In-flight artifacts** — every uncommitted draft, every in-progress bead, every populated plan-file draft subsection. Name them individually, not collectively.
- [ ] **Concrete next action** — either a specific task (with its step number) or an explicit "awaiting Kimberly's verdict on X"
- [ ] **Any surprises** — anything the checklist surfaced that doesn't match your expectations

### Self-check questions (you must be able to answer all of these)

If you can't answer one, re-read the relevant source before sending the synthesis:

1. **"What is the most recent commit SHA and one-line summary?"** — Answer from `git log --oneline -1`. If you can't quote it, you didn't read git state.
2. **"What phase are we in, and who says so?"** — Answer should cite either the active plan file's header / roadmap OR a recent `decision` log entry. "I think Phase 3" is not an answer; "The active plan at `.claude/plans/<file>.md` says Phase 3 Batch 0" is.
3. **"What was the last action Kimberly explicitly approved?"** — Answer from `log.md` `decision` entries or the conversation tail. This tells you what state she expects.
4. **"What's in the plan file's 'Drafts for Kimberly review' section right now?"** — Either empty/placeholder, or populated with draft text. If populated, name which drafts (e.g. "PR-delta scoping, Sweep 1 coverage audit, schema-change decision, retroactive-cleanup decision").
5. **"If I do nothing, what will the next message from Kimberly probably be about?"** — This tests whether your synthesis matches her mental model. If you can't predict it, you're missing context.
6. **"What would be WRONG to do right now without asking?"** — Forces you to think about guardrails. E.g. "commit without her approval of the review gate," "dispatch a subagent for Batch 1 before Batch 0 closes," "modify CLAUDE.md in a way that affects tracked files without her sign-off."
7. **"Is there anything in the working tree I can't explain?"** — `git status` output should be 100% explainable from the log + plan. Unexplained files are a red flag.

### Verification commands (sanity-check the synthesis against ground truth)

Before presenting the synthesis, run these three commands as a reality check:

```bash
# 1. Commit count ahead of origin — must match what your synthesis claims
git log --oneline origin/main..HEAD | wc -l

# 2. Working tree state — must match what your synthesis claims
git status --short

# 3. Plan file drafts state — populated or placeholder?
grep -A 2 "#### Draft:" .claude/plans/$(ls -t .claude/plans/*.md | head -1 | xargs basename)
```

If any of these outputs contradict your synthesis, the synthesis is wrong. Re-read and revise.

### Example: good synthesis vs bad synthesis

**Bad (vague, uncheckable):**

> We're in Phase 3 doing drift analysis. Some commits have landed, and I think the plan is to keep working on Batch 0. Let me know what's next.

Problems: no phase specificity, no SHAs, no commit count, no in-flight state, no concrete next action, no self-check.

**Good (specific, checkable, actionable):**

> Situational state: Phase 3 Batch 0 Task 5 (Review gate) — awaiting your verdict on the schema v1.2 change set.
> Last committed work: `11a2e88` "Update writing-entity-pages + maintaining-wiki-schema skills for Phase 3 v1.2" — landed as part of the CLAUDE.md restructure sequence.
> Commits ahead of origin: 10 (unpushed).
> In-flight: (1) four populated drafts in `.claude/plans/2026-04-14-phase3-drift.md` — PR-delta scoping, Sweep 1 coverage audit, schema-change decision entry, retroactive-cleanup decision entry; (2) working tree clean; (3) no in-progress beads (`wiki-ytq` is the Phase 3 epic, awaiting rehabilitation in Task 7).
> Next action: you review the four drafts + the already-landed restructure commits as the v1.2 on-disk state, then verdict (a) approve all / (b) revise / (c) abort. On verdict (a), I proceed to Task 6 (commit the two log.md decision entries) and Task 7 (rehabilitate wiki-ytq + file Batch 1 anchor bead).
> Blockers: none — ready for your verdict.
> Surprises: none.

Each claim is grounded in a specific source and verifiable by the commands above. This is what "recovered" looks like.

### If you fail your own self-check

- **Missing info:** re-read the specific source that would provide it. Don't guess.
- **Contradictions between sources:** flag them explicitly ("plan says X but log says Y") and ask Kimberly to reconcile rather than choosing.
- **Too many things at once:** run the checklist one step at a time instead of skimming. Depth beats speed.
- **Suspicion something is off but you can't name it:** that's usually right. Keep digging until you name it or explicitly note "something is off, here's what I've verified and what remains unclear."

## What to do if the checklist doesn't resolve the state

If you finish the checklist and still don't know where you are, that's informative — it usually means:

1. **There's no active work.** Present the summary as "no active work; awaiting instruction" and ask Kimberly what to do.
2. **The plan file is out of sync with `log.md`.** Surface the discrepancy ("plan says we're in Batch X but log.md's most recent entry is Batch Y") and ask Kimberly to reconcile.
3. **A bead is claimed but with no corresponding plan or log entry.** Surface the orphan bead and ask Kimberly whether to close, re-scope, or continue.

Never guess. Never start doing something "just in case." Orientation precedes action.

## Reference

- **CLAUDE.md** — "Session startup (required reading)" section (same 7-step orientation, less detail)
- **`log.md`** — the event log, authoritative state
- **`.claude/plans/`** — active phase plan (gitignored; grep for the most recent file)
- **`writing-entity-pages` skill** — for the schema you're working within
- **`tracking-wiki-work` skill** — for bd commands + actor conventions
- **`maintaining-wiki-schema` skill** — for if the lint workflow surfaces something weird during recovery
- **`syncing-gastown-updates` skill** — for if the prior session was doing a release sync
