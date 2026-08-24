---
title: "AbstractMysqlAdapter has two extractPrecision definitions; Rails has one"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails declares `extract_precision` for MySQL exactly once, inside
`AbstractMysqlAdapter`'s `class << self ... private` block
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:751-757`),
where it wraps `super` so the `(date)?time(stamp)?` family resolves a missing
`(N)` modifier to 0 instead of nil.

trails now has two. PR #5541 added the static
`AbstractMysqlAdapter.extractPrecision`
(`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts`, just
after `registerIntegerType`) because the base type map's
`registerClassWithPrecision` reads precision through the _static_ hook — that is
the one Rails models. But a pre-existing `protected extractPrecision(sqlType):
number | null` instance method also lives on the same class (near the end of the
class body, below `buildStatementPool`) with the same rule and a different
nullish convention (`null` vs `undefined`). Rails has no instance-level
counterpart.

The duplicate was left in place by #5541 because its callers were not audited
and the PR was scoped to the type-map `super` convergence.

## Acceptance criteria

- [ ] Enumerate the callers of the instance `extractPrecision` on
      `AbstractMysqlAdapter` and its subclasses.
- [ ] Delete the instance method and re-point its callers at the static one, or
      — if some caller genuinely needs an instance hook — record why at the call
      site and reconcile the `null`/`undefined` return conventions so the two
      cannot drift.
- [ ] `parity:api` for `connection_adapters/abstract_mysql_adapter.rb` and
      `parity:api:extra` must not regress.
- [ ] `connection-adapters/type-lookup.test.ts` and
      `connection-adapters/mysql-type-lookup.test.ts` stay green on the mysql2
      lane.
