# Retrospectives

Append-only retrospective log for the gt-wiki project. Complements [log.md](log.md): log.md records **what happened**, retros.md records **how the work felt and what to change next time**.

## Who writes here

- **Subagents** append a `stage` retro at the end of every unit of dispatched work (one sub-batch, one task, one focused investigation). Written as the last step of the subagent's run, before the subagent reports back to the main session.
- **Main orchestrator (me)** appends a `phase` retro at the end of every phase, consolidating patterns across the phase's stage retros and surfacing recommendations for the next phase.
- **Scheduled retros** are human-LLM discussions Kimberly and I hold when enough friction or success patterns accumulate to warrant it. I flag when I think it's time ("we should schedule a retro"); Kimberly decides when. Notes from scheduled retros land here as `scheduled` entries.

## Rules

1. **Append-only.** Historical entries are preserved verbatim. Corrections land as new entries that reference the old one, not as in-place edits. Same rule as log.md.
2. **Honesty over flattery.** Retros are information, not performance. If a stage went badly, say so plainly. A retro that only records what went well is a broken retro.
3. **Every "what to change" item becomes a concrete follow-up.** Filed as a bead (`bd create`), a skill edit, a plan edit, or a note for the next scheduled retro. Vague aspirations ("do better next time") are not actionable and don't belong here.
4. **Every retro entry tags its origin.** `stage` entries name the subagent role + the phase/batch/sub-batch ID. `phase` entries name the phase number. `scheduled` entries name the participants and the trigger.
5. **Timeline scannable via grep.** `grep "^## \[" retros.md` gives the full retro timeline, same discipline as log.md.

## Entry formats

### Stage retro (subagent — end of sub-batch / task)

```markdown
## [YYYY-MM-DD HH:MM] stage | <phase>.<batch>.<sub-batch> — <short label>

**Actor:** <subagent role / prompt type>
**Unit:** <what this stage covered — pages audited, files read, commits landed>
**Duration:** <approximate wall-clock or "one dispatch">

**What went well:**
- <specific thing, not generic praise>

**What didn't:**
- <specific friction, confusion, or wrong turn>

**What to change next time:**
- <concrete suggestion — skill edit, plan edit, prompt adjustment, bead filing>

**Follow-ups filed:**
- bd-<id> | <title>  (or: "none — lessons are purely informational")
- skill edit suggestion: <skill path> | <change>

**For Kimberly retro discussion:** <optional — only when the lesson is too big for a one-line fix>
```

### Phase retro (main orchestrator — end of phase)

```markdown
## [YYYY-MM-DD] phase | Phase <N> — <name>

**Scope:** <what the phase produced>
**Duration:** <start date → end date>
**Stages logged:** <count of stage retros contributing>

**Patterns across stages:**
- <cross-cutting observations from the stage retros>

**What went well:**
- <project-level, not stage-level>

**What didn't:**
- <project-level friction>

**Schema / skill changes surfaced:**
- <what the phase's experience suggests we should bake in before the next phase>

**Recommendations for next phase:**
- <actionable>

**Outstanding follow-ups:**
- bd-<id> list
- skill edits not yet landed
- plan items deferred

**For Kimberly retro discussion:** <always — phase retros are a natural trigger for a scheduled conversation>
```

### Scheduled retro (human + LLM — triggered discussion)

```markdown
## [YYYY-MM-DD] scheduled | <trigger / topic>

**Participants:** Kimberly + <LLM role>
**Trigger:** <what prompted the scheduling — phase close, friction threshold, specific incident>

**Discussed:**
- <topic> — <conclusion>

**Decided:**
- <decision> → landed as <log.md entry | skill edit | plan edit | bead>

**Deferred:**
- <topic> — <why deferred + when to revisit>
```

## When to flag "we should schedule a retro"

I surface the suggestion to Kimberly when any of these triggers fire:

