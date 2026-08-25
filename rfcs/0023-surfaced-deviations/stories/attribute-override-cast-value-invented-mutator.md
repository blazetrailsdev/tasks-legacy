---
title: "Replace Attribute#overrideCastValue with Rails' value-returning with_cast_value"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`Attribute#overrideCastValue` (`packages/activemodel/src/attribute.ts:230`) has
**no Rails counterpart**. Flagged twice by the reviewer on PR #6813 while it was
touched incidentally (that PR only removed a stale `nullifyBlanks` mention from
its doc comment).

Rails' `ActiveModel::Attribute` is immutable in this respect: the sibling
operations are `with_value_from_user(value)`
(`vendor/rails/activemodel/lib/active_model/attribute.rb:78`),
`with_cast_value(value)` (`:87`) and `with_type(type)` (`:91`) — each returns a
**new** Attribute. There is no in-place cast-value mutator anywhere in
`attribute.rb`.

trails' version mutates the receiver, reaching in to reset four private fields:

```ts
overrideCastValue(value: unknown): void {
  this._value = value;
  this._hasValue = true;
  this._cachedValueForDatabase = undefined;
  this._hasValueForDatabase = false;
}
```

The single production caller is
`packages/activerecord/src/relation/query-attribute.ts:62`. (The other former
caller was `Model#_writeAttribute`'s `nullifyBlanks` hook, itself invented
surface, deleted in #6813 — so the remaining call site is now the only thing
holding this method alive.)

## Converged shape

Replace the mutator with Rails' `withCastValue`, returning a new Attribute
(`attribute.rb:87`), and have `query-attribute.ts:62` bind the returned
attribute rather than mutating in place. Confirm whether
`Attribute#with_cast_value`'s `value_before_type_cast` / type-carrying semantics
(`attribute.rb:16` `with_cast_value(name, value_before_type_cast, type)`) are
what that call site actually wants — if it wants the _user_ arm, the Rails
spelling is `with_value_from_user` (`:78`).

Note `AttributeSet#writeCastValue` (`packages/activemodel/src/attribute-set.ts:172`)
IS a real Rails method (`attribute_set.rb:64`) and is not in scope; only the
per-Attribute mutator is.

## Acceptance criteria

- `overrideCastValue` is gone from `packages/activemodel/src/attribute.ts`.
- The `query-attribute.ts` call site uses the Rails-named, value-returning form.
- `pnpm parity:api:extra --package activemodel` loses the `overrideCastValue`
  novel row.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
