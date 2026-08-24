---
title: "Burn down and enroll i18n"
status: ready
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: ["i18n"]
deps: ["extra-surface-mark-dimensions", "convergeable-tag-story-id"]
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

Wave 1 of the enrollment rollout, and the largest of the three. From the
2026-08-24 run:

| package | files | novel | moved | total | allowed | noCntrp |
| ------- | ----- | ----- | ----- | ----- | ------- | ------- |
| i18n    | 10    | 7     | 84    | 91    | 15      | 49      |

7 novel against **49 no-counterpart** and 84 moved. Like its wave-1 siblings the
work is mapping, not deletion — but i18n also carries **15 allowed**, the
highest ratio of tags to novel names of any near-term candidate (more than
double its novel count). That makes it the package where "reach novel 0 by
tagging" has already partly happened, and where the RFC's `allowed` ratchet
matters most.

`noCounterpart` files are scored with an empty allowed set
(`scripts/api-compare/extra-surface.ts:1439-1440`, counted at `:1734-1735`).
i18n is also a gem port rather than a Rails-framework package, so its Ruby-side
file layout maps onto `scripts/parity/conventions.ts`'s tables differently —
check `PATH_SEGMENT_ALIASES` / `RUBY_FILE_TS_OVERRIDES` before concluding a name
is invented.

`grep -c '@noRailsEquivalent' packages/i18n/src` returns 5 inline tags; the
report's 15 includes inherited/interface-declaration coverage
(`extra-surface.ts:744`, `:626-634`).

## Acceptance criteria

- `i18n` reaches `novel: 0` and `noCounterpartFiles: 0`.
- Enrolled with a four-dimension mark; `moved: 0` NOT required.
- **The 15 existing allowed entries are audited, not inherited.** Each is
  classified PERMANENT or CONVERGEABLE-with-story-id per
  `convergeable-tag-story-id`, and the PR body says how many survived. An
  enrollment that pins 15 unreviewed tags as the floor defeats the ratchet.
- **Tag budget: net zero new tags.** The audit may REMOVE tags; it may not add
  one without naming the language shortcoming.
- The PR body buckets every resolved name under RFC 0117's four triage headings.
- Likely exceeds the LOC ceiling as one PR. Split by shipping the conventions/
  mapping change first and the enrollment second, filing the second as its own
  story — do NOT enroll on a partial contract.
- `pnpm parity:api:extra:gate` passes.
