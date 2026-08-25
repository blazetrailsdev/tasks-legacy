---
title: "DisableJoinsAssociationScope walks the chain eagerly; DJAR sheds its count override"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 450
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `DisableJoinsAssociationScope#scope`
(`vendor/rails/activerecord/lib/active_record/associations/disable_joins_association_scope.rb:6-15`)
walks the through-chain EAGERLY: `last_scope_chain` (:18-31) runs the
intermediate `records.pluck(foreign_key)` inline, so by the time `scope` returns,
`add_constraints` (:33-56) has already produced a relation carrying
`key IN (ids)`. Every inherited `Relation` query method on it — `count`,
`calculate`, `pluck`, `sum` — is therefore correct with no override, and Rails'
`DisableJoinsAssociationRelation`
(`vendor/rails/activerecord/lib/active_record/disable_joins_association_relation.rb`)
defines only `ids`/`key` readers plus `limit`, `first` and `load`.

trails' plucks are async, so `DisableJoinsAssociationScope#scope`
(`packages/activerecord/src/associations/disable-joins-association-scope.ts:111-157`)
cannot walk the chain inside a sync method. It returns a **deferred-mode**
`DisableJoinsAssociationRelation`
(`packages/activerecord/src/disable-joins-association-relation.ts`, `static deferred`)
that carries no `key IN (ids)` until something awaits it, and pays for that with
a `count` override (`disable-joins-association-relation.ts:414-441`) that runs
`_walkOnce()` + `_composeChainedState()` before delegating. Rails has no such
override.

Measured on PR #6613's branch: deleting the `count` override reds
`HasManyThroughDisableJoinsAssociationsTest > counting on disable joins through`
and `… counting on disable joins through using custom foreign key` at 14 rather
than 3 — `Relation.prototype.count` counts the whole table, because the deferred
relation is unconstrained. Any other inherited calculation
(`calculate`, `pluck`, `sum`, `minimum`) is silently wrong on a `disable_joins`
association today for exactly the same reason; `count` is only the arm a test
happens to cover.

The predecessor story `djar-eager-chain-ids-drop-disable-joins-arms`
(0106-wide-call-set-direct-burndown) targeted the `CollectionProxy#calculate` /
`#pluck` `disableJoins` arms, which turned out never to have merged; it closed
with no diff. This story is the part of that deviation that is genuinely still
open.

## Converged shape

The chain walk becomes eager, which means `Association#scope` (and the
`_scope-slots.ts` `DjasScopeFn` seam it reads) has to be able to await — the
whole reason the deferred mode exists. Then:

- `DisableJoinsAssociationScope#scope` returns the constrained relation
  `add_constraints` built, matching disable_joins_association_scope.rb:6-15.
- `DisableJoinsAssociationRelation` sheds deferred mode entirely: no
  `static deferred`, no `_chainWalker` / `_walkPromise` / `_walkOnce` /
  `_composeChainedState`, and no `count` override — leaving only Rails'
  `ids`/`key`, `limit`, `first`, `load`.
- Every inherited calculation on a `disable_joins` association becomes correct
  by construction rather than per-method.

Scope the ripple first: `Association#scope`'s sync callers are the real cost
here, and if the flip cannot be contained, `pnpm tasks block` with the specific
caller set rather than re-ratifying the override.

## Acceptance criteria

- [ ] `DisableJoinsAssociationScope#scope` returns a relation already carrying
      `key IN (ids)`, matching disable_joins_association_scope.rb:6-15.
- [ ] `DisableJoinsAssociationRelation` defines only the members Rails'
      `disable_joins_association_relation.rb` defines; the `count` override and
      the deferred-mode machinery are gone.
- [ ] A regression test covers a non-`count` calculation (e.g. `sum` or `pluck`)
      over a `disable_joins` through association, and fails on the current
      baseline.
- [ ] `has-many-through-disable-joins-associations.test.ts`,
      `has-one-through-disable-joins-associations.test.ts`,
      `cp-count-disable-joins-through.test.ts` and the `disable-joins-*` cluster
      green on SQLite, PostgreSQL and MySQL/MariaDB.
- [ ] `pnpm parity:api:calls` / `:args` / `:extra` clean, no new baseline rows.
