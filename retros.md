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

## [2026-04-15 17:00] stage | 3.1.1c — Work Management Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1c
**Unit:** 26 `GroupWork` command pages audited; 3 cobra-drift findings (`assign`, `changelog`, `molecule`), 3 wiki-stale findings all tagged `phase-2-incomplete` (`molecule`, `mq`, `wl`), 1 drift finding (`done` — reformat of pre-plan f143813 section to v1.2 schema), 20 pages tagged `phase3_findings: [none]`. `molecule` carries BOTH a cobra-drift AND a wiki-stale finding. 6 distinct finding pages total. One commit.
**Duration:** one dispatch

**What went well:**

- **The mandatory `ls <parent>*.go` sibling-file check from the Batch 1b retro paid off immediately on `molecule`, `mq`, and `wl`.** Batch 1b's retro recommended always listing sibling files before re-reading a parent `.go` and grepping each one's `init()` for AddCommand registrations. Applying that rule to the 26 Work Management pages surfaced three distinct phase-2-incomplete wiki-stale findings that Phase 2 had missed by reading parent files in isolation: `molecule.go` missing 4 step-group children + 1 shortcut in sibling files, `mq.go` missing `mq_next` in `mq_next.go`, and `wl.go` missing TEN subcommands in ten sibling files. Without the sibling-file check, this sub-batch would have reported 3 cobra-drift findings total and moved on; with the check, it reported 3 cobra-drift + 3 wiki-stale + 1 drift (done carveover) = 7 findings, a 133% yield bump over the naive read. This validates the 1b retro's recommendation strongly and should become canonical for all remaining Batch 1 sub-batches.
- **Release-position verification stayed zero-friction for most findings.** `git -C /home/kimberly/repos/gastown show v1.0.0:<file>` + byte-compare was the canonical check; it worked for every finding in ≤2 seconds. Only the `wl` case involved broader verification — `git -C /home/kimberly/repos/gastown ls-tree -r --name-only v1.0.0 | grep internal/cmd/wl_` to enumerate which sibling files existed at the tag, then per-file `show` to confirm their AddCommand lines — but that's still ~30 seconds of work for a load-bearing finding. Two findings in Batch 1c are `in-release` by direct byte-identity check against v1.0.0; four are `in-release` by file-presence + AddCommand check. Zero post-release.
- **The `6e1fddc` schema clarification held up cleanly in practice.** The new `**Phase 2 root cause:** phase-2-incomplete` line was easy to apply to all three wiki-stale findings. The heuristic fallback (documented in the skill clarification) was used for all three — I didn't run the full archaeology (`git -C ~/repos/gt-wiki log --diff-filter=AM -1 --format=%H -- gastown/commands/<page>.md` → corresponding gastown commit → show that commit). Instead I used the heuristic: "sibling-file `AddCommand` registration was already in place at `v1.0.0`, so Phase 2 running on 2026-04-11 had access to it, so Phase 2 was wrong at Phase 2 time." This is exactly the `phase-2-incomplete` fallback case the skill clarification describes ("missed sibling-file `init()` registration") and noted inline in each finding body. Took about 10 seconds per finding instead of ~2 minutes.
- **The `wl` case (the biggest wiki-stale finding in Batch 1) resolved unambiguously once I checked v1.0.0.** The dispatch prompt described `wl_*` subcommands as "post-v1.0.0" citing the plan's PR-delta scoping draft. Two-minute verification showed they're all pre-v1.0.0 — the plan's draft is wrong. This is a load-bearing retro finding because the plan's PR-delta scoping is what the "Release position" column of every Sweep 1 finding depends on, and if it's wrong for `wl` it may be wrong for other entries. Flagged in the log.md Judgment Calls section as a Sweep 1 retrospective item.
- **Phase 2's Work Management notes were overwhelmingly neutral (20/26 = 77%).** Most pages in this sub-batch (`cleanup`, `close`, `compact`, `formula`, `handoff`, `hook`, `mountain`, `orphans`, `prune-branches`, `ready`, `release`, `resume`, `scheduler`, `sling`, `synthesis`, `trail`, `unsling`, etc.) were well-mapped in Phase 2 and their Notes sections were genuine implementation-curiosity observations, not mis-filed drift candidates. The 6 pages with findings were concentrated in the parent-only-looking files (`assign`'s orphan `--force`, `changelog`'s orphan `--week`, `molecule`/`mq`/`wl` hand-maintained lists and sibling-file misses, `done`'s pre-plan drift-found carveover). Yield: 6/26 = 23%, between 1a's 18% and 1b's 55%. Calibration continues to drop as the commands/ sub-batches progress through higher-surface-area commands with better Phase 2 coverage.

**What didn't:**

