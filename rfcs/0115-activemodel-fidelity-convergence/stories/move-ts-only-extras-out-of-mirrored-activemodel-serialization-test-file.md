---
title: "Move the TS-only extras out of activemodel's mirrored serialization test file"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 230
priority: null
pr: 7012
claim: "2026-08-24T23:18:09Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-serialization-test-file"
blocked-by: null
closed-reason: null
---

## Context

Same class as `move-ts-only-extras-out-of-mirrored-activemodel-errors-test-file`
(PR #6999), which cleared `errors_test.rb` (Extra 69 → 0, OK 86 unchanged), and
`move-ts-only-extras-out-of-mirrored-activemodel-type-test-files` (PR #6991)
before it. The parent story scoped itself to `errors_test.rb` and asked for the
remaining three offenders to be filed as siblings; this is one of them.

Measured with `pnpm parity:test --package activemodel --sort-extra`:

| Ruby file               | OK  | Extra (TS only) |
| ----------------------- | --- | --------------- |
| `serialization_test.rb` | —   | 31              |

Rails counterpart: `vendor/rails/activemodel/test/cases/serialization_test.rb`.
trails file: `packages/activemodel/src/serialization.test.ts`.

Converged shape: the mirrored `.test.ts` contains only Rails test names; every
TS-only `it(...)` lives in a `serialization.trails.test.ts` sibling under a descriptive
`describe`. Why it matters: with TS-only names interleaved among the Rails ones,
a renamed or split Rails test is invisible to review.

## Acceptance criteria

- Every `it(...)` `parity:test` reports as "extra (TS only)" for
  `serialization_test.rb` moves to `serialization.trails.test.ts`.
- No test is renamed, reworded, or deleted. Only the enclosing `describe`
  changes. Shared helpers are duplicated or exported, never moved out from under
  the Rails tests.
- Afterwards the file reports Extra=0 with its OK count unchanged, and
  activemodel's matched total is unchanged. State before/after in the PR body.
- Watch for duplicated Rails test names: `errors.test.ts` held six Rails names
  twice, and moving the wrong copy silently changed which body `parity:test`
  matched, raising the assertion-value mark. Re-run
  `npx tsx scripts/test-compare/lint-assertion-mismatches.ts` before pushing.
- Both files green.

## Notes

An extra that fails once the Rails test beside it is correct is a real finding —
file it, do not adjust the extra to match.