- A phase just closed (automatic trigger — phase retros warrant discussion).
- Three or more stage retros in a row surface the same "what didn't" pattern.
- A stage retro contains a "For Kimberly retro discussion" note that's too big for a one-line fix.
- A subagent dispatch went badly enough that the pattern needs rethinking before the next dispatch.
- Kimberly has expressed frustration about a recurring friction that I haven't surfaced yet.
- It's been more than one full phase since the last scheduled retro and the backlog of discussion items is non-empty.

Flagging is cheap; Kimberly decides when to actually schedule.

---

## Timeline

## [2026-04-15 00:00] stage | 3.1.1a — Diagnostics Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1a
**Unit:** 22 `GroupDiag` command pages audited; 4 `cobra drift` findings promoted to `## Docs claim` + `## Drift` sections; 18 pages tagged `phase3_findings: [none]`; drift index stub created at `gastown/drift/README.md`; one commit
**Duration:** one dispatch

**What went well:**

- The plan file's "Batch entry format" section and the writing-entity-pages skill's `## Drift` / `## Docs claim` templates were detailed enough to emit correctly-shaped sections without back-and-forth on structure. The only structural question I had to resolve myself was whether to place `## Docs claim` before or after `## Related` — I followed the writing-entity-pages scaffold (Docs claim before Related) rather than some pages' inherited Phase 2 ordering.
- Re-reading `activity.go`, `repair.go`, and `status.go` in full against current HEAD surfaced the same findings Phase 2 had noted but with fresh line refs — the code-first verification discipline produced no line-ref corrections in this sub-batch because Diagnostics commands haven't churned since Phase 2. (See "what to change" #3 below — this is itself a finding.)
- The `v1.0.0` release-position check via `git show v1.0.0:<file>` worked cleanly for all four finding pages. Every cited code block was byte-identical to v1.0.0, so every finding is `in-release`. No post-release surprises in Diagnostics.
- Phase 2's Notes sections were mostly high-quality — 18 of 22 were genuinely neutral observations (implementation curiosities, architecture notes, open questions about helper functions), not mis-filed drift. The 4 findings I did promote were all obvious once the source was re-read.

**What didn't:**

- **The plan's "expected high-yield" list for Batch 1 is stale in one case.** The plan's Batch 1 section names `info` alongside `directive`, `hooks`, `tap`, `warrant`, `repair` as parent-only stubs whose `Long` text advertises subcommands that aren't wired. Current `info.go` is a terminal command with two flags and no subcommands, and its `Long` text is accurate. Either `info` was fixed between Phase 2 and now, or the plan's categorization was wrong from the start. I did not adjust the plan — that's a main-orchestrator call — but future sub-batches should not spend effort on `info` expecting a cobra-drift yield.
- **Ambiguity about whether to promote drift observations that live in the Phase 2 page body rather than in `## Notes`.** Sweep 1's charter says "promote Phase 2 notes bullets to proper sections." Doctor's drift observation was already written in the page body ("The Long description categories are documentation-only — the actual registration order in `runDoctor` above is what ships"); repair's was also in the body as a "Drift risk: ... This is a neutral observation — not filed as drift per phase scoping" paragraph. I promoted both anyway because the charter's intent is clearly "formalize all drift observations regardless of Phase 2 parking spot," but the plan could be explicit about this. A stricter reading would leave body drift observations alone in Sweep 1 and let Sweep 2 or Batch 13 pick them up.
- **The dispatch prompt's "Classify" step lists 8 categories including `neutral`, but the writing-entity-pages SKILL.md taxonomy table lists 9 (the same 8 plus `neutral`).** The skill and the dispatch prompt are consistent, but the plan's `phase3_findings` enum legal-values list explicitly names 8 category tags including `none` but NOT `neutral`. That's semantically fine — neutral observations stay in Notes and don't produce a finding tag — but the terminology confused me briefly: is `neutral` a category that takes up a tag, or a non-tag category that means "stays in Notes, no tag"? It's the latter. A one-line clarification in the skill's taxonomy table would prevent the same pause for the next subagent.
- **The `→ promoted to ## Drift` redirect format in the dispatch prompt is ambiguous about what to do when the drift observation lives in the page body, not in a bullet.** I left the body observation in place (rewriting it to point forward to `## Drift`) for doctor; for repair I replaced the "drift risk ... neutral observation" paragraph with a forward reference. Both are reasonable but inconsistent. The skill could standardize: "when promoting a Phase 2 body observation, replace the original with a one-line forward link; when promoting a Notes bullet, either remove it or add a `→ promoted` redirect suffix."

