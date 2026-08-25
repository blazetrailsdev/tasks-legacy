---
title: "SqlLiteral#eql must compare by SQL text, not Node#eql's serialized fields"
status: draft
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "arel"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6911 (RFC 0106, `where-clause-string-predicate-arms`).

`Arel::Nodes::SqlLiteral < String` (`activerecord/lib/arel/nodes/sql_literal.rb:5`),
so its `==` / `eql?` is `String#==` on the SQL text alone — `retryable`
(`sql_literal.rb:11-16`) is NOT part of equality, and a literal compares equal
to a bare String carrying the same text.

trails' `SqlLiteral` extends `Node`
(`packages/arel/src/nodes/sql-literal.ts:14`), so it inherits `Node#eql`
(`packages/arel/src/nodes/node.ts:79-88`), which:

- rejects any non-object outright (`typeof other !== "object"`), so
  `new SqlLiteral("x").eql("x")` is `false` where Ruby says `true`; and
- compares `stableSerialize(this)`, which includes `retryable`, so
  `new SqlLiteral("x", { retryable: true }).eql(new SqlLiteral("x"))` is
  `false` where Ruby says `true`.

PR #6911 worked around the first half locally: `predicateEql` in
`packages/activerecord/src/relation/where-clause.ts` folds a `string` or a
`SqlLiteral` to its SQL text before comparing, so `WhereClause`'s `equals` /
`minus` / `union` dedup the two spellings the way Rails' plain `predicates ==`
/ `-` / `|` (`activerecord/lib/active_record/relation/where_clause.rb:18,22,75-79`)
do. That is a per-call-site patch of a class-level divergence — every OTHER
`SqlLiteral` comparison in the repo still carries both bugs.

## Converged shape

Override `eql` on `SqlLiteral` so it mirrors `String#==`:

```ts
eql(other: unknown): boolean {
  if (typeof other === "string") return this.value === other;
  return other instanceof SqlLiteral && this.value === other.value;
}
```

Then delete `predicateEql`'s text-folding (`where-clause.ts`) along with its
`@noRailsEquivalent` tag and let the three sites call `Node#eql` directly, as
Rails' bare array operators do — with the bare-`string` side still needing a
guard only because a JS primitive has no method to dispatch on.

## Acceptance criteria

- [ ] `SqlLiteral#eql` compares by SQL text: equal to a bare `string` with the
      same text, and equal across differing `retryable` flags.
- [ ] `predicateEql` and its `@noRailsEquivalent` tag are removed from
      `packages/activerecord/src/relation/where-clause.ts`; the three sites
      compare predicates directly.
- [ ] `where-clause-string-predicates.trails.test.ts`'s cross-spelling case
      still passes.
- [ ] `pnpm parity:api:extra:gate` green (arel is gated; the override is a
      Rails-matched name, not novel surface).
