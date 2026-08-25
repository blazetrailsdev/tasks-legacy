---
rfc: "0124-arel-surfaced-deviations"
title: "Arel surfaced deviations — narrow slots, invented conversions, and value semantics"
status: active
created: 2026-08-25
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "arel"
  - "activerecord"
  - "activemodel"
clusters:
  - "rails-deviation"
related-rfcs:
  - "0023-surfaced-deviations"
  - "0066-arel-visitor-fidelity"
  - "0117-arel-extra-surface-burndown"
  - "0122-arel-assertion-parity"
---

The arel deviations surfaced during ports, pulled out of the
`0023-surfaced-deviations` catch-all into the package they belong to.

0023 is explicitly the fallback bucket — "best-fit always wins; this exists so a
real finding is never dropped just because it lacks a home". These findings now
have a home. Every story here was re-verified against the tree on 2026-08-25
before it moved.

## Scope

Stories whose SUBJECT is arel. Cross-package sweeps that merely touch
`packages/arel/` stay in 0023 or with their owning package — the Ruby-`inspect`
consolidation, the `no-freeform-comments` eslint hole, the SKIP-mirror
relocation burndown, the `assertion-kinds` mapping, and the activerecord
extra-surface marks are all owned elsewhere and were deliberately left behind.

`0122-arel-assertion-parity` stays separate and active: it is a coherent
test-shape/assertion campaign with its own mark, not a deviation register.

## What the re-verification found

37 arel-labelled open stories in 0023. **Seven did not survive**, closed rather
than carried:

- **Six had already converged**, largely under `0066-arel-visitor-fidelity`,
  `0117-arel-extra-surface-burndown` and the RFC 0115 attribute-set work:
  `SelectManager#intersect`/`#except` now take a `SelectManager` and
  `AttributeSet#deepDup` has no clone cache; `visitArelNodesOptimizerHints`
  emits no leading space; `SubstituteBinds#addBind` reassigns `bind` in place and
  `createStringJoin` passes `to` through; `Window#frame`/`rows`/`range` return
  the framing node; `SqlLiteral#toYAML` is gone in favour of `encodeWith`;
  `SelectManager#join` hands `createJoin` the raw relation.
- **One was a duplicate**: two stories filed the same `Nodes.buildQuoted`
  namespace gap. The survivor is the one carrying the RFC 0117 gate finding from
  PR #7016, which is the actual blocker.

Two more carried stale citations that were corrected in the move rather than
left for the claimant: `rubyToS` has been renamed `toS`, and `Attribute.name`
has already widened to `string | SqlLiteral` (the null arm and the cast are what
remain).

## Clusters

**Slots typed narrower than the Ruby slot** — the largest group, and the one
whose acceptance signal is an `as unknown as` cast in a mirrored test. `Table#get`
refuses a node name, `NamedFunction#expressions` and `coalesce` refuse raw
values, `Cte` refuses a manager, `Table#name` refuses a `SqlLiteral`,
`Attribute.name` refuses the null `Table#[]` can produce, and `Nodes::Case`
declares `readonly` where Ruby has `attr_accessor`. Each one forces a cast that
hides the mismatch from the compiler.

**Invented conversions and sentinels in the visitors** — `String(o.expr)`
fallbacks that stringify what the registry would have visited (a wrong-SQL bug,
not cosmetics), the `o.name === "*"` star sentinel, the `toS` wrap on
`quote_table_name`/`quote_column_name` that belongs one layer down in each
adapter's quoting module, and the six-term `instanceof` paren chain both `Cte`
visitors carry because `Cte.relation` holds a statement where Rails holds a
manager.

**Missing arms and missing factory dispatch** — `buildQuoted` has no
`Arel::Table` arm and wraps an `ActiveModel::Attribute` Rails returns unwrapped;
`Arel.sql` ports only the first of its two arms, so the `?` / `:key` bound form
is unrepresentable; `Nodes.buildQuoted` is absent from the namespace Ruby
exposes it on.

**Value semantics** — `SqlLiteral#eql` compares by `stableSerialize` including
`retryable` where Ruby's `String#==` compares SQL text, and
`between`'s degenerate-range arm uses `===` where Ruby uses `==`, so two equal
`Date` bounds do not collapse to an `Equality`.

**Invented public surface** — the `Rollup` alias beside Rails' `RollUp`,
`UnsupportedVisitError` where Rails raises a plain `TypeError`, the `star`
module constant where Rails has a per-call method, `predications`' three
`between` call shapes and their `parseRange` normalizer, and the base `ToSql`
handlers for four nodes only `postgresql.rb` visits.

**Module graph** — `nodes/index.ts` and `visitors/postgresql.js` throw
`Cannot access 'TableAlias' before initialization` when entered as ESM entry
modules. Re-measured 2026-08-25: 286 import cycles still pass through
`nodes/table-alias.ts` and no slot module breaks them.

**Test-body fidelity** — the arel suite was never swept for weakened bodies the
way activerecord was, and one weakened `join_sources` body was concealing three
separate visitor divergences (#5631). Plus two `describe("equality")` bodies in
`over.test.ts` that assert nothing about `Over` equality, and three duplicate
`describe("where_sql")` blocks in `select-manager.test.ts`.

## Working rule

`packages/arel` is gated by the RFC 0117 extra-surface ratchet
(`pnpm parity:api:extra:gate`, arel novel 0 / total 63) and that gate is
only-shrink with no reseed. Several stories here remove public names, which is
the direction the gate wants; at least one (`Nodes.buildQuoted`) adds one and is
blocked on the extractor matching it to `arel/nodes/casted.rb` rather than on
raising the mark. Check the gate before and after, and never widen it.
