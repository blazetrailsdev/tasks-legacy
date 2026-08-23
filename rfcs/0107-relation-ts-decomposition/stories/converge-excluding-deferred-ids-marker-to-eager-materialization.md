---
title: "excluding defers relations.flat_map(&:ids) to load time instead of materializing eagerly"
status: claimed
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: "2026-08-23T17:39:45Z"
assignee: "converge-excluding-deferred-ids-marker-to-eager-materialization"
blocked-by: null
closed-reason: null
---

## Context

`QueryMethods#excluding` (`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:1583-1588`)
materializes relation arguments EAGERLY:

```ruby
relations.flat_map(&:ids)
```

so by the time `excluding!` calls `predicate_builder[primary_key, records].invert`
the predicate is a literal `id NOT IN (1, 2, 3)`.

trails' builder is synchronous and lazy, so `excluding`
(`packages/activerecord/src/relation/query-methods.ts:1960-1969`) cannot run that
id-select at call time. PR #6922 converged the arm-selection question — every
relation argument now goes through the deferred marker uniformly, so `Relation#ids`
alone decides whether a query runs (calculations.rb:373) and a `load_async`
relation drains rather than re-queries. What is still deviant is the DEFERRAL
itself: `excludingBang` records a `DeferredIdsNotIn`
(`query-methods.ts:2018-2040`) that `Relation#_materializeDeferredDistinctPkPredicates`
(`relation.ts:1872-1891`) substitutes at load time.

Observable consequences:

- `toSql()` on an un-loaded `excluding(relation)` renders the marker's
  `IN (SELECT ...)` display fallback, where Rails renders literal ids.
- The ids are materialized once, at first load, rather than at `excluding` call
  time — so mutating the argument relation's rows between the two points changes
  the result in trails and cannot in Rails.

## Converged shape

Materialize `relations.flat_map(&:ids)` where Rails does. Options, in order of
preference:

1. An async `excluding` is not available (the chainable contract is sync), so
   the realistic shape is to keep the marker but pin its semantics to Rails':
   substitute at the FIRST terminal and make `toSql()` materialize too, so no
   caller observes the subquery fallback.
2. If the sync/async boundary can be crossed at all here — e.g. a relation
   argument that is already `loaded?` needs no await and could build the literal
   predicate immediately — take that arm eagerly and defer only the un-loaded
   ones.

Either way the deviation and its `toSql()` fallback should stop being reachable
from user-visible surface.

## Acceptance criteria

- [ ] `Model.excluding(other)` — `other` un-loaded — renders literal ids from
      `toSql()`, not `IN (SELECT ...)`.
- [ ] No user-visible path reaches `DeferredIdsNotIn`'s display fallback.
- [ ] Existing `excluding.test.ts` / `excluding.trails.test.ts` stay green; the
      trails-only test pinning the deferred arm is updated or retired with the
      deviation.
- [ ] `parity:api:calls` / `:args` clean.
