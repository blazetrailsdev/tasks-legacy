---
title: "Move the TS-only extras out of activemodel's mirrored errors test file"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 420
priority: null
pr: null
claim: "2026-08-24T18:04:22Z"
assignee: "descendants-tracker-weakset-include-predicate-name"
blocked-by: null
closed-reason: null
---

## Context

Same class as `move-ts-only-extras-out-of-mirrored-activemodel-type-test-files`
(PR #6991), which cleared `type/time.test.ts` and `type/date-time.test.ts`. The
four worst non-`type/` offenders in activemodel are far larger and each deserves
its own PR.

`pnpm parity:test --package activemodel --sort-extra` on the merge of #6991:

| Ruby file               | OK  | Extra (TS only) |
| ----------------------- | --- | --------------- |
| `errors_test.rb`        | 86  | 69              |
| `callbacks_test.rb`     | 11  | 45              |
| `dirty_test.rb`         | 25  | 36              |
| `serialization_test.rb` | 20  | 31              |

Rails counterparts: `activemodel/test/cases/errors_test.rb`,
`.../callbacks_test.rb`, `.../dirty_test.rb`, `.../serialization_test.rb`.

Converged shape: each mirrored `.test.ts` contains only Rails test names; every
TS-only `it(...)` lives in the `.trails.test.ts` sibling under a descriptive
`describe`. Why it matters: with 69 trails-only names interleaved among 86 Rails
ones in `errors.test.ts`, a renamed or split Rails test is invisible to review —
exactly the failure #6988 had to dig out.

## Acceptance criteria

- Scope this story to **one** of the four files (start with `errors_test.rb`);
  file the remaining three as sibling stories rather than fanning out PRs.
- Every `it(...)` `parity:test` reports as "extra (TS only)" for that file moves
  to its `.trails.test.ts` sibling.
- **No test is renamed, reworded, or deleted.** Only the enclosing `describe`
  changes. Shared helpers are duplicated or exported, never moved out from under
  the Rails tests.
- Afterwards the file reports Extra=0 with its OK count unchanged, and
  activemodel's matched total is unchanged. State before/after in the PR body.
- Both files green.

## Notes

An extra that fails once the Rails test beside it is correct is a real finding —
file it, do not adjust the extra to match.