**What to change next time:**

- **Plan file:** remove `info` from the Batch 1 "expected high-yield" list in `.claude/plans/2026-04-14-phase3-drift.md` at the `Batch 1 — Sweep 1: commands/` section (around line 395). It's noise for Batches 1b–1h dispatchers who see it in their reading list.
- **Plan or dispatch template:** add one sentence to the Sweep 1 charter clarifying that drift observations already written into the Phase 2 page body (rather than in `## Notes / open questions`) should be promoted to `## Drift` sections too — with the original body text rewritten to forward to the new section. I wouldn't make this a separate "pre-existing body drift" category; it's just a clarification that "drift observation in body" is equivalent to "drift observation in notes" for promotion purposes.
- **Skill edit (`.claude/skills/writing-entity-pages/SKILL.md`):** add a one-line clarification to the drift taxonomy table that `neutral` is not a tag that appears in `phase3_findings:` frontmatter — it means "stays in Notes, the page's `phase3_findings` is `[none]` or omits this observation." The current wording ("Stays in `## Notes / open questions`. No promotion.") is accurate but doesn't answer "so what's in the frontmatter list."
- **Dispatch template for Batch 1b onward:** add a one-line note that `v1.0.0` release-position verification is cheap (one `git show v1.0.0:<file>` per finding, byte-compare to HEAD) and should always be done inline when a finding is being written, not deferred. I did it inline for all four findings this sub-batch and it was zero-friction.

**Follow-ups filed:**

- none (bd beads) — all follow-ups are documentation/skill/plan edits that the main orchestrator should apply before Batch 1b dispatches.
- skill edit suggestion: `.claude/skills/writing-entity-pages/SKILL.md` drift taxonomy table | clarify that `neutral` doesn't appear in `phase3_findings:` frontmatter
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Batch 1 section | remove `info` from "expected high-yield" list; add one sentence clarifying body-drift promotion semantics

**For Kimberly retro discussion:** nothing blocking, but the "body drift vs notes drift" promotion question is a potentially recurring pattern — if other Phase 2 categories (packages, files, concepts) have more body-parked drift observations than notes-parked ones, the Sweep 1 charter will need a real rewrite rather than a one-line clarification.

