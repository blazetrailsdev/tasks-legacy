---
rfc: "0107-relation-ts-decomposition"
title: "relation.ts decomposition and invented-machinery burndown"
status: superseded
created: 2026-08-16
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
clusters:
  - "api-compare"
related-rfcs:
  - "0022-relation-arel-ast-convergence"
  - "0027-join-dependency-fidelity"
  - "0084-wide-call-set-burndown"
priority: 1
superseded-by: "0123-blocked-convergence-holding"
---

## Summary

`packages/activerecord/src/relation.ts` is **7,441 lines** against
`vendor/rails/activerecord/lib/active_record/relation.rb`'s **1,502** — a 5x
ratio, the worst in the data layer. This RFC burns that down.

The ratio is _not_ explained by the port of `relation.rb` being verbose. It is
explained by two things: **~3,400 lines of machinery with no Rails counterpart
anywhere**, and **~1,900 lines that belong in a Rails sibling module** whose TS
file already exists.

## The measurement (2026-08-15)

Classified all 473 members of `relation.ts` by which Rails file defines the
counterpart (Ruby-name match against `relation.rb` + `relation/**/*.rb`,
including the `VALUE_METHODS`-generated accessors), summing each member's line
span:

| bucket                                                                             | lines           | members |
| ---------------------------------------------------------------------------------- | --------------- | ------- |
| **no Rails counterpart anywhere**                                                  | **3,407 (46%)** | 150     |
| genuinely `relation.rb`                                                            | 1,745           | 107     |
| belongs in `relation/query_methods.rb`                                             | 1,226           | 144     |
| belongs in `relation/batches.rb`                                                   | 265             | 6       |
| belongs in `relation/finder_methods.rb`                                            | 258             | 41      |
| belongs in `relation/calculations.rb`                                              | 192             | 10      |
| other siblings (`spawn_methods`, `predicate_builder`, `from_clause`, `delegation`) | ~170            | 15      |

**The `relation.rb`-counterpart content is 1,745 lines against Rails' 1,502 —
roughly 1:1.** That part of the port is healthy and is not in scope here.

Note also that `pnpm parity:api:extra --package activerecord` scores
`relation.ts` at only **10 novel / 5 moved** public names. Invented _public_
surface is not the problem, which is why this bloat has never surfaced on a
gate: the invented surface is **74 private `_`-prefixed helpers** (82 total
`_`-prefixed methods, of which only ~8 — `_new`, `_create`, `_createBang`,
`_scoping`, `_execScope`, `_substituteValues`, `_incrementAttribute`, `_clone`
— carry a real Rails name). `parity:api:extra` cannot see private members, and
the call-set gate only scores calls _inside_ Rails-named bodies, so an entire
invented subsystem is invisible to both.

For reference, the call-set baseline for this file is small and not the story:
29 rows in `scripts/api-compare/call-mismatches-exclude/activerecord/relation.json`.

## The five findings

### F1 — two parallel Arel builders; the faithful one is dead code

`packages/activerecord/src/relation/query-methods.ts:2625` `buildArel()` is a
near-line-for-line port of `query_methods.rb:1651` `build_arel`: joins, where,
having, take/skip, group, `build_order`, `build_with`, `build_select`,
`optimizer_hints`, `distinct`, `from`, `lock`, annotate-dedup. It is correct.

**Nothing calls it.** The only reference in the package is a
`typeof resolved.buildArel === "function"` duck-check at
`relation/query-methods.ts:2449`. The live path is a hand-rolled
reimplementation in `relation.ts`:

- `relation.ts:5315` `_buildSelectManager()` — does `build_arel`'s job with
  invented decomposition: `_applyJoinsToManager` (174 lines),
  `_applyWheresToManager`, `_applyOrderToManager`, `_buildProjections`,
  `_buildFromNode`, `_combineNodes`, `_collectAllWhereNodes`
- `relation.ts:4954` `_buildArel()` — **shadows the Rails name** while calling
  `_buildSelectManager` + `_applyCtesAndAnnotationsToManager`
- `relation.ts:7134` `buildArel()` — a wrapper that _does_ delegate to
  `_qm.buildArel`, and is itself unreached

