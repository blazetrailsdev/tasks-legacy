---
title: "Consolidate the duplicated ThroughAssociation module into one mixin"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 300
pr: 6757
claim: "2026-08-20T01:56:44Z"
assignee: "consolidate-duplicated-through-association-module"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the RFC 0099 explicit-host argument rows in PR #6359
(`call-args-ar-host-param-associations`).

Rails has ONE `ThroughAssociation` module —
`vendor/rails/activerecord/lib/active_record/associations/through_association.rb`
— `include`d into both `HasManyThroughAssociation`
(`has_many_through_association.rb:8`) and `HasOneThroughAssociation`
(`has_one_through_association.rb:7`). It defines `transaction` (:10-12),
`through_reflection` (:14-24), `through_association` (:26-28),
`construct_join_attributes` (:56-84), `ensure_mutable` (:86-92) and
`ensure_not_nested` (:94-98), among others.

trails duplicates that module body across the two subclass files. After #6359
each copy is a `this`-typed function installed on its own class's prototype, so
the argument lists are right, but the implementations are still two independent
copies:

- `packages/activerecord/src/associations/has-many-through-association.ts` —
  `throughReflection`, `throughAssociation`, `constructJoinAttributes`,
  `ensureMutable`, `ensureNotNested`
- `packages/activerecord/src/associations/has-one-through-association.ts` —
  the same five, plus `transaction`

`packages/activerecord/src/associations/through-association.ts` already exists
and mirrors part of the same Rails file (`sourceReflection`, `staleStateImpl`,
`throughTargetScope`), so the module has a home — the duplicated members simply
never moved into it.

Two copies of one Rails method is the failure mode this deviation invites: they
have already drifted (`has-many-through`'s `constructJoinAttributes` is
variadic and carries a `sourceType` foreign-type arm; `has-one-through`'s is
not), so a fix applied to one silently misses the other.

## Converged shape

One `ThroughAssociation` module object in `through-association.ts` holding the
Rails members at the Rails names, `include()`d (or `Object.assign`ed onto the
prototype, as #6359 does) into both `HasManyThroughAssociation` and
`HasOneThroughAssociation`. Where the two current copies differ, the union that
matches `through_association.rb` wins — the differences are drift, not
intentional specialisation, and each must be checked against the Ruby before it
is dropped or kept.

## Acceptance criteria

1. `transaction`, `through_reflection`, `through_association`,
   `construct_join_attributes`, `ensure_mutable` and `ensure_not_nested` exist
   ONCE, in `through-association.ts`, at the Rails names, mixed into both
   through-association classes.
2. Every behavioural difference between the two retired copies is resolved
   against `through_association.rb` and cited at the call site.
3. No public surface change: `pnpm parity:api:extra --package activerecord`
   does not grow; `pnpm parity:api` delta is non-negative.
4. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
5. `pnpm vitest run packages/activerecord/src/associations/` passes.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
