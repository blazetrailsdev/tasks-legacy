---
title: "Move the TS-only extras out of activemodel's mirrored translation test file"
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Same class as `move-ts-only-extras-out-of-mirrored-activemodel-errors-test-file`
(PR #6999), `...-dirty-...` (PR #7006) and `...-attributes-...` /
`...-callbacks-...` (PR #7011, which cleared `callbacks_test.rb` 45 → 0 and
`attributes_test.rb` 26 → 0). Measured with
`pnpm parity:test --package activemodel --sort-extra` after #7011:

| Ruby file             | OK  | Extra (TS only) |
| --------------------- | --- | --------------- |
| `translation_test.rb` | 26  | 19              |

Rails counterpart: `vendor/rails/activemodel/test/cases/translation_test.rb`.
trails file: `packages/activemodel/src/translation.test.ts`.

With TS-only names interleaved among the Rails ones, a renamed or split Rails
test is invisible to review.

## Converged shape

The mirrored `.test.ts` holds only Rails test names; every TS-only `it(...)`
lives in `packages/activemodel/src/translation.trails.test.ts` under a descriptive
`describe`.

## Acceptance criteria

- Every `it(...)` `parity:test` reports as "extra (TS only)" for `translation_test.rb`
  moves to the `.trails.test.ts` sibling.
- No test renamed, reworded or deleted — only the enclosing `describe`. Shared
  helpers are duplicated or exported, never moved out from under the Rails tests.
- Afterwards the file reports Extra=0 with OK unchanged at 26, and activemodel's
  matched total (963) is unchanged. State before/after in the PR body.
- Watch for a Rails test name appearing twice (both #6999 and #7011 hit this):
  moving the wrong copy silently changes which body `parity:test` matches.
  Re-run `npx tsx scripts/test-compare/lint-assertion-mismatches.ts` before
  pushing.
- Both files green.

## Notes

An extra that fails once the Rails test beside it is correct is a real finding —
file it, do not adjust the extra to match.
