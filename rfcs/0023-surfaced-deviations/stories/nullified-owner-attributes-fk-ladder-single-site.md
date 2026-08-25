---
title: "Converge the two nullifiedOwnerAttributes FK ladders onto ownerForeignKeyColumns"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR 5636 converged owner-FK _column_ derivation onto `ownerForeignKeyColumns`
(`packages/activerecord/src/associations/foreign-association.ts`), used by the
collection, has_one, and engine paths.

Two `nullifiedOwnerAttributes` helpers were left out of that convergence and
still carry their own three-rung ladder for both the FK and the polymorphic
type column:

- `packages/activerecord/src/associations/has-one-association.ts` —
  `nullifiedOwnerAttributes(assoc)`: rung 1 `ctor._reflectOnAssociation(...)
?.foreignKey`, rung 2 `assoc.foreignKeyColumns()`, rung 3
  `opts.foreignKey ?? (opts.as ? ${as}_id : ${ctorName}_id)`; the type column
  repeats `refl?.foreignType ?? (as ? ${as}_type : null)`.
- `packages/activerecord/src/associations/has-many-association.ts` (~line 462)
  — a byte-for-byte copy of the same ladder.

Rails has no such ladder:
`ActiveRecord::Associations::ForeignAssociation#nullified_owner_attributes`
(`vendor/rails/activerecord/lib/active_record/associations/foreign_association.rb`)
is four lines reading `reflection.foreign_key` and `reflection.type` directly.
These are exactly the copies that drift — the has_one FK copy PR 5636 deleted had
drifted for as long as it existed.

## Correction (2026-08-21, from PR #6815)

**The converged target named below — "resolve through `ownerForeignKeyColumns`"
— is the wrong one, and this story should not be closed by hitting it.**

PR #6815 converged `HasOneAssociation#nullifyOwnerAttributes` (a _different_
method, `has_one_association.rb:119-123`) and established that the rich
reflection's own `foreignKey` getter
(`packages/activerecord/src/reflection.ts:795-817`) already handles the
`options[:foreign_key]` rung and the `queryConstraints` rung. So
`ownerForeignKeyColumns`' extra options rung is redundant with it, and routing
through that helper substitutes one trails-invented indirection for another
rather than converging.

The Rails-faithful shape is to read `reflection.foreign_key` and
`reflection.type` **directly in the body**, exactly as
`foreign_association.rb:13-18` does:

```ruby
def nullified_owner_attributes
  Hash.new.tap do |attrs|
    Array(reflection.foreign_key).each { |foreign_key| attrs[foreign_key] = nil }
    attrs[reflection.type] = nil if reflection.type.present?
  end
end
```

In trails that is `_reflectOnAssociation(ownerClass, name)?.foreignKey` /
`?.foreignType` — the spelling PR #6815 shipped in `nullifyOwnerAttributes`.
Note also that reading the call under a private helper's name is what kept a
`foreign_key` call-mismatch row alive on the ratchet for that method; reading it
directly retired the row.

## Acceptance criteria

- Both `nullifiedOwnerAttributes` free functions read `reflection.foreignKey`
  and `reflection.foreignType` directly, per the Correction above; no
  per-call-site rungs remain, and no new helper is introduced in their place.
- `ForeignAssociation.nullifiedOwnerAttributes` keeps its current signature and
  Rails-faithful body.
- `dependent: :nullify` coverage still passes (has-one, has-many,
  has-many-through, polymorphic nullify cases).
- No behavior change intended; no test renames.