Live callers of the invented path: `toSql()` (`relation.ts:4984`), `_toSql()`
(`:5039`), `toArel()` (`:4716`), `arel()` (`:6048`), `_cteBodyArelNode()`
(`:5529`), `execMainQuery` (`:6947`). Plus a bespoke compile layer —
`_compileSelectSql` (`:5465`), `_compileAstWithBinds` (`:5535`),
`_typeCastBinds` (`:5546`), `_applyBindLimitFallback` (`:5496`),
`_arelVisitor` (`:5429`), `_selectVisitor` (`:5438`) — where Rails just calls
`connection.to_sql(arel)`.

Cluster total: **~700 lines**, zero Rails counterpart.

### F2 — relation.ts hand-resolves association joins, duplicating JoinDependency

`relation.ts:1556-1946`: `_resolveAssociationJoin` (105 lines, `:1645`),
`_resolveThroughJoin` (153, `:1750`), `_resolveHabtmJoin` (48, `:1903`),
`_resolveHasManyJoin` (41, `:733`), `_resolveHasManySubquery` (35, `:698`),
`_deriveForeignKey` (43, `:1590`), `_appendAssociationScope` (34, `:1556`),
`_isAssociationName` (`:1633`), `_isNamedJoinValue` (64, `:414`).

These inspect `assocDef.options` to build FK predicates and join clauses by
hand. Rails does none of this in `relation.rb` — it is
`associations/join_dependency.rb` + `join_dependency/join_association.rb`,
and **trails already has a `JoinDependency`** (constructed at
`relation.ts:2669` for the eager path).

It is also leaking outward: `_isNamedJoinValue` is now read by
`associations/association-scope.ts:1007` and `relation/merger.ts:162`, and
`_applyJoinsToManager` is depended on by `relation/calculations.ts:177`.

Cluster total: **~600 lines**, zero Rails counterpart.

### F3 — eager-load limit / distinct-PK machinery: ~400 lines for Rails' ~30

`_executeEagerLoad` (144, `relation.ts:2673`), `_buildEagerIdSubquery` (47,
`:5158`), `_materializeDeferredDistinctPkPredicates` (50, `:4904`),
`_distinctSelectForLimitedIds` (45, `:5233`), `_eagerJoinDependencyIsLimitable`
(36, `:5122`), `_eagerLoadBypassesJoinDependency` (49, `:3096`),
`_materializeDistinctPkIds` (`:4877`), `_deferredDistinctPkEagerSpecs`
(`:4827`), `_isDeferredDistinctPkSubquery` (`:4846`),
`_buildDeferredDistinctPkInlineSubquery` (`:4864`), `_buildEagerSql` (`:5278`),
`_buildEagerOperandManager` (`:5291`), `_applyEagerJoinDependency` (44,
`:5078`), `_buildEagerJoinDependency` (`:2669`), `_materializeLimitedIds`
(`:5205`), `_includesToPromoteFromReferences` (`:2507`),
`_includesToPromoteFromJoins` (`:2541`), `_joinedIncludesValues` (`:2521`),
`_resolveAssocTables` (`:2552`), `_aliasableReferences` (`:2574`),
`_eagerLoadingForSql` (`:5058`).

Rails: `finder_methods.rb:462-481` `apply_join_dependency` and
`distinct_relation_for_primary_key` — about 30 lines. trails also carries an
invented async twin `_applyJoinDependencyAsync` (`relation.ts:4796`) alongside
`applyJoinDependency` (`:4738`), already baselined in `relation.json`.

Cluster total: **~400 lines**.

### F4 — Rails' `@values` hash flattened into ~25 bespoke private fields

`query_methods.rb:162-183` generates every `*_values` / `*_value` /
`*_clause` reader and writer from `Relation::VALUE_METHODS` in 20 lines of
Ruby. Each reader is `@values.fetch(:name, default)`; each writer is
`assert_modifiable!; @values[:name] = value`.

trails stores each value in a dedicated private field and hand-writes the
accessors at `relation.ts:4090-4395` (~300 lines). The fields do not carry the
Rails names: `_orderClauses` (not `order_values`), `_includesAssociations` (not
`includes_values`), `_isDistinct` (not `distinct_value`), `_selectColumns` (not
`select_values`), `_eagerLoadAssociations`, `_groupColumns`, `_annotations`,
`_ctes` (not `with_values`), `_optimizerHints`.

The consequences fan out: `values()` (`:6268`), `valuesForQueries()` (`:6304`),
`only()` (`:1400`), `except()` (`:1432`), `slice()` (`:6099`), and
`_copyStateFrom` (58 lines, `:6766`) all have to enumerate the fields by hand
where Rails just slices a hash. `relation.rb:1282` `values` is
`@values.dup` — one line.

