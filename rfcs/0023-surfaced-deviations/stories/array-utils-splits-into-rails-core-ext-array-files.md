---
title: "array-utils.ts aggregates four Rails core_ext/array files"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/array-utils.ts` is a trails-invented aggregation
file. It holds members Rails keeps in four separate files under
`activesupport/lib/active_support/core_ext/array/`:

| trails member in `array-utils.ts`   | Rails home                                              |
| ----------------------------------- | ------------------------------------------------------- |
| `wrap`                              | `core_ext/array/wrap.rb:39-46`                          |
| `inGroupsOf`, `inGroups`, `split`   | `core_ext/array/grouping.rb:21-49`, `:57-84`, `:92-107` |
| `extractBang`                       | `core_ext/array/extract.rb:19-26`                       |
| `toSentence`, `toFs`/`toFormattedS` | `core_ext/array/conversions.rb:8-84`, `:94-106`         |

The repo already has the correct destination directory —
`packages/activesupport/src/core-ext/array/` holds `access.ts`, which mirrors
`core_ext/array/access.rb` and uses the settled "Ruby reopens `Array`, so the
port is a class of the Ruby name whose members take the receiver first" idiom.
`array-utils.ts` predates that convention and never moved.

CLAUDE.md makes file layout part of fidelity ("If Rails extracts a private
helper, extract it… One Rails method is one TS method"), and
`docs/ruby-ts-conventions.md` drives `parity:api` off file paths, so the
aggregation file also costs path-level matching. It is why the call-mismatch
baseline shard for these methods is
`scripts/api-compare/call-mismatches-exclude/activesupport/array-utils.json`
rather than one shard per Rails file.

Surfaced while converging the `core_ext/array/*` assertion parity in PR #6620,
which touched `wrap`, `inGroupsOf` and `inGroups` in this file and had to point
their JSDoc at `grouping.rb`/`wrap.rb` by hand.

## Converged shape

One TS file per Rails file, each named by the conventions' path rules:

- `core-ext/array/wrap.ts` — `Array.wrap` (a static, as in Ruby)
- `core-ext/array/grouping.ts` — `inGroupsOf`, `inGroups`, `split`
- `core-ext/array/extract.ts` — `extractBang`
- `core-ext/array/conversions.ts` — `toSentence`, `toFs`, `toFormattedS`

Note `core-ext/array/conversions.test.ts` already exists and currently has no
non-test counterpart in that directory, which is the same gap seen from the
test side.

`kernelArray` is NOT part of this move: it is Ruby core (`Kernel#Array`), not a
Rails core_ext, and already carries `@noRailsEquivalent PERMANENT`. Give it a
home that does not imply an Array core_ext mirror.

Re-point the call-mismatch baseline shard(s) at the new paths as part of the
move — the rows are only-shrink, so they must be carried across, not reseeded.
Related: [[array-utils-renamed-access-and-split-methods]] converges the member
NAMES in this same file; either can land first, but doing the rename first
keeps this move mechanical.

## Acceptance criteria

- Every `core_ext/array/*.rb` member listed above lives in the TS file the
  conventions table produces from its Ruby path, and `array-utils.ts` is gone
  or holds only what has no Array core_ext counterpart.
- `pnpm parity:api` files/methods for activesupport do not drop; the
  call-mismatch baseline rows are moved (not reseeded) and
  `pnpm parity:api:calls` stays green.
- No test renames; `pnpm parity:test` percent for activesupport does not drop.
