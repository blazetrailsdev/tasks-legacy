---
title: "move-ts-only-extras-out-of-the-remaining-activemodel-type-test-files-part-2"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7016
claim: "2026-08-24T23:42:13Z"
assignee: "move-ts-only-extras-out-of-the-remaining-activemodel-type-test-files-part-2"
blocked-by: null
closed-reason: null
---

## Context

Second half of `move-ts-only-extras-out-of-the-remaining-mirrored-activemodel-type-test-files`,
which took `type/decimal.test.ts`, `type/date.test.ts` and
`type/big-integer.test.ts` from 18/18/13 TS-only extras to 0 by moving them into
their `.trails.test.ts` siblings. That PR hit the LOC ceiling; these five files
were left.

`pnpm parity:test --package activemodel` after that PR:

| Ruby file                       | OK  | Extra (TS only) |
| ------------------------------- | --- | --------------- |
| `type/string_test.rb`           | 4   | 8               |
| `type/immutable_string_test.rb` | 2   | 8               |
| `type/value_test.rb`            | 2   | 8               |
| `type/registry_test.rb`         | 3   | 6               |
| `type_test.rb`                  | 1   | 4               |

TS files: `packages/activemodel/src/type/string.test.ts`,
`immutable-string.test.ts`, `value.test.ts`, `registry.test.ts`, and
`packages/activemodel/src/type.test.ts`. Rails counterparts are under
`vendor/rails/activemodel/test/cases/type/` (and `type_test.rb`).

`string.test.ts`, `value.test.ts`, `registry.test.ts` and `type.test.ts` have no
`.trails.test.ts` sibling yet — create one, following the shape established for
`time`/`date-time` (#6991) and `decimal`/`date`/`big-integer`: a descriptive
top-level `describe` per moved group.

## Acceptance criteria

- Every `it(...)` those rows report as "extra (TS only)" moves to the matching
  `.trails.test.ts` sibling. Get the list from `pnpm parity:test --package
activemodel`, not by eye.
- **No test is renamed, reworded, or deleted.** Only the enclosing `describe`
  changes.
- Shared helpers an extra depends on are duplicated or exported, never moved out
  from under the Rails tests.
- Afterwards every file above reports Extra=0 with its OK count unchanged, and
  activemodel's matched total (963) is unchanged. State before/after in the PR
  body.
- All touched `.test.ts` and `.trails.test.ts` files green.

## Notes

Watch for an extra that silently encodes a divergent implementation shape, as
two of the four moved in #6988 did. An extra that fails once the Rails test
beside it is correct is a finding to file, not something to adjust.
