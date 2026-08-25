---
title: "Move the TS-only extras out of activemodel's mirrored naming test file"
status: done
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: 7018
claim: "2026-08-25T00:06:07Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-naming-test-file"
blocked-by: null
closed-reason: null
---

## Context

Same class as `move-ts-only-extras-out-of-mirrored-activemodel-errors-test-file`
(PR #6999), `...-dirty-...` (PR #7006) and `...-attributes-...` /
`...-callbacks-...` (PR #7011, which cleared `callbacks_test.rb` 45 → 0 and
`attributes_test.rb` 26 → 0). Measured with
`pnpm parity:test --package activemodel --sort-extra` after #7011:

| Ruby file        | OK  | Extra (TS only) |
| ---------------- | --- | --------------- |
| `naming_test.rb` | 58  | 25              |

Rails counterpart: `vendor/rails/activemodel/test/cases/naming_test.rb`.
trails file: `packages/activemodel/src/naming.test.ts`.

With TS-only names interleaved among the Rails ones, a renamed or split Rails
test is invisible to review.

## Converged shape

The mirrored `.test.ts` holds only Rails test names; every TS-only `it(...)`
lives in `packages/activemodel/src/naming.trails.test.ts` under a descriptive
`describe`.

## Acceptance criteria

- Every `it(...)` `parity:test` reports as "extra (TS only)" for `naming_test.rb`
  moves to the `.trails.test.ts` sibling.
- No test renamed, reworded or deleted — only the enclosing `describe`. Shared
  helpers are duplicated or exported, never moved out from under the Rails tests.
- Afterwards the file reports Extra=0 with OK unchanged at 58, and activemodel's
  matched total (963) is unchanged. State before/after in the PR body.
- Watch for a Rails test name appearing twice (both #6999 and #7011 hit this):
  moving the wrong copy silently changes which body `parity:test` matches.
  Re-run `npx tsx scripts/test-compare/lint-assertion-mismatches.ts` before
  pushing.
- Both files green.

## Notes

An extra that fails once the Rails test beside it is correct is a real finding —
file it, do not adjust the extra to match.
