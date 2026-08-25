---
title: "update-all-includes-join-strategy"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/relation/update-all.test.ts` ("update all with
includes") carried a `TRACKED DEVIATION` comment, deleted by the
`no-freeform-comments` sweep over `relation/` (a comment recording deferred work
becomes a story, not a better comment):

> includes + where referencing included table should switch to JOIN strategy.
> Trails `includes` does a separate SELECT so toys.name is not available in the
> WHERE clause.

Rails switches an `includes` to an eager JOIN whenever the relation references
an included table, and `update_all` is one of the call sites that acts on that
decision:

- `activerecord/lib/active_record/relation.rb:1238` — `eager_loading?`:
  `includes_values.any? && (joined_includes_values.any? ||
references_eager_loaded_tables?)`.
- `activerecord/lib/active_record/relation.rb:1474` —
  `references_eager_loaded_tables?`, which is what makes a `where` naming the
  included table (via `references_values`) flip the relation to eager loading.
- `activerecord/lib/active_record/relation.rb:605` — `update_all` branches on
  it: `arel = eager_loading? ? apply_join_dependency.arel : build_arel(c)`.
- `activerecord/lib/active_record/relation/finder_methods.rb:457` —
  `apply_join_dependency`, which constructs the `Arel::Nodes::OuterJoin`
  JoinDependency and `joins!`es it after `except(:includes, :eager_load,
:preload)`.

So `Pet.includes(:toys).where(toys: { name: "Bone" }).update_all(...)` compiles
one joined statement. trails' `includes` runs a separate preload SELECT, so the
`toys.name` predicate is not available to the `UPDATE`.

trails: `packages/activerecord/src/relation/update-all.test.ts:137`
(`it("update all with includes")`); the strategy decision would live in
`packages/activerecord/src/relation.ts` (`updateAll`, relation.ts:1273).

## Acceptance criteria

- [ ] `includes` + a `where` referencing the included table switches to the
      eager-JOIN strategy, matching Rails' `eager_loading?` decision.
- [ ] `UpdateAllTest#update all with includes` passes on all adapter lanes
      without the deviation.