## [2026-04-15 02:00] stage | 3.1.1b — Configuration Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1b
**Unit:** 11 `GroupConfig` command pages audited; 6 `cobra drift` findings promoted; 2 `wiki-stale` findings (one page has both, total 7 Drift-section rows); `## Docs claim` + `## Drift` sections added on 5 pages (`account`, `config`, `hooks`, `plugin`, `shell`, `uninstall` — 6 pages, of which `hooks` carries both a cobra-drift and a wiki-stale finding); `directive` was fixed inline via wiki-stale body rewrite (no `## Docs claim`/`## Drift` section, since wiki-stale's fix tier is the wiki itself); 4 pages tagged `phase3_findings: [none]` (`disable`, `enable`, `issue`, `theme`); one commit
**Duration:** one dispatch

**What went well:**

- The dispatch prompt's pre-flagged "expected high-yield pages" list (directive, hooks, shell, plugin, issue, uninstall, config/theme dual writers) was unusually accurate this time — all six named pages except `issue` and the config/theme pair landed actual findings, and `issue`'s dispatch note explicitly said "likely NEUTRAL unless Long text claims beads-integration" which turned out correct. This is the opposite of Batch 1a's `info.md` stale high-yield miss — for 1b the high-yield list actively saved time by pointing the audit at the real findings on the first pass.
- The **sibling-file subcommand-wiring discovery on `directive` and `hooks`** was the load-bearing finding of this sub-batch. Both pages' Phase 2 bodies claimed "subcommands not wired up here" and speculated they might not be implemented at all. A 30-second `ls internal/cmd/directive*.go` and `grep -rn "hooksCmd.AddCommand"` confirmed all subcommands were wired in sibling files, all present at v1.0.0. This is exactly the kind of Phase-2-missed-context that the Phase 3 audit is supposed to catch — the Phase 2 synthesis took `directive.go`/`hooks.go` in isolation and assumed the absent `AddCommand` calls meant unwired subcommands, but Go's per-file `init()` pattern means `AddCommand` registrations are scattered across the tree. The re-read-in-context check caught it.
- The **`hooks init` cobra-drift finding** was a bonus discovery that came directly from reading `hooks_init.go` to verify the wiki-stale claim. The 9th subcommand's existence was not noted in the Phase 2 page at all (Phase 2 only listed the 8 that `hooks.go`'s `Long` names), and it would have been easy to miss if I had stopped at "confirmed all 8 are wired." Reading the full sibling-file directory (via `ls internal/cmd/hooks*.go`) surfaced `hooks_init.go` as a sibling-file not mentioned anywhere in the Phase 2 page. This argues for Batch 1c onward to always `ls` the parent file's sibling set, not just the parent itself.
- **`v1.0.0` release-position verification was zero-friction** — same observation as Batch 1a. For all six cobra-drift findings the v1.0.0 check was one `git show v1.0.0:<file> | sed -n '<lines>p'` and the text was byte-identical every time. Configuration commands are stable across v1.0.0 → HEAD.
- **The `config get dolt.port` self-contradiction** (Long text omits the key but the error message lists it) was a particularly clean finding — the same file at two different lines made opposing claims. Sharp fix-tier call: clearly code (edit the `Long`), clearly in-release, clearly minimal-risk since the error message already names the key as supported.

**What didn't:**

