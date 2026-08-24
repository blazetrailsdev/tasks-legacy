---
title: "Move the 45 TS-only extras out of activemodel's mirrored type test files"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: 6991
claim: "2026-08-24T15:28:24Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-type-test-files"
blocked-by: null
closed-reason: null
---

## Context

RFC 0115's `port-user-input-in-time-zone-and-close-the-activemodel-test-gap`
(PR #6988) closed activemodel's last `parity:test` miss. Root cause: the Rails
test `test_user_input_in_time_zone` had been **split into four renamed tests**
inside the mirrored file `packages/activemodel/src/type/time.test.ts`, so
`parity:test` could match none of them, and the one arm none of the four covered
(the offset round-trip) was hiding a real implementation divergence —
`Type::Time#user_input_in_time_zone` was not delegating to
`Helpers::TimeValue`'s `value.in_time_zone`
(`vendor/rails/activemodel/lib/active_model/type/helpers/time_value.rb:42-44`),
so a string carrying an offset read back as hour 3 instead of 19.

That PR moved those four extras to `type/time.trails.test.ts` and restored the
Rails name. It did **not** move the rest. `parity:test` still reports, for the
mirrored activemodel type files:

| Ruby file                | OK  | Extra (TS only) |
| ------------------------ | --- | --------------- |
| `type/time_test.rb`      | 3   | **19**          |
| `type/date_time_test.rb` | 5   | **26**          |

45 TS-only tests sit in files whose whole job is to mirror a Rails test file.
Every one is a place the same failure can recur: a reviewer reading
`type/time.test.ts` cannot tell at a glance which `it(...)` is a Rails name that
`parity:test` is matching on and which is trails-only, which is exactly the
confusion that produced the four renamed splits.

CLAUDE.md's rule is that TS-only extras live in the `.trails.test.ts` sibling.
Both siblings already exist (`type/time.trails.test.ts`,
`type/date-time.trails.test.ts`), so this is a move, not a new file.

## Acceptance criteria

- Every `it(...)` in `packages/activemodel/src/type/time.test.ts` and
  `packages/activemodel/src/type/date-time.test.ts` that `parity:test` reports as
  "extra (TS only)" moves to the matching `.trails.test.ts` sibling. Get the list
  from `pnpm parity:test --package activemodel` rather than by eye.
- **No test is renamed or reworded, and none is deleted.** A move is a move; if an
  extra looks redundant with a Rails test, leave it and note it — deleting
  coverage is a separate decision.
- Any shared helper an extra depends on (e.g. `timeUtc` in `type/time.test.ts`) is
  duplicated or exported, not moved out from under the Rails tests.
- After the move, `parity:test` shows both files at Extra=0 with their OK counts
  unchanged (`type/time_test.rb` 3, `type/date_time_test.rb` 5), and activemodel
  stays at 962/963. State before/after in the PR body.
- Both `.test.ts` and both `.trails.test.ts` files green.

## Notes

Watch for extras that silently encode a divergent implementation shape, as two of
the four moved in #6988 did — they asserted the pre-convergence return type of
`userInputInTimeZone`. An extra that fails once the Rails test beside it is
correct is a finding, not a merge conflict: file it rather than adjusting it to
match.