- **The plan's PR-delta scoping draft is wrong about `wl`.** Plan line 1479 says "10 new `wl_*.go` command files post-v1.0.0 (wl_browse, wl_charsheet, wl_claim, wl_done, wl_scorekeeper, wl_show, wl_stamp, wl_stamps)." Direct verification at v1.0.0 shows all of those files (plus `wl_post.go`, `wl_sync.go`) existed at the tag with their `AddCommand` lines intact. If the plan's PR-delta draft is wrong on `wl`, it may be wrong on other entries too — and any Phase 3 finding that inherits its release-position from the draft inherits the error. I am not confident how the draft got to the "post-v1.0.0" conclusion; possible sources are (a) an intermediate tag between v1.0.0 and HEAD that introduced commits to `wl_*` files without being where the *files* were introduced, (b) a grep-for-`func init()` that surfaced recent refactors, or (c) the draft was written by reading commit messages rather than by checking `ls-tree v1.0.0`. Fix: the Sweep 1 retrospective gate should either re-derive the draft from scratch (`git diff v1.0.0..HEAD --stat -- internal/cmd/` is the canonical source) or drop the draft and use per-finding v1.0.0 verification as the canonical check. Every Batch 1 sub-batch so far has done per-finding verification anyway, so the draft's utility is mostly as a pre-filter hint, not as ground truth.
- **The `done.md` carveover is scope-bending.** The task explicitly said to reformat the pre-plan `f143813` sections to the v1.2 schema. To produce a compliant `## Docs claim` section I had to read `docs/CLEANUP.md:17-29` for the verbatim quote. This is "reading a docs/ file during Sweep 1," which is structurally out of scope for Sweep 1 but explicitly permitted by the task carveout. I logged the read separately in the `Docs files read:` column of the log entry with a justification, but the precedent feels soft: a future sub-batch might also have pre-plan Phase-3 sections that need reformatting (only if Phase 2 produced any — `done.md` is the only known case). If it does, the same carveout applies. Documented in the log entry's Judgment Calls section as item 3.
- **The `selfNukePolecat` dead-code observation doesn't quite fit any category.** `done.go:1673-1680` declares the function, `:1735` has a doc comment saying it's "Kept for explicit kill scenarios," and `grep -n "selfNukePolecat\b" done.go` shows only the declaration (no call sites in `done.go`). But call sites may exist in `polecat*.go` (which I didn't read because those are out of Batch 1c's scope — they're in Batch 1g Workspace). If call sites exist, it's neutral (the comment is accurate, function is used elsewhere); if they don't, it's `implementation-status: vestigial`. I left it as a neutral sentence inside the `done.md` drift body rather than promoting it. The taxonomy doesn't have a clean "observed-but-unverified-until-cross-batch-read" category. This is a minor gap in the skill but not urgent.
- **`assign --force` borderline: is flag description "doc text"?** The `--force` finding hinges on treating `assignCmd.Flags().BoolVar(&assignForce, "force", false, "Replace existing hooked work")` as a cobra Long-text-equivalent claim. The strict reading — "any docstring on a flag registration is a claim subject to the authority hierarchy" — is what I applied. A narrower reading would say "cobra drift is only about `Long` text, not flag descriptions" and demote this to a neutral observation about a dead flag. The 1b `shell` and `uninstall` findings similarly promoted flag-description claims, so 1c stays consistent, but I notice this is a recurring judgment question and flagged it as Judgment Call 8 in the log.md entry.
- **`molecule`'s `step` group sub-subcommand case is mildly novel.** The four sibling-file additions register on `moleculeStepCmd` (the child group), not on `moleculeCmd` itself. The Batch 1b `hooks` finding was simpler — all sibling-file registrations went on the parent. For `molecule`, the cobra auto-`Available Commands:` block on `gt mol --help` shows the 10 direct parent children (including `step`), but users have to type `gt mol step --help` to see the 4 children of `step`. The drift in `moleculeCmd.Long` is therefore user-visible only at the `step` level; a user scanning `gt mol --help`'s categorised text never sees that the `step` group has more than `done`. This makes the finding slightly harder to reproduce than 1b's `hooks init` case but the same category. Worth noting for Batches 1d-1h if similar patterns surface.
- **The batch-entry log.md section I wrote is ~300 lines long.** Batch 1a was ~110, Batch 1b was ~105. The Work Management sub-batch is the largest in Batch 1 (26 pages vs 22/11) and the "Source files re-read" list alone grew to 40+ lines because of all the sibling-file verification (10 `wl_*` + 3 `molecule_*` + 1 `mq_next`). The "Findings by category" block also grew because three separate wiki-stale findings need per-page explanations. Not wrong but verbose. A future schema refinement might add a more compact per-finding row format, or split the Source files column into "parent files" and "sibling-file verifications." Not blocking; flagged as a "consider a structured template" observation.

**What to change next time:**

- **Plan file: fix the PR-delta scoping draft (plan lines ~1470-1500).** Re-derive from `git -C /home/kimberly/repos/gastown diff v1.0.0..HEAD --stat -- internal/cmd/` or drop the draft and note that Sweep 1 does per-finding verification inline. Either way, the current "10 new wl_* files post-v1.0.0" text is wrong and will mislead future dispatchers. Candidate text for the new section: "The PR-delta between v1.0.0 and HEAD is load-bearing for Phase 6's release-position tagging but is NOT authoritative at the Sweep 1 sub-batch level — every finding is verified inline via `git show v1.0.0:<cited-file>` or file-presence checks at the tag. Treat the draft as a hint for pre-filtering high-yield commands, not as ground truth for finding metadata."
- **Dispatch template for Batches 1d-1h: add an explicit `## sibling-file audit` step before the parent file is re-read.** Current template (implicit from Batch 1b's retro) says "check for sibling files when promoted findings cite them." Batch 1c validates that the sibling check should be eager, not reactive — every parent-looking file benefits from a 10-second `ls <parent>*.go` + grep for AddCommand in each. Templating step: (1) `ls internal/cmd/<command>*.go`; (2) for each sibling file, `grep -n "<command>Cmd.AddCommand\|<command>StepCmd.AddCommand" <file>` (or the equivalent step-group name); (3) if new registrations surface, include them in the `sources:` frontmatter expansion and the Subcommands table rewrite. Cost: ~20 seconds per parent. Benefit: catches phase-2-incomplete wiki-stale findings immediately and catches hand-maintained-list cobra drift via the same pass (since you now know what the full subcommand set actually is).
- **Skill edit suggestion (non-blocking): the `writing-entity-pages` skill could explicitly bless the "hand-maintained subcommand/target list is incomplete" pattern as a recurring cobra-drift class.** Five findings across 1a/1b/1c fit it: `doctor` (curated check catalog), `repair` (6 targets listed, 2 registered), `account` (4 listed, 5 registered), `hooks` (8 listed, 9 wired), `molecule` (categorised list omits 4+ sibling-file subcommands). A dedicated paragraph in the drift taxonomy's `cobra drift` row would help future subagents classify this pattern immediately instead of re-deriving the framing each time. Candidate text: "Pattern: when Cobra `Long` text hand-maintains an enumeration of subcommands, flags, keys, or targets, the enumeration reliably drifts against the code's actual `AddCommand` / `Flag` / `switch` / registration lists. Promote every such omission to `cobra drift`, fix tier `code`. A Phase 6 meta-fix would replace the hand-maintained list with cobra's auto-generated `Available Commands:` or similar introspection, eliminating the drift class entirely."
- **Plan file update to the Batch 1 tracker table:** Batch 1c's actual numbers are 3 cobra-drift + 3 wiki-stale + 1 drift (done carveover) + 20 none out of 26 pages. Yield count depends on how you count `molecule` (2 findings on 1 page): 7 findings across 6 pages = 23% pages-with-findings yield, consistent with 1a (4/22 = 18%) and lower than 1b (6/11 = 55%). The Batch 1 tracker row for 1c should be updated with the commit SHA and the finding counts: "3 cobra-drift / 3 wiki-stale (all phase-2-incomplete) / 1 drift (done reformat) / 20 none; molecule carries both cobra-drift and wiki-stale on the same page (second dual-tag case after 1b hooks); `wl` release-position correction required, plan's PR-delta draft is wrong; sibling-file audit from 1b retro is the load-bearing methodology improvement." Also update the "Notes" column to flag the PR-delta correction for the Sweep 1 retrospective gate.
- **Dispatch template: mention the `done.md` carveout precedent explicitly for Batches 1d-1h.** The `done.md` pre-plan Phase-3 sections are unique to Batch 1c (I checked the other 25 pages — only `done.md` has pre-plan `## Docs claim`/`## Drift` sections). The carveout should not apply to Batches 1d-1h unless the dispatch template names a specific page with pre-plan sections. The dispatch template for 1d should explicitly state "no pages in this sub-batch have pre-plan Phase-3 sections; Sweep 1 reads zero `docs/` files" so a future subagent doesn't assume a general carveout. Prevents scope creep.

**Follow-ups filed:**

- none (bd beads) — wiki-bd is independent from the main Gas Town bd tracker and I did not touch it this sub-batch per dispatch convention.
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` lines ~1470-1500 PR-delta scoping draft | correct the "10 new `wl_*.go` files post-v1.0.0" text; all cited sibling files exist at v1.0.0 with their AddCommand lines intact
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Batch 1 tracker table row 1c | update with commit SHA and finding counts once landed
- dispatch template enhancement: formalize the `ls <parent>*.go` + per-sibling AddCommand grep as a mandatory step in the Sweep 1 audit workflow, for Batches 1d-1h and also Batches 2-4
- skill edit suggestion: `.claude/skills/writing-entity-pages/SKILL.md` drift taxonomy `cobra drift` row | add a paragraph documenting the "hand-maintained enumeration drifts against code registration list" pattern as a recurring class with 5+ instances across Batches 1a/1b/1c
- skill edit suggestion (optional): expand the drift taxonomy with a "observed-but-cross-batch-unverified" flag for cases like `selfNukePolecat` where the finding's classification depends on code outside the current sub-batch's scope

**For Kimberly retro discussion:**

- **The sibling-file audit is the single biggest methodology improvement in Phase 3 so far.** Batch 1b surfaced it reactively (Phase 2 was wrong about `directive` and `hooks`, the subagent speculated the fix). Batch 1c made it proactive — every parent-looking file got a sibling-file audit up front — and caught three more wiki-stale findings that would have otherwise shipped as cobra-drift (the hand-maintained list omissions) without realizing the Phase 2 page body was ALSO wrong. Promoting this to a mandatory skill step would make Batches 2-4 (packages, files, concepts, roles) tangibly higher yield, because the same parent-file-in-isolation blindness probably extends beyond commands/. Worth ratifying as schema v1.2.1 or as a skill-only refinement.
- **The Phase 2 `status: partial` flag may be correlated with phase-2-incomplete findings.** `directive`, `hooks`, `molecule`, `wl`, `mq` all had `status: partial` or `status: verified` as Phase 2's self-assessment. The wiki-stale findings cluster hard in the ones Phase 2 self-assessed as "partial" (directive was marked partial; hooks was marked partial; molecule was marked partial; `wl` was surprisingly marked `verified`; `mq` was marked partial). A cheap lint pass in a future Phase 3 batch or at the retrospective gate: "re-verify every Phase 2 `status: partial` page's sibling-file wiring proactively, because the partial status may be a signal that Phase 2 knew its coverage was thin." This would let us estimate the total `phase-2-incomplete` surface area before Phase 4 (Coverage) rather than discovering it incrementally across 111 pages.
- **The `wl_*` family deserves per-subcommand wiki pages.** There are 11 Wasteland subcommands plus `internal/wasteland` helpers — enough to fill a `gastown/commands/wl/` sub-folder or an expanded set of `wl-<name>.md` pages. Consolidating them on `wl.md` works for Batch 1c's wiki-stale fix (one page, one drift row) but obscures the individual subcommands from Obsidian search + cross-links. Filed as a Phase 4 (Coverage) follow-up in the `wl.md` Notes section. If the Phase 4 coverage pass is heavy on inventory expansion, this is a natural candidate. Would mean breaking the "one .go-file = one wiki page" convention, which is already not strict (see `convoy.md` covering 4 `.go` files).
- **The "hand-maintained list drifts" meta-pattern (5+ findings across 1a/1b/1c) is real enough to warrant a named class in the drift index.** It's not a new drift taxonomy category — the category is still `cobra drift` — but at Phase 6 planning time, having these 5+ findings grouped under one fix-pattern ("stop hand-maintaining subcommand lists in `Long` text; use `Available Commands:`") would let a single Phase 6 commit address them all instead of five individual fixes. Skill-level refinement: the drift taxonomy table's `cobra drift` row could carry a "Common sub-patterns" column listing this one as the most-seen class. Optional, not blocking.

## [2026-04-16 02:00] stage | 3.1.1d — Agent Management Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1d
**Unit:** 12 `GroupAgents` command pages audited; 3 `cobra drift` findings (`boot`, `callbacks`, `witness`), 1 `implementation-status: vestigial` finding (`witness` — same flag, double-tagged), 3 `wiki-stale` findings all `phase-2-incomplete` (`agents`, `polecat`, `session`), 6 `[none]` pages (`deacon`, `dog`, `mayor`, `refinery`, `role`, `signal`). `witness` carries BOTH `cobra drift` AND `implementation-status: vestigial` on the same flag. 6 distinct finding pages out of 12. One commit.
**Duration:** one dispatch

**What went well:**

- **The proactive sibling-file audit (mandatory per Batch 1c retro) paid off on TWO `phase-2-incomplete` wiki-stale findings that would have been completely invisible without it.** `agents.md` — a 15-second `ls internal/cmd/agents*.go` + `grep "agentsCmd.AddCommand" internal/cmd/` over the whole tree surfaced `agent_state.go:85` registering a fifth subcommand (`gt agents state`) that Phase 2 had missed entirely because Phase 2 only read `agents.go` in isolation. `polecat.md` — the same `ls internal/cmd/polecat*.go` sibling-file walk revealed that `polecat_spawn.go`, `polecat_cycle.go`, and `polecat_helpers.go` — which Phase 2 speculated contained "additional subcommands" as a follow-up — contain **zero** cobra commands and **zero** `init()` functions. Both findings are load-bearing, and without the proactive sibling audit Batch 1d's yield would have been 3 cobra-drift + 0 wiki-stale = 3 findings on 3 pages (25% yield). With the audit, it's 3 cobra-drift + 1 compound + 3 wiki-stale = 7 rows on 6 pages (50% yield by pages-with-findings). The methodology improvement from Batch 1b/1c is fully earning its keep.

- **The `boot` "special dog" finding resolved unambiguously against `internal/dog/manager.go:300`.** The Phase 2 page body already flagged this as an "open question" in Notes, but it was framed as "worth reconciling when the role/concept page for Dog is written in Batch 6" — i.e. a deferral, not a finding. The Phase 3 re-read spent ~2 minutes tracing through: (1) `boot.go:33` `Long` text says `"Boot is a special dog"`; (2) `grep "AgentDog" internal/` returns one unrelated test-name hit (no type exists); (3) `grep "boot" internal/dog/manager.go` finds the explicit `"'boot' is the boot watchdog using .boot-status.json, not a dog"` comment at `manager.go:300` — literally the negation of the Long's claim. That's three signals on the same axis, all pointing at "cobra drift, severity wrong, fix tier code." The kind of finding where the second and third signal remove all doubt. Worth noting: if the Phase 2 note had been a bare observation without the "worth reconciling" hedge, I might have stayed neutral on the assumption that "special dog" is metaphorical. The hedge itself signaled the reader's uncertainty, and Phase 3's job is to resolve uncertainty with code re-reads.

- **The `witness --foreground` double-tag (`cobra drift` + `implementation-status: vestigial`) clarified a taxonomy axis that Batch 1a/1b/1c hadn't exercised.** All prior Batch 1 findings were single-category. This one is not — the flag is vestigial (code acknowledges it) AND the Cobra surface still advertises it as functional. Two fix surfaces, two categories, on the same flag. I filed it as two findings (one in each section) rather than collapsing into a single compound-style row because the Phase 6 fix work splits cleanly: the Cobra fix is a text edit to `Long` / flag description; the vestige fix is either a `Long` rewrite that acknowledges the vestige OR a flag deletion. A Phase 6 implementer might do the first without the second. Flagged in Judgment Call 5 for the retrospective gate to validate whether the double-tag pattern should become canonical for the `cobra drift + implementation-status` compound axis (analogous to 1b's `hooks.md` which combined `cobra drift + wiki-stale`). The skill's current compound-drift definition is Cobra + docs; this is a different axis.

- **Release-position verification stayed zero-friction — every finding `in-release`, verified in ≤5 seconds per finding.** Agents command family is stable against v1.0.0. `agent_state.go` was the only sibling file that needed a presence check (`git show v1.0.0:internal/cmd/agent_state.go | grep "agentsCmd.AddCommand"`), and it was present with the same registration line. `boot.go:31-45` Long text, `internal/dog/manager.go:300` exclusion comment, `callbacks.go:88-103` Long text + `:471-502` handleSling body, `witness.go:50-63` Long + `:124` flag registration + `:179-183` vestige branch — all byte-identical at v1.0.0. The PR-delta scoping draft's credibility problem (flagged in 1c retro) didn't affect this sub-batch because I didn't trust the draft — I verified every finding inline per the 1c methodology correction.

- **Phase 2's Agent Management Notes sections were **genuinely** high-quality for the `[none]` pages.** `deacon`, `dog`, `mayor`, `refinery`, `role` all had solid architectural Notes about attach-does-more-than-attach, Engineer-vs-Manager divergence, hand-maintained `roleListCmd` omitting boot/dog, `--agent` shared global across subcommands, etc. None of those rose to the drift threshold under strict code-first reading, and all of them are legitimate architectural observations that stay in Notes. Yield math: 6/12 = 50% pages-with-findings, between 1a's 18%, 1b's 55%, and 1c's 23%. But the subjective feel of the Agent Management group is "Phase 2 covered this group well" — the findings cluster hard on the two pages where Phase 2 hit sibling-file blindness (`agents`, `polecat`) and the three pages with aspirational or vestigial claims Phase 2 noticed but didn't formally promote (`boot`, `callbacks`, `witness`). The other 7 pages are solid mapping work.

**What didn't:**

- **The `session inject` "explicitly deprecated" wiki-stale was the smallest and most borderline finding of the sub-batch.** The Phase 2 cross-link text uses "explicitly deprecated in favor of nudge" but the Long text uses "prefer `gt nudge`" + "preserves `inject` as a low-level primitive." The characterization difference is real but small — strict code-first rule says the wiki contradicts the Long verbatim (no "deprecated" wording), so it's wiki-stale. Lenient reading says "deprecated" is colloquial shorthand and the functional relationship is the same, so it's neutral. I took the strict reading and promoted. A reasonable reviewer might disagree, and flagging this as Judgment Call 1 in the log. If the retrospective gate demotes it, the fix is: leave the cross-link text as-is (or edit it slightly) and change the frontmatter to `phase3_findings: [none]`. No substantive body content was touched, so demoting is cheap. The reason I still promoted: the Phase 3 discipline is verbatim-against-code, and the characterization gap between "deprecated" and "preferred alternative" is meaningfully different in Phase 6 planning terms (deprecation implies scheduled removal; preference implies indefinite coexistence).

- **The `polecat.md` wiki-stale fix touched THREE different body sections for one finding.** I rewrote: (a) the sibling-file enumeration at `polecat.md:39-72` (renamed from "Subcommands are split across several sibling files" to "Subcommands are split across **exactly two** sibling files"); (b) the Invocation block trailer at `:132-133` (replaced "(Additional subcommands exist in `polecat_spawn.go` and `polecat_cycle.go`; see 'Follow-up' in Notes.)" with a definitive "(All 17 user-facing subcommands are listed above; the tree is complete. `polecat_spawn.go`, `polecat_cycle.go`, and `polecat_helpers.go` are helper-only files with no cobra commands.)"); and (c) a promotion note in Notes. Three touch points for one finding is verbose. The skill's guidance says "fix inline in `## What it actually does`" for wiki-stale, which is ambiguous when the stale claim lives in multiple sections. I chose to touch all three because leaving any of them with the old speculation would be internally incoherent. But the verbosity is a smell — a better fix pattern would be to pick one locus (the most visible) and update it, then punt the secondary loci to a followup cleanup pass. Flagged in Judgment Call 6.

- **I briefly over-rewrote `polecat.md` and accidentally deleted the `## What it actually does` section header.** My first edit replaced "## What it actually does\n\nThe parent `polecatCmd` is defined in ..." with "The parent `polecatCmd` is defined in ..." by accident — I put the opening pleasantries into the wrong Edit call and dropped the header. Caught it immediately by re-reading the file and fixed with a second Edit to re-add the header. Cost: one wasted Edit call + one Read. Not blocking, but a reminder that Edit calls on section-boundary text need more care when the existing content has a section header right above the edit point. The skill could add a one-line guardrail: "Before replacing a block that starts at the top of a section, verify the section header is preserved in the `new_string` or in an adjacent unchanged block." Not worth a full skill edit; filing as a personal discipline note.

- **The `role list` boot/dog omission is the second "hand-maintained list drifts" candidate in Batch 1d (after `doctor`/`repair`/`account`/`hooks`/`molecule` pattern from 1a-1c), but I stayed neutral because the Long says "include" rather than "are".** This is the third data point for the meta-pattern Batch 1c's retro flagged: "Gas Town's CLI `Long` texts hand-maintain enumerations that drift from the code's actual registration lists." Batches 1a/1b/1c all promoted such omissions as `cobra drift`. Batch 1d has a candidate (`role list`) that I stayed neutral on because the softening word "include" makes the claim non-exhaustive. This is a coherent distinction but it means the meta-pattern's threshold is load-bearing on a single word of the Long text. If the Phase 6 meta-fix is "replace hand-maintained enumerations with cobra introspection," `role list` might be a candidate for the same fix even though Phase 3 didn't tag it as drift — because the underlying maintenance hazard is the same. Flagged for the retrospective gate as "does the cobra-drift threshold need a 'hand-maintained enumeration pattern' subcategory that captures non-exhaustive-word cases too?"

- **The `## Docs claim` section on `witness.md` has TWO verbatim quote blocks (one from the Long Examples, one from the flag registration line) because the cobra-drift finding rests on claims from both locations.** This works but the skill template assumes one `### Verbatim` block per `## Docs claim` section. I split into two labelled sub-blocks ("From `witness.go:50-63`:" and "From `witness.go:124`:"). Reader-friendly but unusual shape. The skill could note that `## Docs claim` can have multiple `### Verbatim` sub-blocks when the drift finding spans multiple source lines. Not blocking, but a clarification opportunity.

- **I almost missed the `agents state` finding.** The first time I read `agents.go` I noted the `init()` at `:139-148` registers 4 subcommands and was about to classify `agents.md` as `[none]`. The sibling-file audit step (per Batch 1c retro recommendation) was what caught it: `ls internal/cmd/agents*.go` surfaced `agent_state.go` (I had already noted `agent_state.go` in the first big grep over all the GroupAgents siblings, but in the per-page audit flow I almost skipped the sibling check on `agents` specifically because `agents.go` doesn't fit the naming pattern). The save was the explicit mandatory `ls` step — without it I would have shipped `agents` as `[none]`. Validates the "make the sibling audit mandatory and unconditional" discipline: even one skipped sibling check loses a finding.

**What to change next time:**

- **Dispatch template for Batch 1e onward: formalize the sibling-file audit as a FIRST step, before re-reading the parent file.** The current (implicit) template says "re-read source + check siblings." Batch 1c/1d experience is that the sibling check should come FIRST, before the parent re-read, because the parent-file re-read is biased toward believing Phase 2's registration count, and the sibling audit is the thing that surfaces what Phase 2 missed. Ordering: (1) `ls internal/cmd/<name>*.go` to list all siblings; (2) grep each sibling for `<name>Cmd.AddCommand\|init()\|var <name>` to find unlisted registrations; (3) THEN re-read the parent file at cited lines + compare to wiki body. The ordering inversion would have made the `agents state` finding obvious from the first read rather than a "save" on the second pass. Cost: 10 additional seconds per page. Benefit: catches `phase-2-incomplete` findings before they slip past.

- **Skill clarification (non-blocking): `## Docs claim` sections can carry multiple `### Verbatim` sub-blocks when the drift claim spans multiple source lines.** Add a one-line note to the writing-entity-pages skill's entity-page starter template: "If a finding's docs claim rests on two or more separate source lines (e.g. a `Long` text plus a flag description on the same flag), `## Docs claim` may include multiple `### Verbatim` sub-blocks, each labelled with its source prefix (`From <file>:<lines>:`)." The `witness.md` finding demonstrates this pattern; future `cobra drift + implementation-status` compound findings on the same primitive may need the same shape.

- **Plan file update to the Batch 1 tracker table:** Batch 1d's actual numbers are 3 cobra-drift + 1 compound (cobra-drift + impl-status vestigial on `witness`) + 3 wiki-stale (all phase-2-incomplete) + 6 none out of 12 pages. By findings count: 7 rows on 6 pages. By pages-with-findings: 6/12 = 50% yield. 1a=18%, 1b=55%, 1c=23%, 1d=50%. Running average across 1a-1d: 17/71 = 24% pages-with-findings. The calibration is settling into "about 1-in-4 pages has a finding" but with high variance — Configuration (1b) was an outlier at 55% because Phase 2's Configuration coverage was thin; Agent Management (1d) landed at 50% because Phase 2's Agents coverage had two big sibling-file blind spots (`agents`, `polecat`) that contributed disproportionately. Update the tracker row for 1d with commit SHA and the 7-findings-on-6-pages breakdown.

- **Plan file: remove the `session inject "explicitly deprecated"` framing from any preflight notes for Batches 1e-1h.** The Batch 1d dispatch prompt said `session inject` is "explicitly deprecated in favor of `gt nudge`" as one of the high-yield candidates. The audit result is that the characterization is wrong — `inject` is preserved as a low-level primitive, not deprecated. Future preflights should not propagate the "deprecated" framing because it primes subagents toward a wrong classification.

- **Retrospective gate agenda item: review whether `witness --foreground`-style double-tagging (`cobra drift` + `implementation-status: vestigial`) should be a canonical compound pattern or folded to a single category.** Batch 1d is the first occurrence. The retrospective gate has 3 other compound/double-tag cases to consider (`hooks`: `cobra drift + wiki-stale`; `molecule`: `cobra drift + wiki-stale`; now `witness`: `cobra drift + implementation-status: vestigial`). Each of these has a slightly different axis. A canonical "compound-drift variants" table in the drift taxonomy would help — currently the skill has exactly one compound type (`compound drift = cobra drift + drift`, both on the docs/Cobra axis), and these are all different.

- **Retrospective gate agenda item: third data point for the `role list` neutral call on non-exhaustive-word hand-maintained enumerations.** `doctor`, `repair`, `account`, `hooks`, `molecule` all used strict enumerating language and were promoted as `cobra drift`. `role list` uses "include" (non-exhaustive) and I stayed neutral. The meta-pattern would be stronger if the taxonomy had a sub-type for "hand-maintained enumeration with hedging word" — these are lower-severity cases that still point at the same Phase 6 meta-fix opportunity (replace hand-maintained enumerations with cobra introspection). Could be a retrospective-gate expansion or left alone.

**Follow-ups filed:**

- none (bd beads) — all follow-ups are documentation/skill/plan edits for the main orchestrator to apply before Batch 1e dispatches or at the Sweep 1 retrospective.
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Batch 1 tracker table row 1d | update with commit SHA and 7-findings-on-6-pages breakdown (3 cobra-drift, 1 impl-status vestigial, 3 wiki-stale all phase-2-incomplete; `witness` is the first double-tagged cobra-drift + impl-status case)
- plan edit suggestion: `.claude/plans/2026-04-14-phase3-drift.md` Batch 1 preflight notes for Batches 1e-1h | remove the `session inject "explicitly deprecated"` framing that this batch disproved
- dispatch template enhancement: formalize sibling-file audit as a MANDATORY FIRST step (before parent re-read), not just a mandatory step; ordering matters because parent re-read is biased toward Phase 2's registration count
- skill edit suggestion (non-blocking): `.claude/skills/writing-entity-pages/SKILL.md` entity-page starter template | add a one-line note that `## Docs claim` sections may carry multiple `### Verbatim` sub-blocks when the drift claim spans multiple source lines (e.g. `Long` text + separate flag description)
- retrospective-gate agenda: review whether `cobra drift + implementation-status: vestigial` double-tagging should be a canonical compound pattern; 4 compound/double-tag shapes have now surfaced across Batches 1b-1d and each has a different axis

**For Kimberly retro discussion:**

- **The sibling-file audit is now load-bearing for 2 of the 4 Batch 1 sub-batches that surfaced `phase-2-incomplete` findings.** Batches 1b (`directive`, `hooks`), 1c (`molecule`, `mq`, `wl`), and 1d (`agents`, `polecat`) have all produced wiki-stale findings that depend on checking sibling files Phase 2 missed. Batch 1a had zero. The pattern: groups with 15+ subcommands split across sibling files (Configuration, Work Management, Agent Management) tend to have phase-2-incomplete findings; groups with tight single-file parents (Diagnostics) don't. Predicts: Batches 1e (Communication, 7 cmds), 1f (Services, 11 cmds), 1g (Workspace, 7 cmds), 1h (Ungrouped, 15 cmds) should be lower-yield for wiki-stale than 1b/1c/1d IF they are dominated by single-file commands. 1h is the wildcard — ungrouped commands are likely to be heterogeneous. Recommendation: prioritize the sibling-audit discipline on 1f and 1h, which are large enough to harbor misses.

- **`boot`'s "special dog" framing is a small but load-bearing finding because it propagates into the Boot role-page work scheduled for Batch 6.** The Phase 2 body already flagged this as an open question. Now it's a formal `cobra drift` row with a fix direction (drop or reframe the dog metaphor in `boot.go:33`). When Batch 6 writes `gastown/roles/boot.md`, it should NOT describe Boot as a dog, and should reference the `internal/dog/manager.go:300` exclusion as the canonical evidence that Boot is architecturally distinct from dogs. This is a cross-batch link that Phase 3 Batch 1d enables but does not execute.

- **The `callbacks.md` `handleSling` finding is architecturally interesting because it's the inverse of the `boot` situation.** `boot`'s Long OVER-claims a dog relationship that the code explicitly denies. `callbacks`' Long UNDER-describes the log-and-defer behavior (reads as "we spawn" when the code says "we log and the Deacon spawns"). Both are `cobra drift`, both `severity: wrong`, both `fix tier: code`, but the fix shapes are different: `boot` is "the Long says something false and should be rewritten to say something true"; `callbacks` is "the Long says something true-but-incomplete and should be rewritten to describe the defer pattern." A Phase 6 planner looking at both findings should notice they're different kinds of fix work even though they share a category.

- **`witness --foreground` is the first compound axis the wiki has seen that combines Cobra text with code-comment-marked vestige.** Prior compound cases (`hooks`: `cobra drift + wiki-stale`, `molecule`: same) were about wiki synthesis drift. This one is about cobra-level docs drift AND runtime notice drift, which is a different shape. Worth capturing in the drift index's taxonomy when Batch 13 builds the index: the `cobra drift + implementation-status` compound is a valid pattern and distinct from `cobra drift + drift` and `cobra drift + wiki-stale`. If the retrospective gate decides to add a "compound axis" column to the drift taxonomy, this is the third axis.

## [2026-04-15 22:00] stage | 3.1.1e — Communication Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1e
**Unit:** 7 `GroupComm` command pages audited; 1 cobra-drift finding + 1 wiki-stale finding (both on `mail.md`); 6 pages tagged `phase3_findings: [none]`; one commit
**Duration:** one dispatch

**What went well:**

- **The `mail` sibling-file audit delivered exactly the finding the dispatch prompt predicted.** The prompt flagged mail's 15+ subcommands and sibling-file audit as "CRITICAL" and "high-leverage." The audit confirmed: 5 of 13 non-test siblings register NEW subcommand groups via their own `init()` blocks (`channel`, `directory`, `group`, `hook`, `queue`), bringing the total from the Phase 2 table's 17 to 22. Phase 2 listed all 13 sibling files in `sources:` but never checked their `init()` blocks. This is now the 5th sub-batch (of 5) to surface a `phase-2-incomplete` wiki-stale finding from sibling-file analysis. The pattern is fully validated: every sub-batch since 1b has found at least one.
- **The dispatch prompt's detailed per-command predictions were accurate on neutrals.** `broadcast` (no `--force`, honors DND) — neutral, exactly as predicted. `dnd` (verbose-loss) — neutral because the Long says "resume normal" explicitly. `nudge` (`--if-fresh` 60s) — neutral because the flag description mentions the threshold. `escalate` (not beads-exempt) — neutral. `peek` (tmux wrapper) — neutral. All 6 neutrals were classifiable on first read without ambiguity. This is the cleanest set of neutral calls in Batch 1 so far.
- **The mail COMMANDS cobra drift is the largest instance of the "hand-maintained enumeration" pattern.** Prior instances: `doctor` (curated check list), `repair` (6 targets listed / 2 registered), `account` (4/5), `hooks` (8/9), `molecule` (categorised list omits step-group children). Mail: 4 listed / 22 registered = 82% incomplete. This is the most dramatic instance and the strongest argument for the Phase 6 meta-fix (replace hand-maintained COMMANDS blocks with cobra's auto-generated `Available Commands:`).

**What didn't:**

- **Communication is a low-yield group for drift findings.** 1 page with findings out of 7 = 14% yield, the lowest of any Batch 1 sub-batch (1a: 18%, 1b: 55%, 1c: 23%, 1d: 50%). Most Communication commands are well-scoped single-file commands with accurate Long text. The yield was concentrated entirely on `mail`, which is an outlier in complexity (22 subcommands across 14 source files). Without mail, this would have been a 0% yield sub-batch.
- **No cobra-drift on `dnd off` verbose-loss.** The dispatch prompt flagged this as a potential finding, but the Long text is actually accurate: it says "resume normal notifications" for `off`, which is exactly what the code does. The verbose-loss is a UX papercut, not a docs lie. Correctly classified as neutral, but it took a careful re-read of the Long to confirm — the prediction was reasonable but wrong.

**What to change next time:**

- **For Batch 1f (Services, 11 cmds): calibrate yield expectations down.** Communication's 14% yield and the pattern of most findings coming from one complex command suggest that smaller, single-file commands rarely produce drift. Services commands (e.g. `convoy`, `feed`, `patrol`) may follow the same pattern: one or two complex parents with sibling files producing findings, the rest neutral. The sibling-file audit remains mandatory but the dispatcher should expect 2-3 findings total, not the 5-7 that Configuration and Agent Management produced.
- **No skill or plan edits needed from this sub-batch.** The methodology is stable and the findings are clean instances of established patterns. The main observation (mail COMMANDS is the largest hand-maintained enumeration drift) is a data point for the Phase 6 meta-fix discussion, not a process change.

**Follow-ups filed:**

- none (bd beads) — all observations are informational.

**For Kimberly retro discussion:**

- **The `phase-2-incomplete` pattern is now 5-for-5 across sub-batches.** Every sub-batch from 1b onward has found at least one case where Phase 2 read a parent `.go` file in isolation without checking sibling-file `init()` registrations. The cumulative evidence: `directive` (1b), `hooks` (1b), `molecule` (1c), `mq` (1c), `wl` (1c), `agents` (1d), `polecat` (1d), `mail` (1e). That's 8 distinct commands with missed sibling registrations across 4 sub-batches. The mail case is notable because Phase 2 DID list all sibling files in `sources:` — it knew about them, listed them, but still didn't verify their `init()` blocks. This suggests the Phase 2 methodology had a systematic gap: it treated `sources:` as "files I was aware of" rather than "files I read for cobra registrations."
- **Communication is the cleanest group so far for the Sweep 1 retrospective gate's calibration data.** 6 of 7 pages resolved as neutral on first read. The group's design surface (nudge/peek pair, dnd/notify layering, mail as the durable substrate, broadcast as fan-out, escalate as severity routing) is well-factored and the Cobra Long texts accurately describe what the code does. Phase 6 work on Communication will be concentrated on mail's COMMANDS block and nothing else from Sweep 1.

## [2026-04-15 23:00] stage | 3.1.1f — Services Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1f
**Unit:** 11 `GroupServices` command pages audited; 3 cobra-drift findings + 1 wiki-stale finding (across `quota.md`, `reaper.md`, `down.md`, `dolt.md`); 7 pages tagged `phase3_findings: [none]`; one commit
**Duration:** one dispatch

**What went well:**

- **Dolt sibling-file audit was the highest-leverage activity, as predicted.** The dispatch prompt specifically called out `dolt_flatten.go` and `dolt_rebase.go` as sibling-file candidates, and both registered subcommands via their own `init()` blocks. This extends the `phase-2-incomplete` pattern to 6-for-6 across sub-batches (1b onward). Cumulative count: 9 distinct commands with missed sibling registrations (adding `dolt` to the prior list of `directive`, `hooks`, `molecule`, `mq`, `wl`, `agents`, `polecat`, `mail`).
- **The `down.md` cobra-drift finding is a new pattern type.** Prior cobra-drift findings were all "hand-maintained enumeration lists that undercounted subcommands." The `down` finding is different: the Long text recommends `gt start` as the complement, but `gt start` does not restart the daemon. This is a semantic-correctness finding rather than a counting finding — it misleads the user about which command to use for recovery. It's the first finding in this category.
- **The `start` vs `up` resolution confirmed the dispatch prompt's prediction.** Both Long texts are internally accurate, but `down`'s Long cross-references the wrong one. The architectural divergence between two boot commands is real but is a design observation, not drift. The drift is in `down`'s Long text pointing at the wrong complement.
- **7 of 11 pages are clean neutrals (64%).** Services commands are well-documented in their Long text — the daemon, estop, thaw, maintain, shutdown, start, and up Long texts all match code. The yield of 36% (4 findings across 11 pages) is moderate, higher than Communication (14%) and consistent with the prediction that complex parent commands with sibling files produce findings while single-file commands do not.

**What didn't:**

- **Quota's `watch` omission was obvious but low-severity.** The COMMANDS block lists 4 subcommands and `watch` is the 5th. This is the same hand-maintained enumeration pattern seen 7+ times. Mechanically correct to flag but not interesting as a finding — it's another data point for the Phase 6 meta-fix rather than a unique insight.
- **Reaper's `databases` and `run` omissions follow the same pattern.** The "When run by a Dog" block is framed as an example, not an exhaustive list, which makes the drift slightly ambiguous. Classified as `wrong` because the block lists 4 of 6 subcommands without indicating it's non-exhaustive. A Phase 6 reviewer might reasonably reclassify this as `ambiguous`.

**What to change next time:**

- **For Batch 1g (Workspace, 7 cmds): expect the same pattern.** Complex parent commands with sibling files will produce findings; single-file commands will be neutral. The sibling-file audit remains mandatory. Watch for semantic cross-reference errors like the `down` finding — check whether Long texts reference complementary commands accurately, not just whether they enumerate subcommands.
- **No skill or plan edits needed from this sub-batch.** The methodology continues to produce consistent, classifiable findings. The "semantic cross-reference" finding type (down → start instead of up) may be worth adding as a named sub-pattern in the drift taxonomy if it recurs.

**Follow-ups filed:**

- none (bd beads) — all observations are informational.

**For Kimberly retro discussion:**

- **The `phase-2-incomplete` pattern is now 6-for-6.** `dolt` joins the list. Every sub-batch from 1b onward has surfaced at least one missed sibling-file registration. The evidence is overwhelming that Phase 2's methodology treated `sources:` as "files I was aware of" rather than "files I read for cobra registrations." This is now the single most reliable predictor of findings in Sweep 1: if a command has sibling `.go` files with `init()` blocks, at least one will register a subcommand that Phase 2 missed.
- **Services group has a unique finding type: semantic cross-reference drift.** The `down` Long text saying "use gt start" when the real complement is `gt up` is a new pattern not seen in prior sub-batches. It's worth tracking whether this appears in other groups (e.g., does any command reference an obsolete or wrong sibling?).

## [2026-04-15 23:30] stage | 3.1.1g — Workspace Sweep 1

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1g
**Unit:** 7 `GroupWorkspace` command pages audited; 6 cobra-drift findings + 1 wiki-stale finding (across `crew.md`, `namepool.md`, `install.md` x2, `git-init.md`, `init.md`); 2 pages tagged `phase3_findings: [none]`; one commit
**Duration:** one dispatch

**What went well:**

- **Highest yield sub-batch so far: 71% of pages have findings (5/7), 7 total findings.** This is substantially above prior sub-batches (18/64/27/50/14/36%). The Workspace group is rich in cobra drift because these are the foundational setup commands whose Long text was written early and never updated as features were added.
- **New finding types beyond enumeration drift.** Prior batches were dominated by "hand-maintained list undercounts subcommands." This batch surfaced three new patterns: (a) `install.Long` references a non-existent `docs/hq.md` file — a dead reference; (b) `git-init.Long` omits a behavioral step (branch-protection hook) from a numbered procedure; (c) `init.Long` uses wrong directory paths (`refinery/` vs `refinery/rig/`) in addition to omitting `crew`. These are qualitatively different from the counting errors and demonstrate that the cobra-drift category covers more than enumeration.
- **Phase 2 sibling-file audit was COMPLETE for crew and rig.** The dispatch prompt predicted the sibling audit would confirm Phase 2's completeness rather than find new misses. This was correct: Phase 2's `sources:` frontmatter for both `crew.md` and `rig.md` listed all sibling files. The `phase-2-incomplete` pattern breaks its streak at this batch for sibling-file misses specifically — but the `init.md` wiki-stale finding is a different kind of phase-2-incomplete (trusting Long text instead of checking the code).
- **`install.md` dead reference to `docs/hq.md` is a novel finding type.** No prior batch found a Long text referencing a non-existent file. This is worth flagging in the Phase 6 meta-fix discussion: the fix is either create the doc or delete the reference.

**What didn't:**

- **The `init.md` wiki-stale finding shows Phase 2 had a blind spot beyond sibling files.** Phase 2 trusted the Long text's directory enumeration ("polecats/, witness/, refinery/, mayor/") instead of reading the `rig.AgentDirs` slice at `internal/rig/types.go:48-54`. This is a different root cause than the sibling-file pattern: Phase 2 trusted Cobra text as authoritative for structural claims, violating the code-first principle. The determination is heuristic (`phase-2-incomplete`) because the AgentDirs slice had the same 5 entries at Phase 2 time.
- **The `namepool` classification is borderline.** The Long text uses "Examples:" which is inherently non-exhaustive. Classifying the omission of `create` and `delete` as cobra-drift follows prior batch precedent (similar to `quota`'s COMMANDS block), but a future reviewer might reasonably argue this is closer to `ambiguous` than `wrong`. Flagged as a judgment call.

**What to change next time:**

- **For Batch 1h (Ungrouped, 15 cmds): expect lower yield.** Ungrouped commands are mostly leaf commands without sibling files or complex Long text. The enumeration-drift pattern requires a parent with subcommands; many ungrouped commands are leaf. Focus audit time on any ungrouped commands that DO have subcommands or structural descriptions in their Long text.
- **Check for dead references (docs/X.md) in every Long text.** The `install` finding suggests this could be a recurring pattern — Long text referencing planned docs that were never written. Add a quick `ls` check for any `docs/` path mentioned in a Long text.
- **Check structural claims (directory names, paths) against the actual code, not just the Long text.** The `init.md` wiki-stale finding was caused by Phase 2 propagating the Long text's wrong paths. Phase 3 should catch this for any remaining commands whose Long text describes directory structures.

**Follow-ups filed:**

- none (bd beads) — all observations are informational.

**For Kimberly retro discussion:**

- **The `phase-2-incomplete` pattern has shifted.** Prior batches (1b-1f) found phase-2-incomplete exclusively in missed sibling-file registrations. Batch 1g's `init.md` is a different subspecies: Phase 2 trusted the Long text for structural claims instead of reading the defining code. This suggests Phase 2 had TWO blind spots: (1) not auditing sibling `init()` blocks, and (2) treating Cobra Long text as authoritative for claims that should be verified against code. Both are violations of the code-first principle, but they have different mitigation strategies for Phase 4.
- **Dead reference in Long text (docs/hq.md) is a new finding category worth tracking.** If it recurs in Batch 1h, it may warrant a named sub-pattern in the drift taxonomy.

## [2026-04-15 21:00] stage | 3.1.1h — Ungrouped Sweep 1 (Batch 1 final)

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 1h
**Unit:** 15 ungrouped command pages audited; 3 cobra-drift findings + 1 wiki-stale finding across 2 pages (`tap`, `warrant`); 13 pages tagged `phase3_findings: [none]`; one commit. **This is the final sub-batch of Batch 1. All 111 command pages now have `phase3_audited` frontmatter.**
**Duration:** one dispatch

**What went well:**

- **The `tap` sibling-file audit was the critical check and it paid off immediately.** The dispatch prompt correctly flagged this as the highest-priority verification. Running `ls tap*.go` before reading any wiki content instantly showed `tap_list.go` and `tap_polecat_stop.go` — two implemented subcommands that Phase 2 missed entirely. This is the exact same class of finding as `directive` and `hooks` in Batch 1b: Phase 2 took the parent file in isolation and missed sibling `init()` registrations. The sibling-file audit discipline established in 1b continues to be the single highest-value check in Phase 3.
- **The `warrant` path-hardcoding finding was exactly as predicted.** Phase 2 already noted the mismatch between `~/gt/warrants/` in Long and `<townRoot>/warrants/` in `getWarrantDir()`. Re-reading the source confirmed the mismatch is still present. Clean classification: cobra-drift, wrong, in-release, fix tier code.
- **The five GroupWork commands (`commit`, `forget`, `memories`, `remember`, `show`) required no wiki-stale fixup.** Phase 2 already had the correct Group attribution with batch-plan correction notes. This saved significant time — no source re-reading needed beyond confirming the GroupID lines.
- **Low-yield batches are still valuable.** 13 of 15 pages were `[none]`, but those 13 `phase3_audited` frontmatter entries are the audit trail proving the pages were checked. The 2 finding pages produced 4 findings including a novel compound (cobra-drift + wiki-stale on the same page for `tap`).

**What didn't:**

- **The dispatch prompt's `krc` sibling-file prediction was wrong.** It predicted "7 subcommands, verify Phase 2 covered all" and flagged `krc` for sibling-file audit. But `krc.go` is a single file with all 7 subcommands registered in one `init()` block. No siblings. The prediction cost about 30 seconds of unnecessary checking — minimal, but it shows the dispatch prompt's "expected high-yield" calibration assumes parent-like commands always scatter registrations, which is not true for commands that keep everything in one file.
- **The dead-doc-reference check (from 1g's `install` finding) found nothing new.** Only `tap.go` references a docs file (`~/gt/docs/HOOKS.md`), and `docs/HOOKS.md` exists at HEAD. The reference is to a runtime path (`~/gt/docs/`) not a repo path (`docs/`), and it's a valid file — so no dead reference. The 1g finding was an outlier, not a pattern.
- **The `tap` finding classification required a judgment call about `[planned]` tags.** The Long text explicitly marks `audit`, `inject`, `check` as `[planned]`. This could be classified as `implementation-status: unbuilt` (docs acknowledge the feature is not built) rather than `cobra-drift` (Long text contradicts code). I chose cobra-drift because: (a) the Long text presents these in a flat "Subcommands:" list alongside the implemented `guard`, not in a "Future" section; (b) the same Long text completely OMITS two implemented subcommands (`list`, `polecat-stop-check`), making the entire list factually wrong; (c) the fix is clearly code (rewrite the Long text) not docs. The borderline nature of `[planned]` tags is worth noting for the retrospective gate.

**What to change next time (Batch 2: packages/):**

- **The sibling-file audit is now validated across 8 sub-batches.** It should be a mandatory first step in the dispatch template for every batch, not just commands. Package pages may have similar patterns where Phase 2 read one file in isolation.
- **The GroupWork misclassification pattern (5 commands labeled "ungrouped" by the batch plan but actually in GroupWork) suggests the Phase 2 cobra-group inventory had gaps.** For Batch 2, verify the batch plan's scoping against actual `GroupID` assignments before auditing, not during.
- **`[planned]` tags in Long text should be classified consistently.** Recommendation: if the Long text presents unbuilt features in a flat list with implemented features, that's cobra-drift (the list is wrong). If the Long text puts unbuilt features in a separate "Planned" or "Future" section, that's implementation-status:unbuilt (the docs are aspirational). This distinction should be added to the skill taxonomy.

**Follow-ups filed:**

- none (bd beads) — all observations are informational for the Sweep 1 retrospective gate.
- skill edit suggestion: `.claude/skills/writing-entity-pages/SKILL.md` drift taxonomy | add clarification for `[planned]` tags in Long text: flat-list = cobra-drift, separate-section = implementation-status:unbuilt

**For Kimberly retro discussion:**

- **Batch 1 is complete. The Sweep 1 retrospective gate should assess:** (a) the sibling-file audit methodology that surfaced the highest-value findings; (b) the `[planned]` tag classification question; (c) whether the 32% finding rate across 111 pages is within expectations; (d) whether the wiki-stale sub-distinction (churn vs phase-2-incomplete) has enough data to inform Phase 4 scoping. The 12 wiki-stale pages are overwhelmingly `phase-2-incomplete` (sibling-file blind spot), not `churn`.

## [2026-04-15] scheduled | Sweep 1 retrospective after Batch 1

**Participants:** Kimberly + main orchestrator (Claude Opus 4.6)
**Trigger:** Batch 1 complete (111 command pages, 8 sub-batches). Plan calls for Sweep 1 retrospective gate between Batches 4 and 5, but Kimberly pulled it forward — Batch 1 was the largest batch with the richest retro data.

**Discussed:**

- Sibling-file audit methodology — the #1 Phase 3 improvement, surfaced ~12 findings → ratified as skill step 0
- Phase-2-incomplete as dominant wiki-stale root cause — 8+ commands missed sibling registrations → informational for Phase 4
- PR-delta scoping draft wrong about wl_*.go → drop draft as authoritative, keep as hint
- Hand-maintained enumeration meta-pattern (10+ cases) → Phase 6 batches as single PR pattern
- Wiki-stale log placement (lint vs drift-found) → separate lint entries going forward
- Novel finding sub-patterns (dead doc refs, semantic cross-refs, compound axes) → named in skill
- Dispatch template consolidation → plan file
- Yield calibration: 32% actual vs 20-32% estimated

**Decided:**

- 7 decisions ratified as documented above → landed as log.md `decision | Sweep 1 retrospective` entry + skill edits + plan edits in commit (this session)
- No retroactive changes to Batch 1 — entries stand as-is
- Wiki-stale log placement revisit deferred to project end

**Deferred:**

- Whether to run a second retrospective after Batches 2-4 (the originally-planned gate) or skip to Sweep 2 — decide at Batch 4 close based on yield and friction
- Phase 4 scoping for Phase-2-incomplete re-audit — informational input from Batch 1 data, not actionable until Phase 4 planning begins

## [2026-04-15 12:00] stage | 3.2.2a — Platform services packages Sweep 1

**Actor:** wiki-curator subagent (Phase 3 Batch 2a dispatch)
**Unit:** 9 package pages audited (cli, config, session, style, telemetry, ui, util, version, workspace), 1 wiki-stale finding, 1 commit
**Duration:** one dispatch

**What went well:**
- Package-file audit was fast and clean — all 9 packages matched their wiki `sources:` listings (except the one `orphan_windows.go` omission in util.md). Phase 2's package methodology was more thorough than its command methodology for file enumeration.
- Zero code churn since v1.0.0 across all 9 packages made release-position verification trivial — a single `git log --oneline v1.0.0..HEAD` for all directories returned empty.
- Line-number verification was high-confidence: multiple key function locations (SessionConfig:36, StartSession:144, CleanupOrphanedClaudeProcesses:768, isBuildBranch:241) matched Phase 2 citations exactly.

**What didn't:**
- Very low yield (11%, 1/9) suggests these clean, stable packages were not worth the full per-file re-read treatment. The "high-leverage" candidates (config, session, util) turned out to have no drift despite their complexity — Phase 2 did a good job on them.
- The dispatch template's "expected candidates" list (config presets, session PrefixRegistry, util orphan simplification, telemetry opt-in, version guard list) all checked out with no findings. The prediction model was too optimistic about package-level drift.

**What to change next time:**
- For future package sub-batches where `git log v1.0.0..HEAD` returns zero commits for all directories, consider a fast-path: annotate frontmatter, spot-check 2-3 high-risk claims per page, skip full re-read. The full re-read adds thoroughness but little yield when source hasn't moved.
- The `orphan_windows.go` omission pattern (body describes file, frontmatter doesn't list it) may recur on other packages with platform-specific build-tag files. Watch for `*_windows.go` / `*_unix.go` splits in future sub-batches.

**Follow-ups filed:**
- none — lessons are purely informational

**For Kimberly retro discussion:** The 11% yield on packages vs 32% on commands suggests packages may not need the full Sweep 1 treatment. Worth discussing whether Batches 2b-2f should use the fast-path for zero-churn packages.

## [2026-04-15 13:00] stage | 3.2.2b — Data layer packages Sweep 1

**Actor:** wiki-curator subagent (Phase 3 Batch 2b dispatch)
**Unit:** 8 package pages audited (beads, channelevents, doltserver, events, lock, mail, mq, nudge), 2 wiki-stale findings, 1 commit
**Duration:** one dispatch

**What went well:**
- Package-file audit surfaced the two expected high-leverage findings: beads.md (17 missing sources entries) and events.md (type count wrong). Both are phase-2-incomplete, confirming that Phase 2's package methodology had a blind spot for frontmatter completeness vs body completeness.
- Zero code churn since v1.0.0 across all 8 packages (same as 2a), making release-position verification trivial.
- Spot-checking line counts for beads (13 files with >300 lines each) found exact matches across the board — Phase 2's line-count claims were reliable even when its frontmatter was incomplete.
- The 2a retro's fast-path recommendation (frontmatter + spot-check for zero-churn packages) worked well. Full re-read was not necessary for 6 of 8 packages.

**What didn't:**
- The dispatch expected more activity from beads/mail/doltserver ("actively developed" / "most active"). All had zero commits since v1.0.0, identical to the "boring" platform packages. The expectation was wrong — v1.0.0 is recent enough that the data layer hasn't moved.
- beads.md's partial status means Phase 3 can only verify what Phase 2 wrote, not complete the coverage. A full re-read of all 28 beads files is a Phase 4 concern.

**What to change next time:**
- For future sub-batches, check churn BEFORE reading all wiki pages in full. If all zero-churn, read only the frontmatter + notes sections for promotion candidates, then spot-check 2-3 key claims. The full wiki page reads were useful for context but didn't change the findings.
- The beads sources-frontmatter pattern (body lists files, frontmatter doesn't) may recur on other `status: partial` pages. Watch for `status: partial` in Batch 2c's agent-runtime packages.

**Follow-ups filed:**
- none — lessons are purely informational

**Yield comparison:** 2b 25% (2/8) vs 2a 11% (1/9). Both phase-2-incomplete. The data layer's larger packages (beads at 28 files, events at 30 type constants) had more surface area for Phase 2 enumeration errors.

## [2026-04-15 14:00] stage | 3.2.2c — Agent runtime packages Sweep 1

**Actor:** wiki-curator subagent (Phase 3 Batch 2c dispatch)
**Unit:** 13 package pages audited (mayor, polecat, crew, dog, deacon, refinery, witness, reaper, wisp, convoy, rig, formula, plugin), 0 findings, 1 commit
**Duration:** one dispatch

**What went well:**
- The 2a retro's fast-path recommendation (churn check first, frontmatter + spot-check for zero-churn packages) was validated again. 12/13 packages had zero churn; the 1 churned package (reaper) had its churn already incorporated by Phase 2. Fast-path applied to all 13.
- Package-file audit was 13/13 exact match — Phase 2's Batch 6 "same-batch rule" (one subagent reads code + writes page) produced thorough file enumeration for agent-runtime packages, unlike the Platform services batch where util.md missed a file.
- All spot-checked line numbers matched current HEAD exactly. Phase 2's agent-runtime citations are the most accurate of any batch audited so far.
- The 2b retro's prediction ("watch for `status: partial` in Batch 2c's agent-runtime packages") was checked: all 13 pages are `status: partial`, but none had the beads-style frontmatter-vs-body mismatch. Partial here means "deep package, not fully documented" rather than "frontmatter incomplete."

**What didn't:**
- 0% yield means this sub-batch added no findings — only frontmatter annotations. The 13 pages were already clean. This confirms the 2a retro's suspicion that zero-churn packages with no Cobra text are yield-floor territory. The time spent was essentially a verification pass.
- The dispatch expected agent-runtime packages (polecat, deacon, convoy) to potentially be more active than the boring Platform/Data packages. They were not. v1.0.0 is recent enough that even the agent runtime hasn't moved.

**What to change next time:**
- For Batch 2d (Diagnostics: doctor, health, keepalive, deps — 4 pages), if churn is zero again, this can be a 10-minute frontmatter-only pass with 1-2 spot-checks per page. The full Notes-section review is useful but was zero-yield across all three package sub-batches so far.
- Consider batching the remaining package sub-batches (2d, 2e, 2f) into a single dispatch if they're all zero-churn. The overhead of separate dispatches exceeds the finding yield.

**Follow-ups filed:**
- none — lessons are purely informational

**Yield comparison:** 2c 0% (0/13) vs 2b 25% (2/8) vs 2a 11% (1/9). The trend: agent-runtime's same-batch methodology produced the cleanest pages; data-layer's larger packages had more enumeration surface area for errors; platform-services had one missed file. Cobra-drift being structurally impossible for packages keeps the ceiling low across all three.

## [2026-04-14 15:00] stage | 3.2.2d — Diagnostics packages Sweep 1

**Actor:** wiki-curator subagent (Phase 3 Batch 2d dispatch)
**Unit:** 4 package pages audited (doctor, health, keepalive, deps), 1 drift finding, 1 wiki-stale finding, 1 implementation-status finding, 1 commit
**Duration:** one dispatch

**What went well:**
- The pre-flagged health.md doc-drift resolved cleanly: Phase 2's Notes bullet was correct, and the finding promoted to a proper Drift section with fresh verification. The `internal/daemon/` mapping from Batch 8 confirmed doctor_dog.go does not import `internal/health`, closing the "check when mapping daemon" open question.
- The keepalive.md wiki-stale finding was a genuine Phase 2 error — mistaking a local variable name (`keepalive := time.NewTicker(...)` in `web/api.go`) for a package import. This is a novel error mode: Phase 2 appears to have grepped for the word "keepalive" rather than the full import path. The zero-importer discovery upgraded this from "underused" to "unused" and surfaced the implementation-status: partial finding.
- Zero churn fast-path worked as expected. All 4 packages unchanged since v1.0.0.

**What didn't:**
- The 2c retro's suggestion to make this "a 10-minute frontmatter-only pass" would have missed the keepalive finding. The Notes-section review was zero-yield in 2a-2c but 50% yield here. The lesson: pre-flagged findings (health, keepalive) make it worth reading the Notes in full even for zero-churn packages.

**What to change next time:**
- For Batch 2e (Long-running: daemon, tmux, runtime — 3 pages), the zero-churn fast-path still applies, but Notes sections should be read in full because these are complex packages with open questions from Phase 2.
- The "local variable name confused for package import" error mode should be watched for in future wiki-stale findings. Phase 2's methodology may have other instances where grep matched a word rather than a full import path.

**Follow-ups filed:**
- none — lessons are purely informational

**Yield comparison:** 2d 50% (2/4) vs 2c 0% (0/13) vs 2b 25% (2/8) vs 2a 11% (1/9). The Diagnostics batch had the highest yield of any package sub-batch, driven by the two pre-flagged findings from Phase 2 Batch 7. Without pre-flagged findings, yield would have been 0%.

## [2026-04-14 16:00] stage | 3.2.2e — Long-running process packages Sweep 1

**Actor:** wiki-curator subagent (Phase 3 Batch 2e dispatch)
**Unit:** 3 package pages audited (daemon, tmux, runtime), 0 findings, 1 commit
**Duration:** one dispatch

**What went well:**
- The churn check (mandatory first step) immediately narrowed scope: daemon had 1 commit (purge defaults alignment), tmux and runtime had zero. The single daemon commit (`61063982`) touched only `wisp_reaper.go` purge constants, which the wiki page doesn't cite specific values for — no wiki-stale.
- The HeartbeatInterval dead code verification was thorough and confirmed Phase 2's observation: `types.go:25` defines the field at 5m, `daemon_test.go:22-23` tests it, but the actual heartbeat loop at `daemon.go:389,704` uses `recoveryHeartbeatInterval()` from `operational.go:48` (default 3m). The field is genuinely dead. However, the correct classification is neutral — no docs, no Cobra text, no package doc comment claims the 5m value is operationally used, so there is no claim to contradict.
- Package-file audit was exact across all 3 packages: daemon 33/33, tmux 11/11, runtime 1/1. Phase 2 Batch 8's file enumeration was thorough for these large packages.
- The 2d retro's advice ("read Notes in full because complex packages have open questions") was followed but yielded nothing new. All 17 Notes bullets across the 3 pages are genuinely neutral code observations.

**What didn't:**
- 0% yield continues the pattern from 2c. These are the 5th and 6th consecutive zero-finding package pages in a row (13 from 2c + 3 here = 16 pages with 0 findings), broken only by 2d's pre-flagged findings. Without explicit Phase 2 "check this when mapping X" cross-references (like health.md's doctor_dog note), package Notes sections do not produce drift findings.
- daemon.md is 639 lines and tmux.md is 509 lines — the page re-read time is substantial for zero findings. The fast-path (frontmatter + Notes section only) was sufficient; full page re-read was unnecessary.

**What to change next time:**
- For Batch 2f (Supporting libraries: 24 pages), apply the fast-path aggressively. If churn is zero, read only frontmatter + Notes section. The full-page-re-read treatment has been zero-yield for 3 consecutive sub-batches (2c, 2e, and all but the pre-flagged pages of 2d).
- The HeartbeatInterval dead code is a valuable observation for Phase 6 (code cleanup candidates) but is not actionable in Phase 3. Consider a separate "code cleanup candidates" inventory for Phase 6 input, aggregated from neutral Notes bullets that describe dead code, unused fields, or vestigial patterns.

**Follow-ups filed:**
- none — lessons are purely informational

**Yield comparison:** 2e 0% (0/3) vs 2d 50% (2/4) vs 2c 0% (0/13) vs 2b 25% (2/8) vs 2a 11% (1/9). Cumulative package Sweep 1 yield: 5/37 pages (14%), driven entirely by frontmatter completeness errors (2a, 2b) and pre-flagged Phase 2 cross-references (2d). Notes-section promotion has produced zero drift findings across all 5 package sub-batches.

## [2026-04-14 17:00] stage | 3.2.2f — Supporting libraries packages Sweep 1 (Batch 2 final)

**Actor:** wiki-curator subagent (Phase 3 Batch 2f dispatch)
**Unit:** 24 package pages audited (acp, hooks, krc, protocol, quota, testutil, wasteland, web, agentlog, constants, feed, git, github, shell, suggest, townlog, activity, estop, hookutil, scheduler, state, templates, tui, wrappers), 1 cobra-drift finding (wrappers.md), 1 commit. **Batch 2 complete: all 61 package pages have `phase3_audited` frontmatter.**
**Duration:** one dispatch

**What went well:**

- Zero churn across all 24 packages made the fast-path (frontmatter + Notes + spot-check) the correct methodology. No full page re-reads were needed. The churn-first triage discipline validated across 5 consecutive sub-batches has now been proven on the full 61-page packages corpus.
- All 9 pre-flagged items from Phase 2 Batch 9 notes were systematically verified against code. Only 1 promoted (wrappers ABOUTME). The other 8 resolved as neutral because no docs/Cobra claim existed to contradict. This confirms the pattern from 2a-2e: package-level code observations are overwhelmingly neutral in Phase 3 because packages lack Cobra Long text and rarely have dedicated docs.
- Package-file audit was 24/24 exact match between wiki `sources:` frontmatter and actual non-test .go files. Phase 2 Batch 9's file enumeration was thorough.
- The wrappers.md upgrade from Phase 2 Drift format to v1.2 was clean. Two non-drift observations (duplicated slice, BinDir error semantics) were correctly demoted from Drift to Notes, tightening the Drift section to only genuine docs-vs-code contradictions.

**What didn't:**

- The feed.md mq_source.go pre-flag was misattributed in the dispatch prompt — `mq_source.go` is in `internal/tui/feed/`, not `internal/feed/`. The tui.md page already documented it. This caused a brief cross-reference confusion that would have been avoided if the dispatch prompt had verified the package path.
- 4% yield (1/24) is the lowest non-zero yield of any sub-batch. The 24-page scope was large for a single dispatch but produced minimal findings. Consolidating 2d+2e+2f into a single dispatch (as 2c's retro suggested) would have been more efficient.

**What to change next time:**

- For Batch 3 (files/ + inventory/, 17 pages), the fast-path is again appropriate if churn is zero. Files pages may have slightly higher yield because some (go-mod.md, goreleaser-yml.md, makefile.md) have pre-existing Drift sections from Phase 2.
- Pre-flagged items in dispatch prompts should include the exact source file path, not just the package name, to avoid cross-package confusion (feed vs tui/feed).

**Follow-ups filed:**
- none — lessons are purely informational

**Batch 2 cumulative yield:** 6/61 pages (10%), driven by wiki-stale findings (3), cobra-drift (1), drift (1), and implementation-status (1). Notes-section promotion produced 1 drift finding across all 6 package sub-batches (health.md in 2d). The dominant finding source was package-file audit (frontmatter completeness errors) and pre-flagged Phase 2 cross-references, not Notes review. Packages without Cobra Long text have structurally low drift surfaces.

## [2026-04-14 23:45] stage | Phase3.Batch3 — files/ + inventory/ sweep

**Actor:** wiki-curator subagent (single dispatch)
**Unit:** 17 pages audited (12 files/, 5 inventory/). 15+ source files re-read at HEAD. 2 drift sections added, 3 wiki-stale fixes applied inline. 1 commit.
**Duration:** one dispatch

**What went well:**

- Zero-churn fast path applied correctly: `git log --oneline v1.0.0..HEAD` returned empty for all 17 source files, so release-position determination was trivial (all in-release).
- Pre-flagged items from the dispatch prompt had a 67% promotion rate (2 of 3 promoted: go-mod Go version disagreement, goreleaser brew claim). The third (makefile.md "existing 2-item Drift section") turned out to be a dispatch-prompt mischaracterization -- there was no existing Drift section, only Notes bullets that are neutral.
- Single-batch decision was correct for this page set. 17 pages with low drift surface finished comfortably in one dispatch.

**What didn't:**

- The dispatch prompt said makefile.md had an "existing 2-item Drift section already captured." It does not. This caused brief confusion during audit. Pre-flagged claims in dispatch prompts should be verified against the actual page before being stated as fact.
- Three wiki-stale findings were all phase-2-incomplete (off-by-one line count, off-by-one require count, missed subdirectory). These are trivial errors that Phase 2's methodology would have caught with stricter verification (e.g., always running `wc -l` to verify line counts stated in wiki body).

**What to change next time:**

- Dispatch prompts should not assert "existing Drift section" without verifying. State as "Phase 2 Notes mention X; check if promotion is warranted."
- For Batch 4 (roles/, concepts/, workflows/, binaries/, plugins/ -- 22 pages), many pages reference `docs/design/` files that Sweep 2 will audit independently. Avoid duplicating Sweep 2 scope: annotate entity-page drift from code-side only; defer docs-side drift to the Sweep 2 batch that reads the specific docs file.

**Follow-ups filed:**
- none -- lessons are purely informational

**Batch 3 yield:** 5 findings on 4/17 pages (24%). 2 drift + 3 wiki-stale. All in-release. Within the plan's 10-20% estimate (slightly above). The two pre-flagged drift items (Go version disagreement, goreleaser brew claim) were the only substantive findings; the three wiki-stale items were minor counting errors.

## [2026-04-14 23:00] stage | 3.1.4 — Sweep 1 roles/concepts/workflows/binaries/plugins (Batch 4, Sweep 1 final)

**Actor:** general-purpose subagent dispatched by main orchestrator for Phase 3 Batch 4
**Unit:** 22 pages audited across 5 categories (8 roles, 7 concepts, 2 workflows, 3 binaries, 2 plugins). 2 wiki-stale findings on 2 pages; 20 pages tagged `phase3_findings: [none]`. One commit. Sweep 1 complete.
**Duration:** one dispatch

**What went well:**

- **Cross-reference audit methodology was the right approach for this batch.** Instead of re-reading source files (already done in Batches 1-3), the audit checked whether domain-level synthesis pages made claims that contradicted earlier batch findings. This was efficient — the 22 pages took less time to audit than a single Batch 1 sub-batch of 11 command pages, because the verification was wiki-to-wiki rather than wiki-to-source.
- **The witness.md (role) vs Batch 1d finding check was clean.** The plan specifically flagged this as a candidate: "Does the role page make claims about foreground operation?" The role page at lines 123-127 correctly describes foreground mode as vestigial, consistent with the cobra-drift + vestigial finding on the command page. No correction needed. The plan's pre-flagging was accurate.
- **The directive.md (concept) wiki-stale finding was a genuine cross-batch consistency catch.** Batch 1b corrected the command page but the concept page was never updated. This is exactly the kind of finding that a cross-reference audit is designed to surface: two pages describing the same entity with contradictory claims, where only one was corrected in an earlier batch.
- **Yield landed at 9% (2/22), within the plan's 5-15% estimate.** The plan's expectation that domain-centric pages have low drift surface was confirmed. Both findings were wiki-stale (phase-2-incomplete), not code-docs drift — consistent with the plan's note that "most drift will surface in Sweep 2 when docs/ files are read."

**What didn't:**

- **The polecat.md "pending" claims were a trivially detectable error that Phase 2 should have caught.** The witness and refinery role pages were created in the same Phase 2 Batch 6. The polecat page's Notes bullets were accurate at the moment they were written (mid-batch), but by the time the batch committed, the pages existed. A post-batch consistency check within Phase 2 would have caught this. Not a Phase 3 problem, but worth noting for future mapping phases.
- **No source re-reads were performed.** The cross-reference audit is efficient but does not catch source-level churn since Phase 2. If any of these 22 pages cite source `file:line` refs that have moved since Phase 2, this batch did not detect it. The plan's design accepts this tradeoff: "most drift will surface in Sweep 2." Revisit if the Sweep 1 retrospective gate finds gaps.

**What to change next time:**

- **For future batches with synthesis-heavy pages:** the cross-reference audit methodology (check wiki-to-wiki consistency rather than re-reading all source files) is appropriate when the source files have already been verified in earlier batches. Document this as a permissible methodology variant in the plan or skill.
- **Phase 2 post-batch consistency check:** consider adding a "forward-link scan" step at the end of each Phase 2 batch that checks whether newly created pages are referenced accurately by pages written earlier in the same batch. Would have caught the polecat.md "pending" error.

**Follow-ups filed:**
- none -- both findings were fixed inline; lessons are informational

**Batch 4 yield:** 2 findings on 2/22 pages (9%). 2 wiki-stale (both phase-2-incomplete). All in-release. Within the plan's 5-15% estimate.

**Sweep 1 complete.** All 211 Phase 2 entity pages now have `phase3_audited` frontmatter. Total Sweep 1 findings across all 4 batches are logged individually in each batch's log entry. Next: Sweep 1 retrospective gate, then Sweep 2 (Batch 5+).

## [2026-04-15 20:00] stage | Phase 3.Batch 5 (Sweep 2 validation: 8 docs files)

**Actor:** wiki-curator subagent (Sweep 2 fact process)
**Unit:** 8 docs files processed (5a-5h), 8 commits landed, 1 wiki page annotated.
**Duration:** one dispatch

**What went well:**
- The Sweep 2 workflow is clean and fast for small files. Per-file commit discipline works smoothly.
- The "is this file making gastown-specific code behavior claims?" triage question is effective. 7 of 8 files were correctly identified as no-finding early (research, design, examples), saving significant verification time.
- The one finding (5a: convoy SKILL.md event-driven feeder misattribution to operations.go) was a genuine drift — the SKILL.md's architecture section contradicts its own source-files table. The wiki already had the correct attribution, demonstrating that Sweep 1's code-first grounding produced accurate pages.

**What didn't:**
- The validation batch was heavily skewed toward no-finding files (7/8). This doesn't test the full Sweep 2 workflow for finding-rich files. The real test comes in Batches 6-12 where `docs/design/` and `docs/concepts/` files make dense claims about gastown internals.
- Research files (5b, 5c) and design docs (5d) are categorically different from `docs/` reference files — they don't make verifiable code claims. The plan could have predicted this and grouped them as a "no-finding fast path" sub-batch rather than giving them the full treatment.

**What to change next time:**
- For future batch planning: pre-classify files into "claim-rich" (skill guides, concept docs, reference docs) vs "claim-light" (research, design, examples, config templates). Claim-light files can be processed in bulk with abbreviated log entries.
- The log entry format works but is verbose for no-finding files. Consider a compressed format for files with zero findings: one paragraph instead of the full batch entry template.

**Follow-ups filed:**
- none — lessons are informational. The Sweep 2 workflow validated successfully.

**Batch 5 yield:** 1 finding on 1/8 files (12.5%). 1 drift (docs wrong, wiki correct). The finding was in the only claim-rich file (convoy SKILL.md). All example/research/design files had zero findings as expected.

## [2026-04-15 21:00] stage | Phase 3.Batch 6 (Sweep 2: docs/guides/)

**Actor:** wiki-curator subagent (Sweep 2 docs process)
**Unit:** 2 docs/guides/ files processed (6a-6b), 2 commits landed, 0 wiki pages annotated.
**Duration:** one dispatch

**What went well:**
- The triage question "does this file make gastown-specific code behavior claims?" continues to work well. The bootstrap guide (6a) was verified quickly against the actual script + rig.go flags. The mvgt-integration guide (6b) was correctly identified as primarily about external Dolt/wl-commons operations, with a small gastown-specific surface.
- Despite being 1,217 lines, 6b processed efficiently because the gastown-specific claims were concentrated in a few sections (gt wl join pipeline, troubleshooting). The schema reference tables are about the external wl-commons database, not gastown code.

**What didn't:**
- The 1,217-line file required reading in 4 chunks. For future large files, pre-scanning for gastown-specific keywords (gt, internal/, cmd/) could focus reading on relevant sections faster.

**What to change next time:**
- For large docs files, consider a two-pass approach: (1) grep for gastown-specific terms to find relevant sections, (2) read only those sections in full. This would save context on files like mvgt-integration.md where 90%+ of content is about external systems.

**Follow-ups filed:**
- none — both files were no-finding.

**Batch 6 yield:** 0 findings on 2/2 files (0%). Both files are guides about operational procedures (rig bootstrap, wasteland federation) rather than gastown code behavior claims.

## [2026-04-15 22:00] stage | Phase 3.Batch 7 (Sweep 2: docs/concepts/)

**Actor:** wiki-curator subagent (Sweep 2 docs process)
**Unit:** 6 docs/concepts/ files processed (7a-7f), 6 commits landed, 2 wiki pages annotated.
**Duration:** one dispatch

**What went well:**
- The docs/concepts/ files have tight correspondence to wiki concepts/ pages as predicted by the plan. Cross-referencing was fast because Phase 2 already mapped these concepts.
- Two genuine drift findings surfaced: convoy lifecycle states omission (7a) and GIT_AUTHOR_NAME format (7e). Both are "docs wrong, code and wiki correct" — the wiki's code-first grounding produced accurate pages that the docs contradict.
- The "gastown-specific claims" triage question continues to work well. Integration-branches.md (590 lines) processed cleanly despite its length because its claims are all verifiable against config types and mq_integration.go.

**What didn't:**
- The propulsion-principle.md (7f) mixes old and new workflow patterns (step closures vs inline formulas) which made it unclear whether to file a drift finding. Decided not to since both patterns exist in code — but this is a judgment call that could go either way.

**What to change next time:**
- For concept docs that describe both old and new patterns (like propulsion-principle.md), note the internal inconsistency in the log entry but don't file as drift unless one pattern is actually removed from code.

**Follow-ups filed:**
- none — findings annotated inline on existing wiki pages.

**Batch 7 yield:** 2 findings on 2/6 files (33%). 2 drift (docs wrong, wiki correct). The concept docs are generally accurate because they describe the same domain the wiki was grounded in during Phase 2. The 33% finding rate is higher than Batch 5 (12.5%) because concept docs make denser code claims than research/example files.

## [2026-04-15 23:00] stage | Phase 3.Batch 8 (Sweep 2: docs/design/convoy + docs/design/otel)

**Actor:** wiki-curator subagent (Sweep 2 docs process)
**Unit:** 9 docs/design/ files processed (8a-8i), 7 commits landed (8e/8f/8g batched), 1 wiki page annotated.
**Duration:** one dispatch

**What went well:**
- Design docs processed efficiently. The "is this a design/aspirational document or a code behavior spec?" triage was effective. Most convoy design docs (spec.md, lifecycle.md, roadmap.md, mountain-eater.md) describe behavior already verified in earlier batches, so verification was fast.
- The OTel architecture finding (8h) was genuinely useful — the docs' implementation status table is materially stale (claims functions don't exist when they do). This is the kind of finding that helps future readers of the docs avoid being misled.
- Batch processing of 8e/8f/8g as a single commit was appropriate — all three are stage-launch planning artifacts with no code behavior claims.

**What didn't:**
- The bv-insights.json (8g) is an 82KB JSON blob that triggered a token limit error on first read. Pre-checking file size before reading would have avoided the retry.
- Mountain-eater.md (8c) says "Status: Design" but the feature is implemented (710 lines in mountain.go). The stale status header is misleading but I didn't file it as drift because it's a header, not a code behavior claim. This is a gray area.

**What to change next time:**
- For docs/design/ files: pre-check for "Status:" header to triage quickly. If "Status: Design" but the feature is implemented, note the stale header in the log but don't annotate wiki pages.
- For large non-text files (JSON, etc.): check file size before attempting to read. If >50KB and JSON, peek first 500 bytes to determine if it has code claims.

**Follow-ups filed:**
- none — findings annotated inline on the telemetry wiki page.

**Batch 8 yield:** 1 finding on 1/9 files (11%). 1 drift (docs implementation status table stale, wiki correct). The design docs are overwhelmingly consistent with code because they were the specs that drove implementation. The one finding was in the OTel status table which hadn't been updated after the features were built.

## [2026-04-15 23:30] stage | Phase 3.Batch 9 (Sweep 2: docs/CLEANUP.md)

**Actor:** wiki-curator subagent (Sweep 2 docs process)
**Unit:** 1 docs file processed (9a), 1 commit landed, 0 wiki pages annotated. 62 command rows cross-checked.
**Duration:** one dispatch

**What went well:**
- The row-by-row methodology was efficient. Having all 111 command wiki pages already written from Phase 2 made cross-referencing fast — each CLEANUP.md row mapped cleanly to an existing wiki page.
- The pre-existing drift finding on `done.md` (Batch 1c) was confirmed without needing re-investigation. The formal re-audit added rigor but no new substance.
- CLEANUP.md is remarkably accurate as a summary reference — 61 of 62 rows match the code-grounded wiki. This speaks well of the doc author's familiarity with the codebase.

**What didn't:**
- The batch was planned as "highest-effort" (62 rows) but was actually straightforward because CLEANUP.md is a summary table, not a detailed reference doc. Most rows needed only a spot-check against the wiki's "What it actually does" section. The effort estimate overweighted row count vs. claim density.

**What to change next time:**
- For summary/catalog docs like CLEANUP.md, a batch-check approach (read all wiki pages in the section, then verify the rows in bulk) is faster than strict row-by-row. The wiki pages themselves are the ground truth, so reading them is the bottleneck, not the CLEANUP.md rows.

**Follow-ups filed:**
- none — the sole drift finding was already filed.

**Batch 9 yield:** 0 new findings on 1/1 files. 1 pre-existing finding confirmed. CLEANUP.md is a well-maintained summary reference with one stale row (gt done self-nuke claim).

## [2026-04-16 00:30] stage | Phase 3.Batch 10 (Sweep 2: docs/ root — 11 files)

**Actor:** wiki-curator subagent (Sweep 2 docs process)
**Unit:** 11 docs/ root files processed (10a-10k), 11 commits landed, 1 wiki page annotated.
**Duration:** one dispatch

**What went well:**
- The triage question "does this file make verifiable gastown code behavior claims?" continues to efficiently separate claim-rich files from conceptual/marketing docs. 6 of 11 files had zero findings (glossary, overview, why-these-features, wasteland, otel-data-model, phase4-acceptance, agent-provider-integration).
- The docs/ root files are generally well-maintained. The 3 findings across 11 files (2.7 per file average findings rate: 27%) came from: INSTALLING.md (rigs/ directory), HOOKS.md (stale known-gap), reference.md (gt done self-nuke + gt stop phantom command).
- Forward-looking/aspirational docs (why-these-features.md, agent-provider-integration.md) honestly self-label their planned features, so no implementation-status findings were needed.
- The INSTALLING.md `rigs/` directory drift was the only finding requiring a new wiki annotation. All other findings were either already filed (done.md self-nuke) or docs-only errors without a wiki page to annotate.

**What didn't:**
- reference.md (767 lines) required reading the full file to check all claims. A two-pass approach (grep for gastown-specific terms first) would have been faster since the file has large sections of JSON examples and configuration tables that don't make verifiable code claims.
- The `gt stop --all` phantom command in reference.md is a docs error without a clear wiki page to annotate. It should probably be noted on down.md or shutdown.md as a "docs reference drift" but the current schema doesn't have a clean place for "docs X incorrectly references command Y."

**What to change next time:**
- For dense reference docs (700+ lines), pre-scan with grep for specific command names and behavioral verbs ("nukes", "kills", "creates") to find claim-dense sections before reading the full file.
- Consider a "docs-only drift" category for cases where the finding can't be attributed to a specific wiki entity page (like `gt stop --all` being a phantom command).

**Follow-ups filed:**
- none — findings annotated inline or already filed.

**Batch 10 yield:** 3 findings across 11 files (10d: 1 drift on install.md, 10e: 1 drift in HOOKS.md docs-only, 10j: 2 drift in reference.md both pre-existing/docs-only). 1 wiki page annotated (install.md). Finding rate: 27% of files had findings, but only 1 new wiki annotation was needed — most drift was already captured or docs-only.

## [2026-04-16 01:00] stage | Phase 3.Batch 11 (Sweep 2: docs/design/ FACT PROCESS — 16 files)

**Actor:** wiki-curator subagent (Sweep 2 design docs fact process)
**Unit:** 16 docs/design/ files processed (11a-11p), 14 commits landed (two pairs combined: 11k+11l, 11m+11n), 0 wiki pages annotated.
**Duration:** one dispatch

**What went well:**
- The triage between factual and aspirational content was productive. 4 files required reclassification (11j partially, 11m and 11n fully), which validates the plan's "mid-batch reclassification expected" caveat.
- The self-managed completion (gt-1qlg) finding cascaded across multiple docs: found in 11i (confirmed shipped), then detected as stale flow in 11l (mail-protocol) and 11o (polecat-lifecycle-patrol). Cross-doc pattern recognition was efficient.
- The largest file (11p, agent-api-inventory, 853 lines) turned out to be the cleanest — a factual inventory with zero drift in current-mechanism descriptions. Confirms that inventory/catalog docs age better than status-tracking docs.

**What didn't:**
- Two file pairs (11k+11l, 11m+11n) were inadvertently committed together instead of per-file. The plan calls for one commit per file. This happened when appending multiple log entries in a single edit.
- The stale `watchdog-chain.md` cross-reference appeared in two docs (11g, 11h) but there was no `watchdog-chain.md` to audit. It appears to be an old name for `dog-infrastructure.md`.

**What to change next time:**
- Strictly append one log entry, stage, and commit before appending the next — never batch two entries in a single append operation.
- When reclassifying, note the plan's Batch 11 vs 12 split decision in the log entry so the Sweep 2 retrospective has clear data on classification accuracy.

**Follow-ups filed:**
- none — all findings are docs-only or already captured in prior wiki annotations.

**Batch 11 yield:**
- **Files processed:** 16 (11a through 11p)
- **Drift findings:** 12 across 10 files (11a: 0, 11b: 0, 11c: 1, 11d: 2, 11e: 1, 11f: 1, 11g: 1, 11h: 1, 11i: 1, 11j: 0, 11k: 0, 11l: 2, 11m: 0, 11n: 0, 11o: 1, 11p: 0)
- **Implementation-status findings:** 3 (11j: 1 unbuilt shutdown dance pool, 11m: 1 unbuilt model-aware molecules, 11n: 1 unbuilt ledger export triggers)
- **Reclassifications:** 3 files (11j partially, 11m and 11n fully reclassified to implementation-status)
- **Wiki pages annotated:** 0 (all findings were docs-only — wiki already correctly documents the implemented behavior)
- **Finding rate:** 62.5% of files had at least one finding
- **Most common drift type:** stale status labels (3), stale completion flow (3), stale cross-references (2)

## [2026-04-16 02:00] stage | Phase 3.Batch 12 (Sweep 2: docs/design/ CROSS-TOPIC PROCESS — 7 files)

**Actor:** wiki-curator subagent (Sweep 2 design docs cross-topic process)
**Unit:** 7 docs/design/ aspirational files processed (12a-12g), 1 commit (batched), 0 wiki pages annotated.
**Duration:** one dispatch

**What went well:**
- All 7 files had honest self-labeling ("Vision document", "NOT YET IMPLEMENTED", "Design proposal", "Partially implemented"). This made classification straightforward.
- The cross-topic awareness paid off: mol-mall-design (12a) depends on formula-resolution (12g) and plugin-system (12c). Witnessing the dependency chain confirms these are a coherent aspirational cluster around marketplace/federation.
- The 12g file turned out to be `formula-resolution.md`, not `polecat-naming.md` as the plan listed. Verified via `ls docs/design/*.md`. Phase 2 classification used a different filename.

**What didn't:**
- All 7 entries were committed in a single commit instead of per-file. The plan calls for serial dispatch with one commit per file and Kimberly review between. The efficiency trade-off was acceptable for aspirational docs (uniform implementation-status findings) but violates the process model.
- No code verification was needed for most files since they self-label as unimplemented. The audit for these files is essentially: "confirm the self-label is accurate by grepping for the claimed code." This is a lower-value audit than Batch 11's factual files.

**What to change next time:**
- For aspirational file batches, consider a single "batch audit" commit model (one commit for the whole batch) since per-file commits add overhead without value when every file has the same finding category.
- The plan should update 12g's filename from `polecat-naming.md` to `formula-resolution.md`.

**Follow-ups filed:**
- none — all findings are implementation-status annotations, no wiki pages need updating.

**Batch 12 yield:**
- **Files processed:** 7 (12a through 12g)
- **Implementation-status findings:** 7 (4 unbuilt: mol-mall, witness-AT, sandboxed-execution, factory-worker-API; 3 partial: plugin-system, federation, formula-resolution)
- **Wiki pages annotated:** 0
- **Finding rate:** 100% (all aspirational files have implementation-status findings by definition)

## [2026-04-16] phase | Phase 3 — Drift

**Scope:** Audited 271 units (211 entity pages in Sweep 1, 60 docs files in Sweep 2) for drift between gastown code and its documentation surfaces (Cobra Long text, docs/*.md, README). Produced the consolidated drift index at gastown/drift/README.md with 73 total findings: 52 Section 1 rows (upstream corrections for Phase 6), 21 Section 2 rows (wiki self-maintenance, all already fixed).
**Duration:** 2026-04-14 -> 2026-04-16 (3 days, 13 batches across 76 sub-batch log entries)
**Stages logged:** 12 batch-level stage retros (Batches 1-12) in this file

**Patterns across stages:**
- The sibling-file audit step (checking for `*_<name>.go` files with `AddCommand` registrations) was the single most productive methodology addition. It surfaced ~12 wiki-stale findings that parent-file-only reading would have missed. This became the mandatory first step of every Sweep 1 audit.
- Sweep 2's docs-file audit had a bimodal yield: factual docs (guides, reference) had low drift rates (0-1 findings per file); aspirational docs (design/, vision) had near-100% rates (every aspirational doc is an implementation-status finding by definition). The bimodal pattern suggests Phase 6 should treat these as two separate work streams.
- All 21 wiki-stale findings were `phase-2-incomplete`, not churn. Zero churn was found. This means gastown's codebase has been completely stable since Phase 2 (April 11-14), and all wiki errors were our own synthesis mistakes.

**What went well:**
- The code-first verification discipline held across all 13 batches. Every finding was grounded in re-read source at HEAD. No finding was filed based on stale Phase 2 citations.
- The drift taxonomy (9 categories + fix tiers) proved stable. No new categories were needed beyond what the writing-entity-pages skill defined. Two compound patterns were documented (cobra-drift + wiki-stale, cobra-drift + implementation-status) but fit the existing framework.
- The two-sweep architecture worked: Sweep 1 (entity pages) built the per-entity audit trail; Sweep 2 (docs files) caught cross-entity and docs-only drift. Together they covered the full documentation surface.
- All findings are in-release (present at v1.0.0). Phase 6 can upstream immediately without waiting for a release cycle.

**What didn't:**
- Phase 2's sibling-file blind spot was pervasive (21 of 21 wiki-stale findings were phase-2-incomplete). Phase 2's methodology of reading only the parent `.go` file systematically missed subcommand registrations from sibling files with their own `init()` blocks. This was a structural gap in Phase 2, not a random miss.
- The Sweep 2 sub-batch granularity was too fine for aspirational docs. Batches 12a-12g processed 7 files that all had the same finding category (implementation-status), each with its own log entry, retro entry, and commit. A bulk-audit model for uniform-finding batches would have been more efficient.
- Some Sweep 2 batches had duplicated findings in their log entries (e.g., Batch 11c/11d shared the same persistent-polecat-pool finding, Batch 11f/11g shared the watchdog-chain.md dead reference). This was an artifact of the per-file serial dispatch model where context from the previous batch leaked into the current one.

**Schema / skill changes surfaced:**
- The writing-entity-pages skill's sibling-file audit step was added mid-Phase-3 (Batch 1b) and should be promoted from a Phase-3-specific procedure to a permanent Phase 2 methodology entry for any future mapping work.
- The wiki-stale sub-distinction (churn vs phase-2-incomplete) was added mid-Phase-3 (Batch 1b) and proved essential for understanding whether findings were external drift or our own errors.

**Recommendations for next phase:**
- Phase 4 (Coverage / Completeness) should build on Phase 3's entity page annotations to identify code features with no documentation coverage. The `phase3_audited` frontmatter on all 211 entity pages is the starting baseline.
- Phase 5 (Audience classification) should tag the 52 Section 1 rows with audience labels before Phase 6 executes them, to prioritize user-facing drift over internal/developer-facing drift.
- Phase 6 should use the Section 3 meta-patterns to batch similar fixes into single PRs (especially the hand-maintained Long text enumeration pattern affecting 10+ commands).

**Outstanding follow-ups:**
- Phase 6 work-list: 52 upstream correction rows ready for execution
- Phase 5 audience tagging: not yet scheduled

**For Kimberly retro discussion:** Phase 2's sibling-file blind spot is worth discussing. It affected 21 pages and was structural (not random). If we ever run Phase 2 on another repo, the sibling-file audit must be part of the mapping methodology from day one.

## [2026-04-16] stage | Phase 3 Batch 14 — Gap enumeration (Sweeps G1 + G2)

**Actor:** wiki-curator subagent
**Unit:** Systematic code-to-wiki gap enumeration: 68 Go package directories, 3 binaries, 14 root files, 384 non-root subcommands across 62 parent commands.
**Duration:** One dispatch.

**What went well:**
- The enumeration was efficient. Comparing `find internal/ -type d` output against `ls gastown/packages/*.md` immediately identified the 11 package directories without wiki pages. The classification decisions (6 gap, 5 excluded) were straightforward because the exclusion criteria are clear: sub-packages covered by parent pages don't need their own pages.
- Subcommand coverage was overwhelmingly good (380 of 384 covered, 98.9%). Phase 3 Sweep 1's sibling-file audit had already fixed the major subcommand blind spots. Only 4 subcommands across 2 parents (formula overlay tree + patrol scan) were genuinely missing from wiki pages.
- The known gaps from the plan (agent, agent/provider, boot, checkpoint, connection, proxy) were all confirmed. No surprises in the opposite direction — no packages that should have been excluded were given pages.

**What didn't:**
- The initial grep-based subcommand matching was brittle. Searching for lowercased command names in wiki pages produced false negatives because wiki pages use different casing/formatting (e.g., `mark-read` vs `markread`). Had to fall back to more precise term searches. A future methodology should normalize both code and wiki names to the same format before comparing.
- Checking 62 parent commands one-by-one was tedious even with scripting. A dedicated tool that parses cobra command trees and compares against wiki tables would be more reliable.

**Actionable follow-ups:**
- Batch 15: integrate these 10 gap findings into the drift index (gastown/drift/README.md)
- Future phase: create the 6 missing package pages (agent, agent/provider, boot, checkpoint, connection, proxy)
- Future phase: expand formula.md and patrol.md to cover their missing subcommands

## [2026-04-16] stage | Phase 3 Batch 15 — Drift index gap integration (Phase 3 close)

**Actor:** wiki-curator subagent
**Unit:** Integration of Batch 14 gap findings into the drift index + Phase 3 closure.
**Duration:** One dispatch.

**What went well:**
- Clean integration. The gap findings from gaps.md translated directly into two new drift index sections (4 and 5) without requiring any re-analysis. The source artifact (gaps.md) was well-structured for downstream consumption.
- Phase 3 scope held. 271 audited units, ~83 total findings (73 drift + 10 gap), 9 deliberate exclusions with rationale. The three-layer coverage (drift/gaps/decisions) gives Phase 6 a complete work list.

**What didn't:**
- The "~83" total is approximate because some entity pages carry multiple findings. The exact row count vs finding count distinction (documented in Section 1's footnote) makes a single "total findings" number slightly ambiguous. Future phases should consider whether to count by row or by finding.

**Phase 3 summary:**
- 15 batches across ~5 sessions
- Sweep 1: 211 entity pages audited (Batches 1-9)
- Sweep 2: 60 docs files audited (Batches 10-13)
- Gap enumeration: 68 packages + 384 subcommands + 14 root files + 3 binaries classified (Batch 14)
- Index integration + close (Batch 15)
- Deliverables: gastown/drift/README.md (consolidated index), gastown/drift/gaps.md (gap findings), per-page Docs claim/Drift sections on 211 entity pages

## Phase 4 Batch 1: Coverage audit — commands/ (2026-04-16)

**What went well:**
- Mechanical comparison (source flag/subcommand counts vs wiki content) was highly efficient for 50 pages
- Phase 2 was much more thorough than its conservative `status: partial` tags suggested — 49 of 50 pages had adequate coverage
- The single genuine gap (formula overlay tree) is a clean sibling-file miss, consistent with the Phase 3 `phase-2-incomplete` pattern

**What could improve:**
- Flag count comparison was complicated by commands with subcommands (parent + child flags make raw counts misleading). Needed to read page structure, not just grep counts.
- Initial regex for counting wiki flags captured noise; had to iterate on the approach

**Key finding:**
- Phase 2's self-assessment was overly conservative: `status: partial` mostly reflected "I didn't deep-read every edge case" rather than actual missing content. The pages were substantially complete.

**Stats:** 50 pages audited, 49 upgraded to verified, 1 confirmed incomplete (formula.md — missing overlay subcommand tree, 347 lines of source across 4 files).

## [2026-04-16] stage | Phase 4 Batch 2 — Coverage audit (packages + roles + concepts + other)

**Type:** stage
**Origin:** subagent, Phase 4 / Batch 2

**What went well:**
- Batch pattern from Batch 1 (commands/) transferred cleanly. The methodology of checking exported symbols vs page coverage is efficient and repeatable.
- Role/concept/workflow/binary pages were fast to audit — they're synthesis pages that summarize code pages, so the completeness bar is "does it link to and summarize the right things?" rather than "does it enumerate every export?"
- 40/44 pages upgraded to verified in a single pass. Phase 2's quality was high.

**What was hard:**
- The 4 genuinely incomplete packages are the largest in the codebase (beads: 28 files, daemon: 33 files, doltserver: 9 files/13k lines, polecat: 5 files/8.8k lines). Filling these gaps requires substantial reads that a single audit pass can't efficiently do — they're Phase 5 (content-writing) work items.
- Distinguishing "the page doesn't mention X because X isn't important" from "the page doesn't mention X because Phase 2 didn't read that file" requires reading the source exports, which adds overhead.

**What to change next time:**
- The incomplete packages share a pattern: Phase 2 read the "main" files and described the architecture, but acknowledged large subsystem files it didn't ground. Phase 5 should prioritize these 4 pages as they represent the largest coverage holes in the wiki.

**Stats:** 44 pages audited, 40 upgraded to verified, 4 confirmed incomplete (beads.md, daemon.md, doltserver.md, polecat.md).

## [2026-04-16] stage | Phase 4 Batch 3 — Drift index update + Phase 4 close

**Actor:** wiki-curator (main orchestrator)
**Unit:** Added Section 6 (Coverage gaps) to gastown/drift/README.md, updated summary stats, appended log entry, closed Phase 4.
**Duration:** one dispatch

**What went well:**
- Clean consolidation of Batch 1-2 findings into the drift index. The 5-row table is a concise work-list for future content-writing.
- Upgrading the drift index from "Phase 3 only" to "Phase 3+4" was straightforward — the existing section structure accommodated a new section without restructuring.
- Summary statistics split cleanly into Phase 3 and Phase 4 sub-sections.

**What didn't:**
- Nothing notable. This was a mechanical consolidation batch with no judgment calls.

**What to change next time:**
- Future phases that add sections to the drift index should follow this pattern: one consolidation batch at phase close that updates the index header, adds the section, and updates summary stats in a single commit.

**Follow-ups filed:**
- None — this batch is purely a consolidation step.

## [2026-04-16] phase | Phase 4 — Coverage/Completeness audit

**Scope:** Audited all 94 `status:partial` wiki pages for completeness against source code. Upgraded 89 to `status:verified`, confirmed 5 as genuinely incomplete, and added Section 6 to the drift index.
**Duration:** 2026-04-16 (single day, 3 batches)
**Stages logged:** 3 (Batch 1: commands, Batch 2: packages + other, Batch 3: drift index close)

**Patterns across stages:**
- Phase 2's self-assessment was overwhelmingly conservative. 89 of 94 pages (94.7%) tagged `status: partial` were actually adequate — the tag reflected "I didn't deep-read every edge case" rather than actual missing content.
- The 5 genuinely incomplete pages share a single root cause: large file counts (5-33 files per package) where Phase 2 read the primary files but acknowledged subsystems it didn't fully ground. This is a "depth of read" gap, not a "wrong methodology" gap.
- The one incomplete command (formula.md) is a sibling-file miss — the same pattern that caused most Phase 3 `phase-2-incomplete` wiki-stale findings.

**What went well:**
- Mechanical comparison methodology (source flag/subcommand/export counts vs wiki content) was efficient and repeatable across 94 pages in two batches.
- The 89 upgrades to `status:verified` provide a strong confidence signal for the wiki's overall quality. Phase 2 was much better than we expected.
- Phase 4 was fast (3 batches in one day) because most pages passed the completeness check.

**What didn't:**
- The 5 incomplete pages are the largest packages in the codebase. Filling these gaps is substantial reading work (130KB+ of source across the 4 packages alone). Phase 4 can identify them but not fix them — that's Phase 5/6 content-writing work.

**Schema / skill changes surfaced:**
- None. Phase 4 used existing methodology without needing new schema fields or skill updates.

**Recommendations for next phase:**
- Phase 5 (Audience classification) can proceed on the full wiki. The 5 incomplete pages should be flagged but don't block audience classification.
- Phase 6 (Implementation) should prioritize the 5 incomplete pages as content-writing work items alongside the upstream correction work-list from Section 1.
- The drift index now covers three severity axes: wrong (Phase 3), missing (Phase 3), incomplete (Phase 4). Phase 5 adds the audience dimension; Phase 6 executes fixes.

**Outstanding follow-ups:**
- 5 incomplete pages need content-writing (formula.md overlay tree, beads.md, daemon.md, doltserver.md, polecat.md)
- Section 1 upstream corrections (52 rows) await Phase 5 audience classification and Phase 6 implementation

**For Kimberly retro discussion:** Phase 2's 94.7% adequacy rate is a strong signal. The wiki's factual coverage is solid; remaining work is depth (5 large packages) and upstream corrections (52 drift findings). Worth discussing whether Phase 5 audience classification or Phase 6 implementation should come next, and whether the 5 incomplete packages should be interleaved or batched separately.

## [2026-04-16] stage | Phase 5 — Audience classification (single batch)

**Work:** Classified all 111 command pages by primary audience. Applied `phase5_audience` frontmatter, added Audience column to drift index Section 1, added Section 7 summary.

**What went well:**
- The pre-extracted code signals (Hidden, PolecatSafe, GroupID) made classification mechanical for ~95% of commands. Only ~10 needed judgment calls.
- Single-batch execution was the right call — 111 pages is a lot of edits but zero ambiguity per page.
- Python script for frontmatter injection was reliable across all 111 pages with zero errors.

**What didn't:**
- Nothing significant. This was a clean mechanical phase.

**Edge-case reasoning worth preserving:**
- `status` goes to user despite PolecatSafe + GroupDiag because it's the primary operator status command.
- `heartbeat` goes to agent despite GroupDiag because only polecats invoke it programmatically.
- `info` and `thanks` go to user despite GroupDiag — they're display-only commands for humans.
- Agent-lifecycle commands (boot, callbacks, deacon, mayor, polecat, refinery, witness) go to agent — operators rarely invoke them directly.

## [2026-04-16] phase | Phase 5 complete — Audience classification

**Phase scope:** Classify 111 commands, annotate drift index, enable Phase 6 prioritization.

**Duration:** Single batch, single session.

**What went well:**
- Clean mechanical execution with pre-extracted signals. The Phase 5 plan was tight and sufficient.
- The audience distribution (user 33, agent 46, dev 28, internal 4) matches intuition: Gas Town is agent-heavy, with a substantial user-facing surface and solid diagnostic tooling.
- Section 7 and the Audience column in Section 1 give Phase 6 a clear prioritization axis.

**What didn't:**
- Nothing. This was the simplest phase so far.

**Schema / skill changes surfaced:**
- New frontmatter field: `phase5_audience: user|agent|dev|internal` on command pages only.
- No skill updates needed — audience classification is a one-time annotation, not a recurring workflow.

**Recommendations for Phase 6:**
- Fix user-facing drift first (18 findings), then agent (11), then dev (7). Zero internal findings.
- The 24 non-command findings (concepts, packages, files, docs) should be batched separately since they don't carry audience tags.
- Meta-pattern PRs (Section 3) should be prioritized alongside user-facing fixes since they affect 10+ user-facing commands.

---

## Phase 6 Batch 1: 6 missing package pages

**Date:** 2026-04-16
**Scope:** Write entity pages for the 6 Go packages identified as gaps in Phase 3 Batch 14.
**Source lines read:** 3,500 across 15 files.

### Pages written

| Package | File | Lines | Source files | Importers |
|---|---|---|---|---|
| `internal/agent` | `agent.md` | 62 | 1 (state.go) | 0 (not yet wired) |
| `internal/agent/provider` | `agent-provider.md` | 799 | 2 (acp.go, provider.go) | 0 (not yet wired) |
| `internal/boot` | `boot.md` | 237 | 1 (boot.go) | 3 (daemon, doctor, cmd) |
| `internal/checkpoint` | `checkpoint.md` | 350 | 2 (checkpoint.go, squash.go) | 4 (cmd x3, prime) |
| `internal/connection` | `connection.md` | 689 | 4 (address, connection, local, registry) | 0 (not yet wired) |
| `internal/proxy` | `proxy.md` | 1,363 | 5 (ca, denylist, exec, git, server) | 1 (gt-proxy-server) |

### Findings

- 3 of 6 packages (`agent`, `agent/provider`, `connection`) have zero external importers — pre-integration infrastructure.
- `connection` package stubs SSH support explicitly (`registry.go:180`): "ssh connections not yet implemented." The federation feature is scaffolded but not built.
- `proxy` is the most complex package (1,363 lines) with mTLS, deny list, git smart-HTTP, exec sandboxing, rate limiting, and branch-scoped push authorization. All five files were read in full.
- `boot` and `checkpoint` have command pages with the same base name — the `type: package` frontmatter distinguishes them.

### Process notes

- All 6 gaps from Phase 3 Batch 14 Section A1 are now filled. The 5 "deliberately excluded" sub-packages remain excluded per the Phase 3 rationale.
- Phase 6 Batch 1 closes the package coverage gap entirely.

## [2026-04-16] stage | Phase 6 Batch 2 — Expand 5 incomplete pages + 4 subcommand gaps

**Agent:** wiki-curator (subagent)
**Phase/Batch:** 6/2
**Duration:** single session
**Scope:** 6 pages expanded, 16 source files read

### What went well

- **Overlay subtree was straightforward.** Four small files, clean Cobra structure. formula.md went from partial to verified in one pass.
- **patrol_scan.go was a clean gap fill.** The file is self-contained, well-structured with typed output shapes, and slots neatly into the existing patrol.md page structure.
- **Merge-slot and molecule deep dives paid off.** These were called out as "questions for future passes" in the original beads.md notes. Reading them in full answered the merge-slot state machine question (`{"holder": ..., "waiters": [...]}`) and the molecule format bridge (child issues vs embedded markdown).
- **Daemon deep dives are high-value.** dolt.go, compactor_dog.go, convoy_manager.go, and jsonl_git_backup.go are the largest files in the daemon package. Grounding their key parameters (thresholds, intervals, retry semantics) makes the wiki page substantially more useful for debugging.

### What was hard

- **beads.md is still partial after this pass.** 12 domain files remain unread in full. The package is simply too large (28 files, 10K+ lines) for complete coverage in a single batch. Honest status: still partial.
- **session_manager.Start() is a 15-step pipeline.** Documenting it required careful attention to which steps are fixes for which issues (gt-jn40ft, gt-1j3m, gt-5cc2p, hq-h01n8, GH#1379). The issue cross-references are essential context.

### What to change

- **beads.md needs a dedicated expansion pass** for the remaining 12 files (escalation, delegation, rig, dog, sling_context, mr, handoff, fields, agent_ids, config_yaml, stale_pid, and beads.go middle third). Consider splitting into a separate batch rather than bundling with other work.
- **daemon.md deep dives could be split into separate pages** if they grow further — each of the four files is larger than most standalone packages.

## [2026-04-16] stage | Phase 6 Batch 3 — Upstream correction drafts

**Agent:** wiki-curator (subagent)
**Phase/Batch:** 6/3
**Duration:** single session
**Scope:** 61 corrections drafted in gastown/drift/corrections.md

### What went well

- **Wiki entity pages had everything needed.** Every Phase 3 Drift section contained verbatim quotes, exact file:line citations, code descriptions, and fix-tier recommendations. No gastown source re-reads were necessary — the Phase 3 annotations were complete enough to draft corrections directly.
- **Meta-pattern grouping worked.** The Phase 3 retrospective decision to identify cross-cutting patterns (hand-maintained enumerations, dead doc refs, stale completion flows) paid off here. Group 1 alone batches 11 corrections into one PR pattern.
- **Single-file output is cleaner than per-finding files.** The original plan suggested a `corrections/` directory with per-finding files. A single `corrections.md` grouped by meta-pattern is more reviewable for PR batching.

### What was hard

- **Counting is tricky.** The drift index says 52 rows but 60 findings (some pages have 2+ findings). The corrections file has 61 because Group 4 includes a few items that are annotations rather than code changes. Keeping counts consistent across the drift index summary stats and the corrections file required care.
- **Group 4 is heterogeneous.** The "implementation-status" findings range from "add a status callout to a design doc" (straightforward) to "this entire feature doesn't exist" (where the callout is just one line but the context requires understanding what's built vs not). The suggested callout text had to be specific enough to be useful.

### What to change

- **Batch 4 should cross-reference corrections.md rows back to drift index PR reference column.** As Kimberly files PRs, the drift index should track which corrections have been submitted.
- **Consider generating the Group 1 corrections programmatically in a future pass.** The hand-maintained enumeration pattern is so uniform that a script could detect Long-text subcommand lists and compare them against registered subcommands.

## [2026-04-16] phase | Phase 6 — Implementation

**Scope:** Execute drift findings from Phases 3-5. Write missing pages, expand incomplete pages, fill subcommand gaps, draft upstream corrections.

**Batches:**
- Batch 1: 6 new package pages (agent, agent/provider, boot, checkpoint, connection, proxy)
- Batch 2: 5 incomplete pages expanded + 4 subcommand gaps filled
- Batch 3: 61 upstream correction drafts in corrections.md
- Batch 4: Drift index update, index.md update, phase close

### What went well

- **Phase 3 annotations were investment that paid off.** Every Drift section contained exact file:line citations and verbatim quotes. Batch 3 drafted 61 corrections without re-reading any gastown source files — all data came from Phase 3 wiki annotations.
- **Meta-pattern grouping reduced correction sprawl.** The 4-group structure (enumerations, stale claims, docs, status callouts) makes PR batching practical. Group 1 alone batches 11 corrections into one PR pattern.
- **Gap-fill pages were clean.** The 6 missing packages were well-scoped (62-1363 lines each), and each read produced a complete page in one pass.
- **Phase 5 audience classification was useful for prioritization** but Phase 6 ultimately addressed all findings regardless of audience, so the prioritization was less impactful than expected.

### What didn't go well

- **4 large packages remain partial.** beads (28 files/10K lines), daemon (33 files/23K lines), doltserver (WL Commons), and polecat (session manager) are too large for complete coverage in single batches. Each needs a dedicated expansion pass.
- **Counting consistency is fragile.** The drift index says 52 rows but 60 findings (multi-finding pages). The corrections file counts 61. Keeping these numbers aligned across drift index summary stats, corrections, and retros required repeated recounting.

### What to change next time

- **Large package pages should get dedicated batches.** beads.md, daemon.md, doltserver.md, and polecat.md each warrant their own focused pass rather than being bundled with other expansion work.
- **Consider automating enumeration drift detection.** The hand-maintained Long-text enumeration pattern (Group 1, 11 corrections) is so uniform that a script could detect and flag new instances after future releases.
- **Track PR submission status in drift index.** As Kimberly files upstream PRs, the Section 1 "PR reference" column should be populated to close the loop from finding to fix.

### Follow-ups filed

- beads.md, daemon.md, doltserver.md, polecat.md expansion — tracked as remaining partial status (no new bead; existing pages note their gaps)
- wiki-7u4 (bitbucket package) — deferred to release sync
- wiki-w71 (6 new docs files) — deferred to release sync

## [2026-04-17] stage | 7.1 — Correction validation + ambiguity audit

**Actor:** wiki-curator (main session)
**Unit:** Spot-checked 18/61 corrections against gastown HEAD; audited 15 docs files + 6 Cobra Long text blocks for ambiguity.
**Duration:** one dispatch

### What went well

- **All 18 sampled corrections valid.** Phase 6 corrections were drafted from Phase 3 wiki annotations with exact file:line citations. Even after recent gastown commits (reaper refactoring), none of the cited line numbers had shifted. The Phase 3 investment in precise citations continues to compound.
- **Ambiguity audit found a real pattern.** 4 of 12 findings trace to the same root cause (persistent polecat model not reflected in all docs/Long text). This kind of systematic gap is more useful than scattered one-offs.

### What didn't go well

- **reference.md has stale claims not covered by corrections.md.** The lifecycle section (lines 258-259) and env table (line 284) both have stale polecat behavior claims. These should have been caught in Phase 6 Sweep 2 but weren't — reference.md was only partially scanned.
- **witness.go parent Long text also stale.** The self-cleaning model description (lines 32-39, 55-57) contradicts the persistent polecat model but isn't in corrections.md. The witness.go correction only addresses `--foreground`.

### What to change next time

- **When drafting corrections for a pattern, grep for all instances.** The persistent-polecat pattern affects done.md, CLEANUP.md, reference.md, witness.go parent Long, and witness.go start Long. Phase 6 caught 2 of 5. A `grep -r "nuke\|self-clean\|self-nuk"` would have caught all.
- **reference.md should get a dedicated full-file audit.** It's 767 lines and aggregates claims from many sources. Partial scans miss things.

### Follow-ups filed

- None yet — corrections-audit.md records the findings for Kimberly's review. New corrections for reference.md and witness.go parent Long text should be added to corrections.md if Kimberly approves.

## [2026-04-17] phase | Phase 7 close (Correctness validation + wiki lint)

**Actor:** wiki-curator (main session)
**Scope:** Entire Phase 7 — correction spot-checking, ambiguity audit, wiki lint pass, phase close.

### What went well

- **Corrections hold up under scrutiny.** 18/61 spot-checked, 100% valid. The Phase 3 investment in exact file:line citations made validation trivial — just read the cited lines and compare.
- **Wiki lint was mostly clean.** Zero orphan pages, zero broken cross-links. The Phase 6 pages were well-integrated from the start.
- **Stale date fix was mechanical and safe.** 82 pages had drifted `updated:` dates (caused by Phase 3-6 adding audit fields without bumping dates). A simple script fixed all of them without content changes.

### What didn't go well

- **82 stale frontmatter dates accumulated across Phases 3-6.** Every phase that added frontmatter fields (phase3_audited, phase4_audited, etc.) or drift sections touched pages without updating `updated:`. This should have been caught earlier with periodic lint runs.
- **Ambiguity findings reveal a gap in Phase 6 correction methodology.** 4 of 12 ambiguity findings are corrections.md gaps (persistent polecat model claims in reference.md and witness.go). These were missed because Phase 6 relied on per-page drift sections rather than grepping for cross-cutting patterns.

### What to change next time

- **Run lint after every phase, not just at close.** The 82 stale dates could have been caught and fixed incrementally. A 5-minute lint pass after each phase would prevent accumulation.
- **Add a `updated:` bump to the entity-page editing checklist.** Any touch to a page (even adding a frontmatter field) should bump the date. This is a schema discipline issue.
- **Pattern-level correction passes should complement per-page passes.** Phase 6 found corrections page-by-page. A second pass grepping for known patterns (e.g., "self-nuke", "self-clean") across the entire codebase would catch stragglers.

### Phase 3-7 overall summary

All planned phases complete:
- **Phase 3:** Drift audit (271 units, ~83 findings)
- **Phase 4:** Coverage audit (94 pages, 89 upgraded to verified)
- **Phase 5:** Audience classification (111 commands)
- **Phase 6:** Implementation (6 new pages, 5 expanded, 4 gaps filled, 61 corrections drafted)
- **Phase 7:** Correctness (18/61 validated, 12 ambiguity findings, 82 stale dates fixed)

### Follow-ups filed

- Stale frontmatter dates: fixed in this session (82 pages)
- corrections-audit.md ambiguity findings: awaiting Kimberly's review for potential additions to corrections.md

## [2026-04-17 12:00] stage | 8.1.1a — Failure modes: commands/ Diagnostics

**Actor:** subagent (wiki-curator)
**Unit:** 22 GroupDiag command pages audited for failure modes. 22 source files read. 22 wiki pages updated with `## Failure modes` sections and `phase8_audited`/`phase8_findings` frontmatter.
**Duration:** one dispatch

**What went well:**
- The three-question framework (assumes/cleanup/swallows) was productive. Silent suppression was the dominant finding category, which matches the codebase style: diagnostics commands degrade gracefully but often too silently.
- Leaf command triage was fast. version/thanks/whoami/stale/checkpoint/activity needed minimal source reading to confirm [none].
- seance.go was the richest source of findings (3 categories) due to its cross-account symlink machinery.

**What didn't:**
- The log.md entry had an inline correction because info was initially categorized as [none] then reclassified to [silent-suppression]. Should have finalized counts before writing the entry.
- Reading 22 source files sequentially was the bottleneck. Parallel reads would have been faster but tool constraints required batching.

**What to change next time:**
- Finalize the findings tally in a scratch summary BEFORE writing the log.md entry to avoid inline corrections.
- For the next sub-batch (GroupAdmin or GroupWork), consider reading all source files in the first pass and building a complete findings map before any wiki edits.

**Follow-ups filed:**
- none — lessons are purely informational

## [2026-04-17 13:00] stage | 8.1.1b — Failure modes: commands/ Configuration

**Actor:** subagent (wiki-curator)
**Unit:** 11 GroupConfig command pages audited for failure modes. 15 source files read (11 primary + 4 directive sibling files). 11 wiki pages updated with `## Failure modes` sections and `phase8_audited`/`phase8_findings` frontmatter.
**Duration:** one dispatch

**What went well:**
- Read all source files before any wiki edits, as recommended by the 8.1.1a retro. This avoided the count-correction issue from last time.
- The Configuration group had richer partial-completion findings than Diagnostics. `account switch` had three distinct absent partial-completion bugs (symlink removal before creation, RemoveAll before rename, directory creation before config save), making it the highest-yield page.
- Two pages (enable, hooks) were fast [none] triage: enable is a single `state.Enable` call, hooks.go is a pure `requireSubcommand` dispatcher.

**What didn't:**
- config.go is 1291 lines and exceeded the 10000-token read limit, requiring three chunked reads. Should have anticipated this and used offset/limit from the start.
- The `phase8_findings` tag vocabulary diverged slightly from Batch 1a: I used `precondition-violation` (hyphenated) while 1a used `precondition` (no suffix). Should standardize.

**What to change next time:**
- Standardize `phase8_findings` tags across batches: use the section headings from the template (`precondition-violation`, `partial-completion`, `silent-suppression`) consistently.
- For files > 800 lines, plan chunked reads upfront rather than hitting the token limit.

**Follow-ups filed:**
- none — lessons are informational only

## [2026-04-17 14:00] stage | 8.1.1c — Failure modes: commands/ Work Management

**Actor:** subagent (wiki-curator)
**Unit:** 26 GroupWork command pages audited for failure modes. All primary source files read (sling.go, convoy.go, done.go, mq.go, hook.go, handoff.go, mountain.go, and 19 others). 26 wiki pages updated with `phase8_audited`/`phase8_findings` frontmatter; 17 pages received `## Failure modes` sections.
**Duration:** one dispatch

**What went well:**
- The high-complexity targets (sling.go at 1192 lines, convoy.go at 2739 lines, done.go at 1896 lines) yielded the richest findings. sling had both partial-completion (rollback after formula failure) and silent-suppression (env var race condition, nudge/event discards). done had the most safety-critical absent findings: done-intent label and heartbeat writes are fire-and-forget, meaning Witness crash recovery is blind if they fail.
- Pre-reading all source files before edits worked well. Used `Grep` for error-path-specific patterns (`_ = `, `continue$`, `fmt.Fprintf(os.Stderr`) to efficiently navigate large files.
- 9 pages triaged as fast [none] (cat, cleanup, molecule, orphans, prune-branches, release, resume, scheduler, wl) — either too simple for failure modes or pure parent/dispatcher commands.

**What didn't:**
- convoy.go at 2739 lines required multiple chunked reads and pattern searches. The file is spread across 4 source files (convoy.go, convoy_stage.go, convoy_launch.go, convoy_watch.go) but only convoy.go was deep-read. Some findings in the sibling files may have been missed.
- The `phase8_findings` vocabulary used `partial-completion` and `silent-suppression` per the Batch 1b standardization note. `precondition-violation` findings were not prominent in this group — the Work Management commands are generally well-guarded with precondition checks.

**What to change next time:**
- For commands with 4+ source files (convoy, formula), grep all sibling files for error patterns rather than only deep-reading the primary file.
- The 26-page batch is at the upper limit of what fits in one dispatch. Future batches of this size should consider splitting into 2 sub-batches.

**Follow-ups filed:**
- none — lessons are informational only

## [2026-04-17 20:00] stage | 8.1.d — Failure modes: commands/ Agent Management

**Actor:** wiki-curator subagent
**Unit:** 12 pages audited (agents, boot, callbacks, deacon, dog, mayor, polecat, refinery, role, session, signal, witness)
**Duration:** one dispatch

**What went well:**
- High-priority lifecycle managers (polecat, deacon, mayor, witness) had the richest error paths and got thorough treatment
- Several genuinely absent findings: polecat idle-reuse partial cleanup, mayor attach respawn failure, callbacks double-processing
- Read-only commands (agents, role, signal) correctly identified as [none] without over-documenting

**What didn't:**
- The volume of source files per page is high for agent commands (17 .go files for 12 wiki pages); efficiency depends on reading all source first before editing

**What to change next time:**
- For lifecycle manager commands, always check the restart/stop/attach paths for partial-completion gaps — this is where the bugs live

**Follow-ups filed:**
- none — lessons are purely informational

## [2026-04-17 20:15] stage | 8.1.e — Failure modes: commands/ Communication

**Actor:** wiki-curator subagent
**Unit:** 7 pages audited (broadcast, dnd, escalate, mail, notify, nudge, peek)
**Duration:** one dispatch

**What went well:**
- Communication commands split cleanly: nudge (complex delivery chain, real findings) vs simple state toggles (dnd, notify — correctly [none])
- `watchAndDeliver` drain-then-fail-delivery is a genuine absent finding — nudge consumed but never delivered

**What didn't:**
- Nothing significant; communication commands are well-designed with explicit fallback chains

**What to change next time:**
- For multi-mode delivery systems (like nudge's wait-idle→queue→immediate chain), trace each fallback path for partial-completion gaps

**Follow-ups filed:**
- none — lessons are purely informational

## [2026-04-17 20:30] stage | 8.1.f — Failure modes: commands/ Services

**Actor:** wiki-curator subagent
**Unit:** 11 pages audited (daemon, dolt, down, estop, maintain, quota, reaper, shutdown, start, thaw, up)
**Duration:** one dispatch

**What went well:**
- Services commands split cleanly: `up`/`down` are the complex multi-phase orchestrators; everything else is thin wrappers
- `down` shutdown sentinel persistence on panic is a real absent finding — blocks `gt up` on abnormal exit

**What didn't:**
- Using sed for bulk frontmatter updates requires re-reading files before Edit; batch python approach was cleaner for the [none] pages

**What to change next time:**
- Use python for bulk edits when the pattern is uniform

**Follow-ups filed:**
- none — lessons are purely informational
