---
title: "Move the TS-only extras out of the remaining mirrored activemodel type test files"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 550
priority: null
pr: 7001
claim: "2026-08-24T18:28:32Z"
assignee: "move-ts-only-extras-out-of-the-remaining-mirrored-activemodel-type-test-files"
blocked-by: null
closed-reason: null
---

## Context

Follow-up to `move-ts-only-extras-out-of-mirrored-activemodel-type-test-files`
(PR #6991), which took `type/time.test.ts` and `type/date-time.test.ts` from 19
and 26 TS-only extras to 0 by moving them into their `.trails.test.ts` siblings.

The rest of `packages/activemodel/src/type/` still carries the same smell — a
mirrored Rails test file whose `it(...)` list is mostly trails-only, so a reader
cannot tell which name `parity:test` is matching on. That confusion is what
produced the four renamed splits of `test_user_input_in_time_zone` that hid a
real `Type::Time#user_input_in_time_zone` divergence (closed by #6988).

`pnpm parity:test --package activemodel --sort-extra` on the merge of #6991:

| Ruby file                       | OK  | Extra (TS only) |
| ------------------------------- | --- | --------------- |
| `type/decimal_test.rb`          | 10  | 18              |
| `type/date_test.rb`             | 2   | 18              |
| `type/big_integer_test.rb`      | 4   | 13              |
| `type/string_test.rb`           | 4   | 8               |
| `type/immutable_string_test.rb` | 2   | 8               |
| `type/value_test.rb`            | 2   | 8               |
| `type/registry_test.rb`         | 3   | 6               |
| `type_test.rb`                  | 1   | 4               |

Rails counterparts live under `activemodel/test/cases/type/` (e.g.
`activemodel/test/cases/type/decimal_test.rb`, `.../date_test.rb`).

Converged shape: each mirrored `.test.ts` contains only Rails test names; every
TS-only `it(...)` lives in the `.trails.test.ts` sibling, under a descriptive
`describe` of its own. This is the shape #6991 established for `time` and
`date-time`.

## Acceptance criteria

- Every `it(...)` those files' rows report as "extra (TS only)" moves to the
  matching `.trails.test.ts` sibling. Get the list from
  `pnpm parity:test --package activemodel`, not by eye.
- **No test is renamed, reworded, or deleted.** A move is a move. Only the
  enclosing `describe` changes.
- Shared helpers an extra depends on are duplicated or exported, never moved out
  from under the Rails tests.
- Afterwards every file above reports Extra=0 with its OK count unchanged, and
  activemodel's matched total is unchanged. State before/after in the PR body.
- All touched `.test.ts` and `.trails.test.ts` files green.

## Notes

Watch for an extra that silently encodes a divergent implementation shape, as
two of the four moved in #6988 did (they asserted the pre-convergence return
type of `userInputInTimeZone`). An extra that fails once the Rails test beside it
is correct is a finding to file, not something to adjust.

Likely needs splitting across two PRs under the LOC ceiling; if so, split by
file with non-overlapping paths and file the second half as its own story.
