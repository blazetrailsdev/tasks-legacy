---
title: "Move the TS-only extras out of activemodel's mirrored callbacks test file"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: 7011
claim: "2026-08-24T22:54:10Z"
assignee: "forced-mutation-tracker-takes-an-attributeset-where-rails-passes-the-model"
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

| Ruby file           | OK  | Extra (TS only) |
| ------------------- | --- | --------------- |
| `callbacks_test.rb` | —   | 45              |

Rails counterpart: `vendor/rails/activemodel/test/cases/callbacks_test.rb`.
trails file: `packages/activemodel/src/callbacks.test.ts`.

Converged shape: the mirrored `.test.ts` contains only Rails test names; every
TS-only `it(...)` lives in a `callbacks.trails.test.ts` sibling under a descriptive
`describe`. Why it matters: with TS-only names interleaved among the Rails ones,
a renamed or split Rails test is invisible to review.

## Acceptance criteria

- Every `it(...)` `parity:test` reports as "extra (TS only)" for
  `callbacks_test.rb` moves to `callbacks.trails.test.ts`.
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
