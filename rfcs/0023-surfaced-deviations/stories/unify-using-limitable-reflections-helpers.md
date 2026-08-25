---
title: "Collapse the four using_limitable_reflections? spellings into one helper"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the live eager path onto `build_arel` (#5909).

Rails has ONE test for whether an eager limit/offset needs the distinct-pk
rewrite (`finder_methods.rb:463-470`): both `using_limitable_reflections?` over
the eager JoinDependency's reflections AND over
`construct_join_dependency(select_association_list(joins_values).concat(
select_association_list(left_outer_joins_values)), nil).reflections`.

trails now has three helpers spelling that one test, in
`packages/activerecord/src/relation.ts`:

- `_eagerJoinDependencyIsLimitable(jd)` — added by #5909; takes an already-built
  JoinDependency (the eager paths hold one).
- `_applyJoinDependencyIsLimitable(eagerSpecs)` — takes specs and rebuilds a
  throwaway JoinDependency via `_eagerReflectionsAreLimitable`.
- `_eagerReflectionsAreLimitable(specs)` / `_joinsReflectionsAreLimitable()` —
  the two clauses, split.

All four agree today (#5909 made the eager paths use the two-clause form), but
four names for one Rails predicate is exactly the shape that drifts: the bug #5909 fixed was one call site testing `jd.nodes.some(hasMany)` instead of the
pair, and it survived because no single helper owned the rule.

## Acceptance criteria

- One helper expresses Rails' two-clause `using_limitable_reflections?` test,
  taking either a built JoinDependency or specs (resolving specs to a
  JoinDependency internally, as Rails' `construct_join_dependency` does).
- `_eagerLoadBypassesJoinDependency`, `_applyEagerJoinDependency`, the async
  `limitedIds` prefetch guard, and `applyJoinDependency` all call it.
- No behavior change: the composite-PK bypass and both eager branches keep their
  current answers (covered by the `relation.trails.test.ts` limitable pair).
