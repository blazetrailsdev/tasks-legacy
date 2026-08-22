---
title: "ids' eager arm routes through pluck because applyJoinDependency early-returns"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Relation#ids` (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:387-390`)
handles the eager arm as:

```ruby
if has_include?(primary_key)
  relation = apply_join_dependency.group(*primary_key_array)
  return relation.ids
end
```

Rails' `apply_join_dependency` (`finder_methods.rb:456-488`) ALWAYS builds the
join dependency and strips the eager values
(`except(:includes, :eager_load, :preload).joins!(join_dependency)`), so the
recursive `relation.ids` falls through to the query arm.

trails' `Relation#applyJoinDependency` (`packages/activerecord/src/relation.ts`,
~line 4669) opens with `if (!this._eagerLoadingForSql()) return this;`. For a
non-referencing `includes` — `eager_loading?` false, `includes_values.present?`
true — `has_include?` is true but `applyJoinDependency` is a no-op, so writing
Rails' line verbatim recurses forever (verified: `RangeError: Maximum call stack
size exceeded` on `ids with includes offset`).

PR #6564 ported the rest of `ids` faithfully and routed only this arm through
`pluck(...primaryKeyArray)`, which already owns the eager-spec expansion and the
`distinct_relation_for_primary_key` materialization (`finder_methods.rb:463`)
that limit/offset needs. The divergence is documented at the call site
(`relation.ts`, `ids`, `DIVERGENCE (calculations.rb:387-390)`), and `pluck`'s own
eager arm carries the same shape.

## Converged shape

Make `applyJoinDependency` build the join dependency whenever `has_include?`
would be true — i.e. drop the `_eagerLoadingForSql()` early return in favour of
Rails' unconditional `construct_join_dependency` + `except(:includes,
:eager_load, :preload)` — then restore `ids`' arm to Rails' two lines:

```ts
const relation = this.applyJoinDependency().group(...primaryKeyArray);
return relation.ids();
```

`pluck`'s hand-rolled eager arm (`relation.ts`, `_pluckInner`) collapses onto the
same call once that holds, which is the larger prize here.

## Acceptance criteria

- [ ] `applyJoinDependency` no longer early-returns `this` for a non-referencing
      `includes`; it mirrors `finder_methods.rb:456-488`.
- [ ] `ids`' eager arm is Rails' `apply_join_dependency.group(*primary_key_array)`
      recursion, and the `DIVERGENCE` comment in `ids` is deleted.
- [ ] `ids with includes offset` / `ids with includes limit and empty result` /
      `ids with includes and non primary key order` stay green on SQLite, PG and
      MySQL/MariaDB.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
