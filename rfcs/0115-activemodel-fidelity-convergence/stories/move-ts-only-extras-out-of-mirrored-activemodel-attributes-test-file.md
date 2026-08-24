---
title: "Move the TS-only extras out of activemodel's mirrored attributes test file"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 7011
claim: "2026-08-24T22:54:10Z"
assignee: "forced-mutation-tracker-takes-an-attributeset-where-rails-passes-the-model"
blocked-by: null
closed-reason: null
---

## Context

Same class as `move-ts-only-extras-out-of-mirrored-activemodel-errors-test-file`
(PR #6999) and `move-ts-only-extras-out-of-mirrored-activemodel-dirty-test-file`
(PR #7006), which cleared `errors_test.rb` (Extra 69 → 0) and `dirty_test.rb`
(Extra 36 → 0) respectively. The parent story named callbacks, serialization and
dirty as the remaining offenders; `attributes_test.rb` was missed and now sits
third by extra count.

Measured with `pnpm parity:test --package activemodel --sort-extra` on `main`
after PR #7006 merged:

| Ruby file               | OK  | Extra (TS only) |
| ----------------------- | --- | --------------- |
| `callbacks_test.rb`     | 11  | 45              |
| `serialization_test.rb` | 20  | 31              |
| `attributes_test.rb`    | 13  | 26              |

The first two already have stories
(`move-ts-only-extras-out-of-mirrored-activemodel-callbacks-test-file`,
`move-ts-only-extras-out-of-mirrored-activemodel-serialization-test-file`);
this is the third.

Rails counterpart: `vendor/rails/activemodel/test/cases/attributes_test.rb`.
trails file: `packages/activemodel/src/attributes.test.ts` (a
`attributes.trails.test.ts` sibling already exists and holds 2 tests).

Why it matters: with TS-only names interleaved among the Rails ones, a renamed
or split Rails test is invisible to review.

## Converged shape

The mirrored `.test.ts` contains only Rails test names; every TS-only `it(...)`
lives in `attributes.trails.test.ts` under a descriptive `describe`.

## Acceptance criteria

- Every `it(...)` `parity:test` reports as "extra (TS only)" for
  `attributes_test.rb` moves to `attributes.trails.test.ts`.
- No test is renamed, reworded or deleted. Only the enclosing `describe`
  changes. Shared helpers are duplicated or exported, never moved out from under
  the Rails tests.
- Afterwards the file reports Extra=0 with its OK count unchanged (13), and
  activemodel's matched total is unchanged. State before/after in the PR body.
- Watch for duplicated Rails test names: `errors.test.ts` held six Rails names
  twice, and moving the wrong copy silently changed which body `parity:test`
  matched, raising the assertion-value mark. Re-run
  `npx tsx scripts/test-compare/lint-assertion-mismatches.ts` before pushing,
  and tighten `assertion-mismatch-mark.json` DOWN for activemodel only — a bare
  reseed also lowers activerecord/activesupport from sibling work and buries the
  change.
- Both files green.

## Notes

An extra that fails once the Rails test beside it is correct is a real finding —
file it, do not adjust the extra to match.
