---
title: "Compute new-record dirtiness lazily and delete withoutMarkingRead"
status: done
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
pr: 6936
claim: "2026-08-23T18:08:08Z"
assignee: "tablename-only-model-resolves-columns-without-explicit-reflect"
blocked-by: null
closed-reason: null
---

## Context

Rails computes a record's dirtiness lazily: `ActiveModel::Attribute#changed?`
(`vendor/rails/activemodel/lib/active_model/attribute.rb:139-141`) runs only when
someone asks, and `Attribute#value` (`:100-103`) memoizes and flips
`@has_been_read` as a side effect of that ask.

trails computes it EAGERLY, in two places:

- `DirtyTracker#reinstateNewRecordChanges`
  (`packages/activemodel/src/dirty.ts`), run from `_reinstateConstructorDirtiness`
  (`packages/activerecord/src/base.ts:790-796`) for every new record — it walks
  every attribute and calls `attr.isChanged()` / `attr.value`.
- `Model#_writeAttribute` (`packages/activemodel/src/model.ts`), which needs the
  cast value at write time to feed `DirtyTracker#attributeWritten`.

Neither has a Rails counterpart, and both had to be made read-transparent when
`accessed_fields` converged onto `AttributeSet#accessed` (PR #6854): the
constructor pass now asks through the `@internal`
`Attribute#withoutMarkingRead`, and `_writeAttribute` casts through `type_cast`
rather than `fetchValue`. Both are receipts for the eager machinery, not for a
TypeScript shortcoming.

## Acceptance criteria

- New-record dirtiness is derived from the `Attribute` graph when asked, the way
  `ActiveModel::Dirty` does, rather than precomputed at construction.
- `Attribute#withoutMarkingRead` is deleted with the eager pass that needed it.
- `_writeAttribute` stops computing a cast value for the tracker (Rails reaches
  `type.changed?` from `Attribute#changed?`, `attribute.rb:155-160`).
- `accessed_fields` stays empty on a freshly constructed record
  (`packages/activerecord/src/attribute-methods.test.ts` `accessed_fields`, and
  the `marks the field accessed when read through %s` cases in
  `attribute-methods.trails.test.ts`).
