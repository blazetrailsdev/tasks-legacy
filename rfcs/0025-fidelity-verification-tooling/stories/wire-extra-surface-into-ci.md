---
title: "Gate the extra-surface novel count on an only-shrink high-water mark"
status: draft
updated: 2026-08-03
rfc: "0025-fidelity-verification-tooling"
cluster: null
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

> **Partly delivered by PR #6897 (2026-08-23).** The only-shrink mark, the gate
> script (`scripts/api-compare/lint-extra-surface-ratchet.ts`,
> `pnpm parity:api:extra:gate`) and the CI wiring in the
> `Rails API/Test Comparison` job all exist now — but scoped to **arel only**
> (`GATED_PACKAGES` in `scripts/api-compare/extra-surface-mark.ts`), pinned at
> the post-burndown `novel 0, total 63`. Widening to the other packages is
> `extra-surface-ratchet-widen-beyond-arel`. What remains here is only whatever
> this story wanted beyond that.

The 2026-08-03 api-signals audit found `pnpm parity:api:extra`
(`scripts/api-compare/extra-surface.ts`) is not referenced anywhere in
`.github/workflows/ci.yml` (grep count 0), despite hard-exiting non-zero on
stale `@noRailsEquivalent` tags and unclassified permanence claims — and it
fails today on one unclassified tag (`packages/activerecord/src/associations/errors.ts`
`NestedAttributesDisplacementError`). Invented public surface currently lands
ungated; only reviewer vigilance enforces the no-extra-surface rule.

Caveat from prior experience: raw extra totals move with BUILD state, not
commit (unbuilt packages drop members from the population), so absolute counts
must not be gated. Gate tag hygiene and only-shrink marks instead.

## Acceptance criteria

- A CI step in the Rails API/Test Comparison job runs the extra-surface gate
  after the API comparison step (manifests are fresh at that point).
- The gate fails on: a stale `@noRailsEquivalent` tag, a tag with no
  PERMANENT/CONVERGEABLE permanence claim, and a per-package NOVEL-count above
  a committed only-shrink high-water mark (same pattern as
  `scripts/api-compare/call-mismatches-wide-unreviewed/`).
- Raw extra totals (novel+moved absolute counts) are NOT gated.
- The one currently-unclassified tag is classified so the new gate lands green.

## Re-verified 2026-08-17 (draft sweep)

**Narrowed — three of four acceptance criteria have landed.**

What is done: `.github/workflows/ci.yml:1397` runs
`pnpm exec tsx scripts/api-compare/extra-surface.ts` in the Rails API/Test
Comparison job, after the API comparison step, with no `--exclude-glob` (an
exclusion would disarm the stale gate). It fails on a stale `@noRailsEquivalent`
tag, a reason stating neither PERMANENT nor CONVERGEABLE, and a refused
file-level tag. The formerly-unclassified `NestedAttributesDisplacementError`
tag is gone. The 2026-08-17 run reports 149 tags, 149 matched, 139 PERMANENT /
10 CONVERGEABLE / **0 unclassified**.

What remains, and is now the whole story: the **per-package NOVEL-count
only-shrink high-water mark** was never built. Novel counts are ungated, so
invented surface can still accumulate as long as it carries no tag. Current
baseline to seed from (2026-08-17, full build): activerecord 548, activesupport
373, actiondispatch 250, trailties 197, activemodel 132,
activerecord-test-support 103, arel 84, actioncontroller 77, actionview 68,
rack 49, abstractcontroller 13, i18n 6, globalid 4, did-you-mean 0.

The original caveat stands and is why only the mark may be gated: raw totals
move with BUILD state, not commit, so an unbuilt package silently drops members
from the population. Title updated to match the narrowed scope.
