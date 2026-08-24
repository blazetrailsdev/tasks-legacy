---
title: "converge-sync-eager-builders-async-to-sql"
status: blocked
updated: 2026-08-24
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 6
pr: null
claim: "2026-08-17T22:06:05Z"
assignee: "converge-lock-value-stores-locks-not-clause-string"
blocked-by: "Re-verified against origin/main 2026-08-24: blocker still live, and wider than written. Relation#toSql is still declared `toSql(): string` (relation.ts:1915) and the sync eager builders are all still present (_applyEagerJoinDependency relation.ts:1974, _buildEagerOperandManager :1924, _materializeDeferredDistinctPkPredicates :1857, and relation/predicate-builder/deferred-distinct-pk-in.ts). The call-site count in the body has DRIFTED UPWARD: toSql() is now read by 683 test and 31 non-test call sites (body says 412/16), so the sync-to-async flip is a bigger PR than when filed. Still needs an async toSql (or a sync query seam) plus its own three-lane adapter run; does not fit this RFC's 700 LOC ceiling and wants its own RFC."
closed-reason: null
---

## Context

Residue of `converge-apply-join-dependency-sync-eager-residue`, which converged
the `eager_loading?` half of that story (`Relation#isEagerLoading` and
`#joinedIncludesValues` now mirror `relation.rb:1237-1249` verbatim;
`_promotedIncludes`, `_includesToPromoteFromReferences`,
`_includesToPromoteFromJoins`, `_joinedIncludesValues`, `_eagerLoadingForSql`
and `_deferredDistinctPkEagerSpecs` are deleted; `exec_queries`' preload list is
Rails' `preload += includes_values unless eager_loading?`, relation.rb:1321-1322;
`calculations.ts#hasInclude` is `calculations.rb:431`; and the composite-PK arm
of `_eagerLoadBypassesJoinDependency` is retired down to the single synchronous
`_buildEagerOperandManager` site).

What did NOT converge is the reason the story was written: the invented
synchronous eager builders in `relation.ts`, kept alive by three entry points
Rails runs synchronously because Ruby can execute SQL synchronously.

Still present (line numbers against the head of the PR that closed the parent
story):

- `_applyEagerJoinDependency`, `_eagerJoinDependencyIsLimitable`,
  `_buildEagerIdSubquery`, `_materializeLimitedIds`,
  `_distinctSelectForLimitedIds`, `_buildEagerOperandManager`
- the deferred distinct-PK marker cluster: `_isDeferredDistinctPkSubquery`,
  `_buildDeferredDistinctPkInlineSubquery`, `_materializeDistinctPkIds`,
  `_materializeDeferredDistinctPkPredicates`, and
  `relation/predicate-builder/deferred-distinct-pk-in.ts`
- the two `@nie disposition=TODO` `NotImplementedError` sites added by #6634:
  `relation/predicate-builder/relation-handler.ts` and `buildFrom` in
  `relation/query-methods.ts`

The three synchronous entry points are `Relation#toSql`
(`relation.rb:1210-1222`), `PredicateBuilder::RelationHandler#call`
(`predicate_builder/relation_handler.rb:8`) and `QueryMethods#build_from`
(`query_methods.rb:1789`). Rails routes all three through
`apply_join_dependency` (`finder_methods.rb:457-481`), whose
`distinct_relation_for_primary_key` branch EXECUTES a query
(`schema_statements.rb:1429-1452`); trails' `applyJoinDependency` is therefore
async, so the sync callers either duplicate the builder or raise.

Why it was not done in the parent PR: the only convergences that delete these
helpers are (a) an async `Relation#toSql` — 412 test call sites plus 16 source
call sites treat it as sync — or (b) an async `where`, which is not on the table.
The marker cluster additionally exists because MySQL/MariaDB reject
`IN (SELECT … LIMIT n)`, which is what the synchronous inline-subquery fallback
emits, so deleting it without an async path is a real MySQL regression, not a
tidy-up. That decision needs its own PR and its own adapter-lane run.

Added by #6648: `_buildEagerOperandManager` also now carries the last
composite-PK residue. #6648 retired the composite-PK arm of
`_eagerLoadBypassesJoinDependency` (the async path handles a composite key via
the adapter's `distinct_relation_for_primary_key`,
`schema_statements.rb:1429-1452`), but this synchronous builder substitutes an
inline `pk IN (SELECT DISTINCT …)` for the executed limited-ids query and a
composite key has no single column to nest that under, so it returns null and
falls back to the plain arel. Deleting the builder deletes that residue too —
there is no separate story for it.

## Acceptance criteria

- Decide and implement the convergence for the three synchronous entry points:
  an async `toSql`/`where` path, or a documented, single-sited
  language-shortcoming deviation.
- Every helper listed above is deleted from `relation.ts`, along with
  `relation/predicate-builder/deferred-distinct-pk-in.ts`.
- Both `@nie disposition=TODO` `NotImplementedError` sites are gone; Rails
  raises at neither.
- `pnpm parity:api:calls` / `:args` clean; `parity:api` / `parity:test` deltas
  non-negative; green on all three adapter lanes.
