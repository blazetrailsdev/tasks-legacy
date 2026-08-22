---
title: "Retire arel's root-file and barrel extra-surface residue"
status: claimed
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: ["arel"]
deps:
  [
    "arel-attribute-extra-surface",
    "arel-no-counterpart-invented-files",
    "arel-to-sql-inline-helpers",
  ]
deps-rfc: []
est-loc: 150
priority: 10
pr: null
claim: "2026-08-22T23:08:30Z"
assignee: "arel-root-and-barrel-tail"
blocked-by: null
closed-reason: null
---

## Context

The root-file and barrel residue — the 1-to-3-extra files that are too small
for their own story and share one shape
(`pnpm parity:api:extra --package arel`, 2026-08-22).

**Root files with novel names:**

| file                                       | novel                                 | note                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------ | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `packages/arel/src/table.ts`               | `attr`                                | Rails' `table.rb` has `[]` (`Table#[]`), no `attr`. `[]` is in `OPERATORS`, so this may be the same class of miss the conventions story fixes — check whether `arel-operator-spellings-in-conventions` should have covered `table.rb` too, and extend that scoped table rather than renaming the method |
| `packages/arel/src/errors.ts`              | `NotImplementedError`                 | `errors.rb` defines `EmptyJoinError` and `UnsupportedVisitError` only; `NotImplementedError` is a Ruby _builtin_, so trails must not declare it in `errors.ts`. Check where it is thrown and whether the builtin `Error` shape suffices                                                                 |
| `packages/arel/src/predications.ts`        | `isEnumerable`, `isSelectManagerLike` | type guards Rails gets from `is_a?` / duck typing; fold into their call sites                                                                                                                                                                                                                           |
| `packages/arel/src/visitors/dot.ts`        | `DotEdge`, `DotNode`                  | Rails' `visitors/dot.rb` defines `Node = Struct.new(...)` and `Edge = Struct.new(...)` _inside_ `Arel::Visitors::Dot`. The names differ only by the `Dot` prefix — this is a naming question, not invented surface; check `docs/ruby-ts-conventions.md` for the nested-class convention before renaming |
| `packages/arel/src/visitors/postgresql.ts` | `PostgreSQLWithBinds`                 | sibling of the `to-sql.ts` `compile` split; coordinate with `arel-to-sql-inline-helpers` and skip here if that story retires it                                                                                                                                                                         |

**Barrels** (`index.ts` 3 novel / 4 moved, `visitors/index.ts` 2 novel,
`nodes/index.ts` 1 novel / 1 moved, `arel.ts` 2 moved): every extra is a
re-export of a name owned by another file. `index.ts`'s `relationName`,
`tableRealName`, `tableSqlName` die with `attributes/attribute.ts` and
`table-ref.ts`; `visitors/index.ts`'s `Index` and `substituteBoundValues` die
with `visitors/substitute-bound-values.ts`. **Run this story last** and
re-measure first — most barrel rows should already be gone, and the residue is
the check that no story left a dangling export.

## Acceptance criteria

- Every novel name in the table retired, or — for `attr` and `DotEdge` /
  `DotNode` — shown to be a conventions question and fixed in
  `scripts/parity/conventions.ts` with the doc regenerated.
- Barrels re-measured: `index.ts`, `nodes/index.ts`, `visitors/index.ts`, and
  `arel.ts` report **0 novel**; any residue means an earlier story left a
  dangling re-export, which this story fixes.
- `pnpm parity:api:extra --package arel` reports **0 novel for the whole
  package** once this story lands (it is the last of the burndown stories).
- `pnpm parity:api` arel deltas non-negative; `pnpm vitest run packages/arel`
  green.
- No new tag.
