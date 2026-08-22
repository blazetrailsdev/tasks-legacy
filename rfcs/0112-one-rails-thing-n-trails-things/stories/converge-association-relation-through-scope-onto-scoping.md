---
title: "AssociationRelation create/build use scoping instead of the _pendingThroughScope field"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing #6388 (`converge-collection-proxy-build-record`).

Rails' `AssociationRelation#_create` (and `#_new`) runs the build inside
`scoping { ... }`
(`vendor/rails/activerecord/lib/active_record/associations/association_relation.rb:38-46`),
so the relation's values are the current scope by the time
`HasManyThroughAssociation#build_record` captures them:

```ruby
@through_scope = scope   # has_many_through_association.rb:92
```

trails has no current-scope stack on this path. Instead
`AssociationRelation#create` writes the relation onto the proxy as
`_pendingThroughScope`
(`packages/activerecord/src/association-relation.ts:151-157`) and
`HasManyThroughAssociation#buildRecord`
(`packages/activerecord/src/associations/has-many-through-association.ts:305-315`)
reaches back for it through `this.owner._collectionProxies.get(name)` before
falling back to `this.scope()`. #6388 moved this read to the association (it was
previously an ad hoc `_isThrough ? this.scope() : this` on the proxy), which is
closer to Rails but still not `scoping`.

The observable behaviour it buys is `post.people.where(readers: { skimmer: true })
.create(...)` putting `skimmer` on the join row — covered by
`HasManyThroughAssociationsTest > through record is built when created with where`
(`packages/activerecord/src/associations/has-many-through-associations.test.ts:1220`).

## Converged shape

`AssociationRelation#create` / `#build` run the proxy call inside a real
`scoping` block, and `HasManyThroughAssociation#buildRecord` reads the through
scope as Rails does — `this._throughScope = this.scope()`, unconditionally, with
`scope()` seeing the current scope. The `_pendingThroughScope` field and the
`_collectionProxies` reach-back both disappear.

This depends on a current-scope mechanism that reaches `Association#scope`;
check what `Relation#scoping` / `Base.currentScope` already support before
starting, and block the story if the scope stack is not yet wired for it.

## Acceptance criteria

- [ ] `_pendingThroughScope` is deleted from `association-relation.ts` and
      `has-many-through-association.ts`.
- [ ] `HasManyThroughAssociation#buildRecord` sets `_throughScope` from
      `this.scope()` alone, matching has_many_through_association.rb:92.
- [ ] The scoped build/create tests above stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
