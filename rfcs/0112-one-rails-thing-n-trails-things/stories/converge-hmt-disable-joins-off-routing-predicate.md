---
title: "Converge the HMT disable_joins arm off the DJAS routing predicate onto scope.to_a"
status: done
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 250
pr: 6900
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6113 (`hoist-disable-joins-branch-into-hmt-find-target`), which
put Rails' `disable_joins` arm back into
`HasManyThroughAssociation#findTarget` in Rails' statement order
(`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:225-230`):

```ruby
def find_target(async: false)
  raise NotImplementedError, "..." if async
  return [] unless target_reflection_has_associated_record?
  return scope.to_a if disable_joins
  super
end
```

The branch could not be ported literally as `if (this.disableJoins) return
this.scope().toArray()`. trails gates the disable-joins path on
`_canRouteThroughViaDisableJoinsAssociationScope` (`associations.ts:902`), a
routing predicate Rails does not have: it demands a through reflection, a
source reflection, and a polymorphic-source/`sourceType` pairing, and any
shape it rejects falls back to the 2-step `loadHasManyThrough` loader instead
of the scope chain. A bare `this.disableJoins` read would therefore change
which loader the un-gated shapes get, so #6113 routed the hoisted branch
through the same predicate the loader consults — correct, but still a branch
Rails does not write.

Rails needs no gate because `DisableJoinsAssociationScope#scope`
(`disable_joins_association_scope.rb:6-15`) handles every shape uniformly:
`reflection.chain` flattens nested-through into a straight reflection list and
the `get_chain` / `last_scope_chain` reverse walk iterates it, so there is no
shape for which DJAS "doesn't apply". The predicate's own JSDoc admits this
("Rails' DJAS has no routing gate at all and handles each shape via the
generic chain walk").

This is the plural sibling of
[[converge-singular-through-two-step-load]], which tracks the same predicate on
the `has_one :through` side (`singular-association.ts`). Both call sites die
with the gate.

## Converged shape

Widen trails' DJAS until it covers every through shape the gate currently
rejects, then delete `_canRouteThroughViaDisableJoinsAssociationScope` and
spell the branch the way Rails does:

```ts
if (this.disableJoins) return this.scope().toArray();
```

`Association#scope()` already has its own `disable_joins` branch delegating to
DJAS (`association.rb:302`, `association.ts:318`), so `scope().toArray()`
becomes the whole implementation and `_loadThroughViaDisableJoinsScope`
disappears with it.

## Acceptance criteria

- [ ] `HasManyThroughAssociation#findTarget`'s disable-joins arm reads
      `if (this.disableJoins) return this.scope().toArray();` with no routing
      predicate.
- [ ] `_canRouteThroughViaDisableJoinsAssociationScope` is deleted, or reduced
      to the singular call site if that side lands separately.
- [ ] Every disable-joins through suite passes unchanged, no test renames:
      `disable-joins-*.test.ts`, `has-many-through-disable-joins-associations.test.ts`,
      `cp-count-disable-joins-through.test.ts`.
- [ ] A shape the old gate rejected (e.g. polymorphic source without
      `sourceType`) is covered by a test proving it now routes through DJAS.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
