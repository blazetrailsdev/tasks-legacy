---
title: "Retire the trails-only _joinClauses store: raw SQL / Arel joins belong in joins_values"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 400
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

trails keeps pre-resolved raw SQL join specs (table + ON) in a `_joinClauses`
store that sits entirely outside `joins_values`. Rails has no such store: every
`.joins` argument — raw SQL strings and Arel join nodes included — lands in the
one `@values[:joins]` array and is partitioned on read by `build_join_buckets`
(`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:1824-1876`).

The split forces trails-only compensation at several sites:

- `emitJoinPlan` emits `_joinClauses` in its own loop before any bucket node
  (`packages/activerecord/src/relation/query-methods.ts`), so they bypass the
  leading-join / join-node routing entirely.
- `seedJoinClauseAliases` (`relation/merged-join-alias-tracker.ts`) exists purely
  to teach the shared `AliasTracker` about tables Rails would have seen through
  `alias_tracker(leading_joins + join_nodes, aliases)`.
- `foldMergeJoins` (`relation/merge-joins.ts:37-38`) copies `_joinClauses`
  unconditionally, ahead of Rails' `return if other.joins_values.empty?` guard —
  a merge whose source has only raw join clauses folds them across where Rails
  would have returned early.
- Every "is this relation joinless" guard has to test `_joinClauses.length`
  alongside `joins_values` (`relation.ts` `pureLeftOuter`, `buildJoins`
  `hasEagerAssocs`, and the `_isPlainRelation`-style checks).

RFC 0027's `fold-named-inner-joins-into-build-join-dependencies` folded plain
`.joins(:assoc)` out of `_joinClauses`; the raw-SQL/Arel remainder was left
behind and is what this story retires.

## Acceptance criteria

- Raw SQL join strings and Arel join nodes live in `joinsValues` alongside every
  other `.joins` argument; `_joinClauses` is gone.
- `emitJoinPlan`'s dedicated `_joinClauses` loop is removed — those joins route
  through `buckets[:leading_join]` / `buckets[:join_node]` like Rails.
- `seedJoinClauseAliases` is removed or reduced to whatever the bucket path
  genuinely cannot supply, with the reason stated at the call site.
- `foldMergeJoins` no longer copies a separate clause store ahead of the
  `joins_values.empty?` guard.
- Ported `joins` / `merge` / `left_outer_joins` relation tests pass unchanged,
  no test renames.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