- **The wiki-stale vs drift-found log-entry category question.** Per the skill taxonomy table, wiki-stale findings are "logged under `lint` verb, not `drift-found`." But Batch 1b's audit pass produced wiki-stale findings interleaved with cobra-drift findings on the same pages, and splitting them into two separate log entries would (a) lose the audit-trail continuity of "this is what I found in one pass of re-reading the Configuration group" and (b) double the log-entry count for no clear reader benefit. I chose to bundle wiki-stale findings into the `drift-found` batch entry with a taxonomy note acknowledging the convention; Batch 1a set no precedent here because it had zero wiki-stale findings. This is a decision the orchestrator should ratify before Batch 1c. Candidate resolutions: (a) ratify the bundle-into-drift-found pattern for Sweep 1 sub-batches because Sweep 1 is "promotion from Phase 2 notes" and wiki-stale is a flavor of that; (b) require a separate `lint` entry per sub-batch that has wiki-stale findings, accepting the double-entry overhead; (c) distinguish "wiki-stale against churn" (real lint) vs "wiki-stale against Phase-2-time truth" (Phase 2 was wrong at the time, so arguably a drift-found because it's still a drift-against-code finding). My intuition says (a) is right for Sweep 1 and (c) is the analytically correct framing — Batches 1b's directive/hooks findings are **"Phase 2 was wrong at Phase 2 time against code that existed,"** not **"Phase 2 was right then and churn made it wrong,"** which is a meaningfully different thing.
- **The body-parked drift observation clarification from Batch 1a** was useful in principle but the 1b pages that had body-parked observations mostly already ALSO had them mirrored in Notes (plugin's "Gate semantics beyond cooldown" lived in Notes; shell's "install→enable coupling" lived in Notes; uninstall's "narrow workspace heuristic" lived in Notes). The charter clarification was aimed at "drift observation lives in body, not Notes" — that pattern didn't actually surface much in Batch 1b. The pages where the body was wrong (directive, hooks) were not body-parked drift observations; they were body-parked *claims* that turned out to be false, which is a different thing — wiki-stale, not promoted drift. The charter clarification didn't speak to this case. A separate charter sentence is needed: "when the body makes a factual claim about code behavior that is contradicted by re-reading source, the claim is `wiki-stale` regardless of whether Phase 2 framed it as a claim or an observation."
- **The `plugin` and `theme` dual-writer-of-`CLITheme` observations both stayed neutral**, but the reason is slightly weak. Neither command's `Long` claims to be the sole writer, so there's no docs claim to contradict. But a user reading `gt theme cli --help` would reasonably assume the theme command is the place to set the CLI theme, and a user reading `gt config set cli_theme --help` would reasonably assume the config command is the place. Both are true simultaneously, which is a maintenance hazard (future drift between the two validators) but not a current-state docs drift. Classified as neutral, consistent with Batch 1a's `--since` parser asymmetry neutral call. If a future phase catches one of the two writers diverging from the other's validator, this becomes a proper drift finding retroactively. Low-priority lint-style issue.
- **The `account` Commands-list omission is a borderline call**. Hand-maintained summary lists in `Long` text are common across the gastown CLI; treating every omission as a `cobra drift` finding will inflate the Phase 6 work-list. Batch 1a made the same call on `doctor` (the `Long` catalog enumerates a curated subset of checks). The consistent framing is "if the `Long` text is the only user-facing description of the subcommand/check/option set, an omission in that list is a drift claim." But the opposite framing — "hand-maintained summary lists are informational; cobra's own `Available Commands:` block is authoritative" — is also defensible and would demote these. I followed Batch 1a's precedent (promote) but flagged it as a judgment call in the log entry for orchestrator review.
- **The batch entry's cross-link discipline section grew verbose.** I found myself writing five distinct clauses about which pages got which sections, plus a parenthetical about taxonomy convention for wiki-stale. Batch 1a's entry was tighter. The extra length isn't wrong but it's a smell that the entry format could use a more structured template (e.g. "Sections added: N x Docs claim, M x Drift, P x redirects" rather than prose).

**What to change next time:**

- **Dispatch template for Batch 1c onward: add a mandatory `ls <parent-file>*.go` step.** Before re-reading a parent-looking `internal/cmd/<command>.go`, list the sibling-file set. If there are siblings named `<command>_*.go`, read each one's `init()` block for `<command>Cmd.AddCommand(...)` calls. This catches (a) wiki-stale "subcommands not wired" claims when Phase 2 missed the siblings, and (b) `cobra drift` claims where the parent's hand-maintained subcommand list is incomplete (as on `hooks init`). Cost: 10 seconds per parent page. Benefit: catches the exact class of findings that made Batch 1b's yield. This pattern would have caught `hooks init` much earlier if I had started there instead of finding it mid-audit.
- **Plan file clarification on wiki-stale log-entry placement.** The skill's "log under `lint` verb, not `drift-found`" rule should be qualified: "Sub-batch pass may consolidate wiki-stale findings into the `drift-found` entry if they surface interleaved with drift findings; standalone wiki-stale findings (e.g. a lint pass between audit batches) get their own `lint` entry." The distinction matters because Sweep 1 sub-batches are structurally audit passes, not lint passes. Batches 2-4 will hit this too.
- **Dispatch template: add a Phase-2-time-vs-churn clause for wiki-stale.** When a wiki-stale finding surfaces, the subagent should decide whether the Phase 2 claim was wrong at Phase 2 time (code existed at 2026-04-11 contradicting the wiki page on day one) vs stale against churn (code moved after Phase 2 landed). The two have different Phase 6 implications: "wrong at Phase 2 time" is a lint find about Phase 2's rigor; "stale against churn" is a release-sync find. Batch 1b's directive + hooks findings are both "wrong at Phase 2 time" per `git show v1.0.0:<file>`. Worth capturing that distinction in the log entry so Phase 6 and release-sync can scope differently.
- **Skill edit suggestion (non-blocking):** the `writing-entity-pages` skill's drift taxonomy row for `wiki-stale` should explicitly note that wiki-stale findings do NOT get `## Docs claim` / `## Drift` sections — they get inline edits to `## What it actually does` and a one-line note in `## Notes / open questions` pointing forward to the inline fix. Currently the skill says "Fix inline in `## What it actually does`. Log under `lint` verb, not `drift-found`." but doesn't explicitly say "no `## Drift` section." I followed this reading for `directive` (no Drift section) but for `hooks` I put the wiki-stale row inside the existing `## Drift` section alongside the cobra-drift finding because that page has both. If the skill rule is read strictly, the wiki-stale row should NOT live in a `## Drift` section even when it co-habits with a cobra-drift. A clarification would prevent the next subagent from making the same call.
- **Plan file update to the Batch 1 tracker table:** Batch 1b's actual numbers are 6 cobra-drift + 2 wiki-stale + 4 none out of 11 pages (82% yield if we count every non-none tag as yield, or 55% if we count pages-with-findings as yield). Compared to Batch 1a's 4-cobra-drift-out-of-22 (18% yield), Configuration is the highest-yield sub-batch so far. This changes the calibration expectation for Batches 1c-1h. Update the tracker table's "Findings count" column and add a "Yield surprise" note if the orchestrator wants to re-scope remaining sub-batches.

**Follow-ups filed:**

- none (bd beads) — all follow-ups are documentation/skill/plan edits for the main orchestrator to apply before Batch 1c dispatches or at the Sweep 1 retrospective.
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Batch 1 tracker table | update 1b row with 6 cobra-drift + 2 wiki-stale + 4 none; note yield surprise
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Sweep 1 charter | clarify wiki-stale log-entry placement (bundle vs separate lint entry) and Phase-2-time-vs-churn distinction for wiki-stale findings
- skill edit suggestion: `.claude/skills/writing-entity-pages/SKILL.md` drift taxonomy wiki-stale row | add explicit "no `## Drift` section — fix inline, note the correction in `## Notes / open questions`" clarification
- dispatch template enhancement: add mandatory `ls <parent-file>*.go` + sibling-file `init()` read step before any parent-looking command audit

**For Kimberly retro discussion:**

- The wiki-stale-at-Phase-2-time finding class is genuinely interesting. Two of 11 pages had wiki bodies that were wrong at Phase 2 time — not stale against churn, but wrong on day one, because Phase 2 took parent `.go` files in isolation without checking sibling-file registrations. This suggests Phase 2's coverage was patchier than the 213-page count implies — some pages were filed as `status: partial` or "parent-only stub" when the subcommand tree was actually fully wired. The Phase 3 audit is now doubling as a Phase 2 rigor check. If the pattern repeats in Batches 1c-1h and Batch 2 (packages/), it's worth re-scoping Phase 3 to include "re-verify Phase 2 'partial' status flags against current source" as an explicit sub-goal, rather than discovering them opportunistically. Could also be handled as a Phase-7 validation pass, but surfacing it now is cheaper than rediscovering later.
- The `account` and `doctor`-style "hand-maintained summary list is incomplete" drift class (now three findings across 1a + 1b) may be a single meta-finding worth documenting: "Gas Town's CLI `Long` texts include hand-maintained enumerations of subcommands/targets/keys that reliably drift from the code's actual registration/switch/enumeration lists." Phase 6 could address this class with a single pattern (e.g. auto-generate subcommand lists via cobra introspection instead of hand-maintaining them) rather than fixing each finding individually. Not Phase 3's call, but worth noting for the Phase 6 planning input.
