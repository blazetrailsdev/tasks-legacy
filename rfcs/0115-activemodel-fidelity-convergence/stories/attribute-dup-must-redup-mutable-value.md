---
title: "attribute-dup-must-redup-mutable-value"
status: blocked
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-25T13:11:04Z"
assignee: "attribute-dup-must-redup-mutable-value"
blocked-by: "Blocked on open PR #7028 (converge-attribute-set-builder-residue): the story's whole surface, Attribute#dup() in packages/activemodel/src/attribute.ts, does not exist on main — #7028 introduces it. Implementing initialize_dup semantics here would either stack on that branch or re-add the method in attribute-set/builder.ts's pre-#7028 shape (dupAttribute + LazyAttributeHash.cloneAttr), conflicting with it file-wide. Re-ready once #7028 merges."
closed-reason: null
---

## Context

`Attribute#dup()` (`packages/activemodel/src/attribute.ts`, added by #7028 to
replace `attribute-set/builder.ts`'s `dupAttribute` and
`LazyAttributeHash.cloneAttr`) is a plain shallow copy:

```ts
return Object.assign(Object.create(Object.getPrototypeOf(this)), this);
```

Ruby's `Object#dup` runs `initialize_dup`
(`vendor/rails/activemodel/lib/active_model/attribute.rb:155-159`):

```ruby
def initialize_dup(other)
  if @value&.duplicable?
    @value = @value.dup
  end
end
```

`duplicable?` is true for `Array`, `Hash` and `Time` — the mutable cast values
an array-typed or datetime-typed column produces. In trails those are JS
`Array` / `Date` objects, so two Attributes off one `dup()` share the same
instance and an in-place mutation of one bleeds into the other. Rails separates
them.

Call sites that inherit the gap:

- `attribute-set/builder.ts` `LazyAttributeSet#defaultAttribute` and
  `LazyAttributeHash#assignDefaultValue` (`attr.dup()` for a schema default,
  `builder.rb:84`, `builder.rb:175`).
- `attribute-set/builder.ts` `LazyAttributeHash#deepDup` (`builder.rb:120`).
- `attribute-set.ts` `AttributeSet#cloneAttribute`, the private helper backing
  `deepDup` / `reverseMergeBang` — it shallow-copies the same way and should
  route through `Attribute#dup()` once this lands.

The gap predates #7028 (both spellings it merged had it); #7028 only named it.

## Acceptance criteria

- `Attribute#dup()` re-dups a duplicable `_value`, mirroring
  `initialize_dup` (attribute.rb:155-159). Decide and document what JS values
  stand in for Ruby's `duplicable?` — at minimum `Array` and `Date`, which the
  array and datetime types produce.
- `AttributeSet#cloneAttribute` (`attribute-set.ts`) uses `Attribute#dup()`
  rather than its own `Object.assign(Object.create(...))`, so there is one
  copy semantics in the package. `UserProvidedDefault`'s `dupForDeepClone`
  arm keeps whatever behaviour its own test pins.
- A regression test that fails on baseline: `deepDup` an AttributeSet holding
  an array-typed (or datetime-typed) attribute, mutate the copy's cast value
  in place, assert the original is unchanged.
- The `@noRailsEquivalent`-style caveat in `Attribute#dup()`'s JSDoc naming
  this story is removed.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
