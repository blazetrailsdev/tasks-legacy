---
title: "AssociationRelation overrides toArray where Rails overrides exec_queries"
status: in-progress
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6912
claim: "2026-08-23T12:57:31Z"
assignee: "converge-association-relation-inverse-wiring-onto-exec-queries"
blocked-by: null
closed-reason: null
---

## Context

Rails' `AssociationRelation` wires the inverse instance and strict-loading
mode by overriding **`exec_queries`**:

```ruby
# activerecord/lib/active_record/association_relation.rb:43-49
def exec_queries
  super do |record|
    @association.set_inverse_instance_from_queries(record)
    ...
  end
end
```

trails hangs the same work off **`toArray`** instead
(`packages/activerecord/src/association-relation.ts:302-383`): it sets
`this._instantiateBlock`, calls `await super.toArray()`, and restores the
previous block in a `finally`.

That placement is load-bearing today only because trails' `load()` calls
`toArray()`. Rails' chain runs the other way — `to_ary` is `records.dup`
(relation.rb:337-339), `records` is `load; @records` (:342-345), and `load`
calls `exec_queries` (:1179-1186) — so in Rails an `AssociationRelation`
loaded via `records()` or `load()` still gets its inverse wiring, while in
trails those entry points only get it by the accident of the inverted chain.

Surfaced while converging the `Relation` loaded-arm readers onto the
`loaded?` / `records` seams (PR #6905): the `to_ary` → `records` → `load`
inversion could not be done because it would route around this override.

## Converged shape

Move the body to `execQueries`, keeping the Rails name and the block form:
the override sets `_instantiateBlock`, calls `super.execQueries()`, and
restores in a `finally`. `toArray` loses its override entirely.

## Acceptance criteria

- `AssociationRelation` overrides `execQueries`, not `toArray`.
- `blog.posts.where(...)` still caches `post.blog = blog` (the
  `set_inverse_instance_from_queries` behaviour) when loaded through
  `toArray()`, `records()` AND `load()`.
- Strict-loading propagation from the owner is unchanged.
- `pnpm parity:api:calls` / `:args` add zero rows.
