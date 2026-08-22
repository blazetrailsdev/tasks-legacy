---
title: "Map Arel::Math operator methods to their TS spellings in conventions.ts"
status: done
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: ["arel"]
deps: []
deps-rfc: []
est-loc: 90
priority: 1
pr: 6856
claim: "2026-08-22T12:20:33Z"
assignee: "arel-operator-spellings-in-conventions"
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:api:extra --package arel` (2026-08-22) reports 89 novel names.
**36 of them are the same 9 names repeated across 4 files**:

| file                                                | novel names                                                                                                                      |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `packages/arel/src/math.ts:33-70`                   | `multiply`, `subtract`, `divide`, `bitwiseAnd`, `bitwiseOr`, `bitwiseXor`, `bitwiseNot`, `bitwiseShiftLeft`, `bitwiseShiftRight` |
| `packages/arel/src/attributes/attribute.ts:115-140` | same 9                                                                                                                           |
| `packages/arel/src/nodes/node-expression.ts`        | same 9                                                                                                                           |
| `packages/arel/src/nodes/infix-operation.ts`        | same 9                                                                                                                           |

They are **not invented surface**. They are `Arel::Math`'s operator methods:
`vendor/rails/activerecord/lib/arel/math.rb:5-40` defines `*`, `+`, `-`, `/`,
`&`, `|`, `^`, `<<`, `>>`, `~@`.

They land in the extra population because
`scripts/parity/conventions.ts:1478` (`rubyMethodToTsWithoutUnderscore`)
returns `null` for every name in `OPERATORS`
(`scripts/parity/conventions.ts:346-368`), so the Ruby side produces no
expected TS name and the TS spelling has nothing to match.

The precedent for the fix is four lines below, in the same function:
`conventions.ts:1486` maps `-@` to `negate`.

Each of the four files additionally shows `add` as **moved** — `+`'s TS
spelling matching some unrelated Ruby `add`. Mapping `+` correctly retires
those 4 rows too.

## Approach

Add a **file-scoped** Ruby-operator → TS-name table to
`scripts/parity/conventions.ts`, keyed on the defining Ruby file
(`arel/math.rb`), in the same shape as `SCOPED_SKIP_GROUPS`:

```text
* → multiply   + → add        - → subtract   / → divide
& → bitwiseAnd | → bitwiseOr  ^ → bitwiseXor
<< → bitwiseShiftLeft  >> → bitwiseShiftRight  ~@ → bitwiseNot
```

**Scoped, not global** — `<<` means _append_ on `SelectManager` and in the
collectors, so a global entry would mis-credit those.

`docs/ruby-ts-conventions.md` is generated from `conventions.ts`
(`conventions.ts:1610` already renders the `OPERATORS` list) — regenerate it,
never hand-edit, and extend the generated section to render the new table.
`scripts/parity/conventions.test.ts` gets cases for a scoped hit and for
`<<` staying unmapped outside `arel/math.rb`.

## Acceptance criteria

- `pnpm parity:api:extra --package arel` novel drops **89 → 53**, total drops
  **258 → ~218**; `math.ts`, `nodes/node-expression.ts`, and
  `nodes/infix-operation.ts` fall out of the drift list or retain only their
  non-operator rows.
- No change to any file under `packages/arel/src/`.
- `docs/ruby-ts-conventions.md` regenerated from source; its conventions test
  passes.
- `pnpm parity:api` method totals non-negative (the mapping adds matches, so
  the arel matched-method count should _rise_).
- No new `@noRailsEquivalent` tag.
