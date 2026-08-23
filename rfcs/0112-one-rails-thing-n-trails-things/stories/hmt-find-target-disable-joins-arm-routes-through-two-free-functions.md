---
title: "HasManyThroughAssociation#find_target's disable_joins arm routes through two trails-only free functions instead of calling scope"
status: done
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6900
claim: "2026-08-23T00:57:31Z"
assignee: "wave-5g-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

Rails' `HasManyThroughAssociation#find_target`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:225-230`)
is four lines, and the `disable_joins` arm calls `scope` directly:

```ruby
return [] unless target_reflection_has_associated_record?
return scope.to_a if disable_joins
super
```

trails'
`packages/activerecord/src/associations/has-many-through-association.ts:83-92`
replaces `return scope.to_a if disable_joins` with a two-part indirection:
`_canRouteThroughViaDisableJoinsAssociationScope(reflection, options)` decides
the arm, and `_loadThroughViaDisableJoinsScope(owner, reflection, options)` —
a free function in `associations.ts` — builds and runs the relation. `scope()`
is therefore called one frame down, which is why the body carries a
`@missingRailsCall scope` receipt (migrated out of the RFC 0084 seed baseline by
the RFC 0106 wave-5e head sweep).

The routing predicate exists so this branch and the one in
`HasManyAssociation#findTarget` agree by construction. Rails needs no such
agreement: `disable_joins` is a plain reader on the association
(`association.rb:26`), and both bodies read it.

## Acceptance criteria

- [ ] `HasManyThroughAssociation#findTarget` spells Rails' arm as
      `if (this.disableJoins) return this.scope().toArray()` — the `scope()`
      call in this body, not one frame down.
- [ ] `_canRouteThroughViaDisableJoinsAssociationScope` /
      `_loadThroughViaDisableJoinsScope` are gone, or reduced to whatever
      `HasManyAssociation#findTarget` still genuinely needs.
- [ ] The `@missingRailsCall scope` receipt on `findTarget`
      (has-many-through-association.ts) is deleted, not reworded.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows; has_many
      :through and `disable_joins` suites green on SQLite, PostgreSQL and
      MySQL/MariaDB.
