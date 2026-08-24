---
title: "excluding defers relations.flat_map(&:ids) to load time instead of materializing eagerly"
status: blocked
updated: 2026-08-24
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps:
  - port-with-connection-acquisition-seam-for-the-arel-reader
deps-rfc: []
est-loc: 150
priority: 7
pr: 6928
claim: "2026-08-23T17:39:45Z"
assignee: "converge-excluding-deferred-ids-marker-to-eager-materialization"
blocked-by: "Loaded arm converged in PR #6928 (option 2 of the story: relation args that are already loaded? materialize eagerly, no marker, literal ids in toSql). The un-loaded arm is blocked: ids runs the SELECT (calculations.rb:390-404) while excluding is a synchronous chainable and every adapter select_all is Promise-returning (database-statements.ts:1293/1375); toSql(): string is read inline by 26 non-test callers so it cannot await either. Needs a synchronous query seam — same blocker as port-with-connection-acquisition-seam-for-the-arel-reader."
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
