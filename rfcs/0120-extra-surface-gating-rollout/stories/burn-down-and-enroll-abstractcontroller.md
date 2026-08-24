---
title: "Burn down and enroll abstractcontroller"
status: ready
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: api-compare
packages: ["abstractcontroller"]
deps: ["extra-surface-mark-dimensions"]
deps-rfc: []
est-loc: 400
priority: 5
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Wave 1 of the enrollment rollout. From the 2026-08-24 run:

| package            | files | novel | moved | total | allowed | noCntrp |
| ------------------ | ----- | ----- | ----- | ----- | ------- | ------- |
| abstractcontroller | 5     | 4     | 41    | 45    | 0       | 29      |

The most extreme instance of the pattern this RFC's contract exists for: only
**4 novel** names, but **29 no-counterpart** extras across 5 files, and 41
moved. Enrolling on `novel: 0` alone would pin a package whose real state is
"most of its surface is in files the comparator cannot map", and the pin would
re-open the first time someone maps one.

`noCounterpart` files are scored with an EMPTY allowed set
(`scripts/api-compare/extra-surface.ts:1439-1440`, counted at `:1734-1735`), so
every public name in them lands as an extra regardless of whether Rails defines
it. Fixing the mapping is therefore likely to move names from `noCntrp` into
`moved` or out of the population entirely, not into `novel`.

abstractcontroller was one of RFC 0080's three migration packages (14 of the 27
`extra-surface-allow.json` entries were its), so its tag history is well
documented — but it reports **0 allowed** today, meaning those tags either
resolved or no longer match.

Use RFC 0117's triage model: delete / relocate / fold / tag LAST. Check
`PATH_SEGMENT_ALIASES` and `RUBY_FILE_TS_OVERRIDES` in
`scripts/parity/conventions.ts` before calling any name invented.

## Acceptance criteria

- `abstractcontroller` reaches `novel: 0` and `noCounterpartFiles: 0`.
- Enrolled with a four-dimension mark; `moved: 0` is NOT required, `total`
  records what remains.
- **Tag budget: at most 2 new `@noRailsEquivalent` tags** for the whole package,
  each naming the TypeScript language shortcoming in the PR body. A third means
  blocked, not tagged.
- The PR body accounts for all 29 no-counterpart extras: which files got mapped,
  which got deleted, which are genuinely unported (and belong in the unported
  register rather than the extra population).
- If this exceeds the LOC ceiling, ship the mapping half and file the remainder
  as a follow-up story — do NOT enroll on a partial contract.
- `pnpm parity:api:extra:gate` passes.
