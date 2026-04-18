---
title: internal/suggest
type: package
status: verified
topic: gastown
created: 2026-04-11
updated: 2026-04-17
sources:
  - /home/kimberly/repos/gastown/internal/suggest/suggest.go
tags: [package, suggest, fuzzy, levenshtein, did-you-mean, error-messages]
phase3_audited: 2026-04-14
phase3_findings: [none]
phase3_severities: []
phase3_findings_post_release: false
phase8_audited: 2026-04-17
phase8_findings: [none]
---

# internal/suggest

"Did you mean?" helper. Given a misspelled name (a command, a rig, an
agent) and a list of candidates, returns the top few best matches
ranked by a weighted similarity score. Also formats the error message
that presents those suggestions back to the user. Pure utility package
— no state, no I/O.

**Go package path:** `github.com/steveyegge/gastown/internal/suggest`
**File count:** 1 go file (`suggest.go`), 1 test file.
**Imports (notable):** stdlib only (`sort`, `strings`, `unicode`). No
internal dependencies.
**Imported by (notable):** [`gt`](../binaries/gt.md) top-level command
for cobra's unknown-subcommand handling, and various rig / agent /
bead entity lookups that want to produce "bead 'gt-042z' not found —
did you mean gt-042x?" errors.

## What it actually does

Source: `/home/kimberly/repos/gastown/internal/suggest/suggest.go`.

### Public API

- `FindSimilar(target string, candidates []string, maxResults int)
  []string` (`suggest.go:54-84`):
  - Lowercases `target` once.
  - Scores every candidate via `similarity` (also lowercased).
  - Drops zero-score candidates (`score > 0` filter).
  - Sorts descending, truncates to `maxResults`, returns just the
    string values.
  - Returns nil on empty candidates or `maxResults <= 0`.
- `FormatSuggestion(entity, name string, suggestions []string,
  createHint string) string` (`suggest.go:245-268`) — builds the
  standard error text:

  ```
  <entity> '<name>' not found

    Did you mean?
      • <suggestion-1>
      • <suggestion-2>
      ...

    <createHint>
  ```

  `createHint` and `suggestions` are both optional; empty values
  collapse the corresponding block.

### Scoring (`similarity`, `suggest.go:92-139`)

Additive bag-of-signals score. Weights are exported as named constants
(`suggest.go:13-43`) so they can be referenced from tests and tuned in
one place:

| Constant | Value | Applied when |
|---|---:|---|
| `ScoreExactMatch` | 1000 | `a == b` (short-circuits) |
| `ScorePrefixWeight` | 20 | per char of `commonPrefixLength(a, b)` |
| `ScoreContainsFullWeight` | 15 | if `b` contains `a`: `len(a)` chars |
| `ScoreSuffixWeight` | 10 | per char of `commonSuffixLength(a, b)` |
| `ScoreContainsPartialWeight` | 10 | if `a` contains `b`: `len(b)` chars |
| `ScoreDistanceWeight` | 5 | when Levenshtein distance `<= maxLen/2`: `(maxLen - dist)` chars |
| `ScoreCommonCharsWeight` | 2 | per character shared (multiset, letters + digits only) |
| `LengthDiffPenalty` | 2 | subtracted per char when `|len(a)-len(b)| > LengthDiffThreshold` |
| `LengthDiffThreshold` | 5 | threshold for the length-diff penalty |

The prefix weight dominates everything else, which is intentional —
users who mistype a command usually got the start right.

### Helpers

- `commonPrefixLength` / `commonSuffixLength` (`suggest.go:142-161`) —
  byte-wise comparison. Non-ASCII inputs work but multi-byte runes can
  produce partial-rune prefix lengths; for the names this package
  sees (cobra commands, bead IDs, rig names) that is a non-issue.
- `commonChars` (`suggest.go:164-180`) — multiset intersection count
  restricted to letters and digits (`unicode.IsLetter` /
  `unicode.IsDigit`), so punctuation differences don't tank the
  score. Iterates with `range` so runes are handled correctly.
- `levenshteinDistance` (`suggest.go:183-217`) — textbook 2D DP. Not
  optimized for memory since candidate lists are small (hundreds at
  most).
- `min`, `max`, `min3`, `abs` — package-private numeric helpers.

### Match struct

`Match struct { Value string; Score int }` (`suggest.go:47-50`) is
the intermediate form used by `FindSimilar` during scoring. Not
returned to callers — they get `[]string` with scores stripped.

## Related wiki pages

- [gt](../binaries/gt.md) — cobra's `SuggestionsMinimumDistance` /
  unknown-subcommand path is the primary consumer. Bead lookups and
  rig/agent name resolution in various commands also use it.
- [go-packages inventory](../inventory/go-packages.md).

## Notes / open questions

- The weight tuning is unexplained in code comments beyond "prefix
  matches are weighted highly as users often type the start of
  commands" (`suggest.go:17`). There is no tuning log or decision
  record; changes should be validated against the test file rather
  than the comments.
- `levenshteinDistance` uses full O(n*m) memory rather than the
  rolling two-row optimization. Acceptable at current candidate-list
  sizes but a cheap win if it ever becomes hot.
- `commonChars` uses a multiset match (decrements the counter as
  characters are consumed) — "banana" vs "banner" scores 5, not 6,
  because the second `a` in "banner" can't pair with anything.
- `FindSimilar` filters on `score > 0` rather than a tunable
  threshold. The scoring is additive enough that almost anything
  scores above zero via `ScoreCommonCharsWeight`, so in practice
  `maxResults` is the only real filter.
