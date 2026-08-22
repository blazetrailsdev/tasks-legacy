---
title: "unionOrderClauses is a second, differently-equal spelling of Ruby Array#|"
status: draft
updated: 2026-08-22
rfc: "0082-ruby-ts-idiom-conversion-classes"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`unionOrderClauses` (`packages/activerecord/src/associations/association-scope.ts`)
is a trails-only helper with no Rails counterpart. It exists because Rails'
`add_constraints` ends its loop body with the `|` operator:

    scope.order_values = item.order_values | scope.order_values

(`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb:154`,
and the same line in `disable_joins_association_scope.rb:45`). Ruby's
`Array#|` dedups by `eql?`/`hash`, which is STRUCTURAL for the string and
`[col, direction]` tuple shapes `order_values` holds; JS `Array.includes` is
reference equality, so a literal port would double up two equal tuples built
separately.

PR #6886 inlined `add_constraints`' three invented helpers and, to let the
`DisableJoinsAssociationScope` copy of the same Rails line reuse it rather
than duplicate it, exported `unionOrderClauses` as `@internal`. That is the
right call over duplication, but it leaves a trails-only name exported from a
Rails-matched file, and it is the SECOND hand-rolled spelling of Ruby's `|`:
`structuralUnionEq` already lives in `relation/query-methods.ts:1335` (also
`@internal`-exported) doing the same job for `joins_values` /
`left_outer_joins_values`, with its own `deepEqual` rather than
`unionOrderClauses`' string-key set.

Two helpers for one Ruby operator, with two different equality
implementations, is the divergence: whether two order clauses are "the same"
is now answered by a `T:${col}:${dir}` string key in one file and by
`deepEqual` in another.

RFC 0082 (`0082-ruby-ts-idiom-conversion-classes`) enumerates exactly this
kind of Ruby-operator-with-no-JS-equivalent as a conversion class.

## Converged shape

One `Array#|` idiom for the repo, implemented once on `structuralUnionEq`'s
equality (it already handles strings, arrays, plain-object specs and Arel
nodes via `deepEqual`, which subsumes the tuple case `unionOrderClauses`
special-cases), and reached by both order-value call sites and both
join-value call sites. Delete `unionOrderClauses` and its export from
`association-scope.ts`; the two Rails `|` lines then read as the one shared
operator, and the Rails-matched file loses its trails-only exported name.

Verify the equality really is equivalent for order values first: the tuple
key `T:${String(o[0])}:${String(o[1])}` folds two tuples whose first element
stringifies the same (e.g. two distinct Arel nodes with the same `toString`),
where `deepEqual` would compare them structurally. If the behaviours differ on
a real fixture, converge toward `eql?` — that is what Ruby's `|` uses.

## Acceptance criteria

- [ ] `unionOrderClauses` is gone from `association-scope.ts` (name and export).
- [ ] Both `order_values |` sites (association_scope.rb:154,
      disable_joins_association_scope.rb:45) and the join-value union sites go
      through one shared Ruby-`Array#|` idiom.
- [ ] `pnpm parity:api:extra --package activerecord` loses the
      `unionOrderClauses` name and gains none.
- [ ] `packages/activerecord/src/associations/` and `relation/` suites green on
      SQLite, PG and MySQL/MariaDB.