### F5 — ~330 lines of pure `this`-rebinding thunks

`relation.ts:6967-7295` is ~60 one-line wrappers of the form:

```ts
private buildWhereClause(opts: unknown, rest: unknown[] = []): unknown {
  return _qm.buildWhereClause.call(this as any, opts, rest);
}
```

They exist so private helpers implemented in `relation/query-methods.ts`,
`relation/finder-methods.ts`, `relation/batches.ts` and
`relation/spawn-methods.ts` are reachable as `this.x()`. Rails gets this for
free from `include QueryMethods, FinderMethods, ...` (`relation.rb:68`).

CLAUDE.md already prescribes the trails answer for instance methods mixed in
bulk — `include()` / `Included<>` from `@blazetrails/activesupport` — and
`relation.ts` already uses it for the public surface (the
`export interface Relation<T>` merge blocks at `:7297`, `:7324`, `:7385`). The
private half was never migrated.

### F6 — ~1,900 lines parked in the wrong file

Members whose Rails counterpart lives in a sibling module, when the
corresponding TS sibling file already exists:

- **`query_methods.rb` → `relation/query-methods.ts`** (1,226 lines, 144
  members): `where` (`:515`), `rewhere` (`:602`), `whereNot` (`:774`),
  `whereAssociated` (`:629`), `whereMissing` (`:660`), `or` (`:856`), `and`
  (`:867`), `order` (`:976`), `limit` (`:988`), `offset` (`:997`), `select`
  (`:1011`), `reselect` (`:1033`), `distinct` (`:1044`), `group` (`:1065`),
  `having` (`:1076`), `regroup` (`:1091`), `reorder` (`:1101`), `reverseOrder`
  (`:1113`), `inOrderOf` (76 lines, `:1124`), `invertWhere` (`:1200`), `none`
  (`:1262`), `lock` (`:1271`), `readonly` (`:1280`), `strictLoading` (`:1325`),
  `annotate` (`:1334`), `optimizerHints` (`:1344`), `from` (`:1366`),
  `createWith` (`:1375`), `unscope` (`:1384`), `extending` (`:1414`), `joins`
  (`:1455`), `leftJoins`/`leftOuterJoins` (`:1485`/`:1495`), `includes`
  (`:1951`), `preload` (`:1961`), `eagerLoad` (`:1971`), `with` (`:5708`),
  `withRecursive` (`:5731`), `references` (`:5750`), plus the whole F4
  accessor block.
- **`batches.rb` → `relation/batches.ts`** (265 lines): `findInBatches`
  (`:4444`), `findEach` (`:4498`), `inBatches` (148 lines, `:4551`).
- **`finder_methods.rb` → `relation/finder-methods.ts`** (258 lines): `exists`
  (73, `:3328`), `applyJoinDependency` (58, `:4738`), `include` (`:6185`),
  plus the thunk half of F5.
- **`calculations.rb` → `relation/calculations.ts`** (192 lines): `ids` (88,
  `:3613`), `pick` (`:2817`), `pluck` (`:3452`) + `_pluckInner` (153, `:3460`),
  the `async*` readers (`:3401-3443`).
- **`spawn_methods.rb` → `relation/spawn-methods.ts`** (42 lines): `only`
  (`:1400`), `except` (`:1432`), `relationWith` (`:7286`).

Beyond fidelity, this blocks `blazetrails/rails-file-structure-method-order`
from doing anything useful for `relation.ts`: the lint orders members against
the Rails file's source order, and half the members here belong to a different
Rails file.

## Non-goals

- Changing SQL output or public behavior. Every story is a refactor with the
  existing suites as the contract.
- Touching the `relation.rb`-counterpart 1,745 lines, which measure ~1:1.
- Reseeding any baseline. `relation.json`'s 29 rows shrink only as bodies
  converge; the reseed rule in CLAUDE.md applies unchanged.

## Sequencing

F5 and F6 are largely mechanical and unblock the rest — once the private
helpers live on the class via `include()` and the parked members sit in their
Rails file, the F1/F2/F3 clusters become locally reviewable. F1 must land
before F2 and F3, since both feed the builder.

Ordering: **F5 → F6 → F1 → F2 → F3 → F4**, with F4 last because its blast
radius (every sibling module reads the private fields) is widest and it is
cheapest once the file has stopped moving.
