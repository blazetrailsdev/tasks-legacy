---
title: "toSql renders no eager JOIN for a composite PK with LIMIT/OFFSET over collection reflections"
status: ready
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: ["converge-sync-eager-builders-async-to-sql"]
deps-rfc: []
est-loc: 150
priority: 7
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# The sync eager path still answers `null` for a composite PK with LIMIT/OFFSET over collection reflections

## Context

`fold-eager-id-subquery-builder-onto-build-arel` (PR #6764) folded
`_buildEagerIdSubquery` onto `build_arel`: `Relation#_limitedDistinctRelation`
(`packages/activerecord/src/relation.ts`) is now Rails'
`limited = relation.reselect(values).distinct!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1438`)
over the relation `apply_join_dependency` spawns
(`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:461`).

That story asked for the pre-existing "single-column pk only" limitation to be
re-checked once the two builders were one. Re-checked, and it splits in two:

- The **async/execution** path is fine. `_materializeLimitedIds` maps every pk
  column and `_applyEagerJoinDependency` rewrites per column
  (`Array(relation.primary_key).zip(limited_ids.transpose)`,
  `schema_statements.rb:1448`), so a composite key behaves as Rails does.
- The **synchronous** path still gives up. `_buildEagerOperandManager` returns
  `null` when `Array.isArray(basePk) && hasLimitOrOffset &&
!_eagerJoinDependencyIsLimitable(jd)`, so `Relation#toSql` on such a relation
  renders the plain arel with no eager JOIN and no pk restriction at all —
  SQL that does not describe the query that would actually run. Rails has no
  such arm: `to_sql` is
  `apply_join_dependency { |relation, jd| jd.apply_column_aliases(relation).to_sql }`
  (`relation.rb:1210-1222`), and `apply_join_dependency` executes
  `distinct_relation_for_primary_key` unconditionally.

The cause is the same seam `converge-sync-eager-builders-async-to-sql` is
blocked on: `toSql` is synchronous in trails and cannot execute the limited-ids
query, so it substitutes an inline `pk IN (SELECT DISTINCT …)` — and a
composite key has no single column to nest that under.

## Converged shape

Fall out of the async `toSql` seam: once `toSql` can await, the sync fallback
and its composite-PK `null` arm both delete and `toSql` routes through
`applyJoinDependency` like every other eager entry point. Until then this is
the one remaining shape difference in the folded builder, and it should be
converged with — not separately from — that seam.

Likely a dep on `converge-sync-eager-builders-async-to-sql` rather than
independent work; filed so the composite-PK arm is named rather than
rediscovered when that story unblocks.

## Acceptance criteria

- [ ] `_buildEagerOperandManager` has no composite-PK `null` arm; `toSql` on a
      composite-PK relation with LIMIT/OFFSET over collection reflections
      renders the eager JOIN and the pk restriction Rails renders.
- [ ] `_eagerJoinDependencyIsLimitable` (the sync copy of
      `apply_join_dependency`'s inline guard) is gone or has one caller left.
- [ ] Green on SQLite, PostgreSQL and MySQL/MariaDB.
