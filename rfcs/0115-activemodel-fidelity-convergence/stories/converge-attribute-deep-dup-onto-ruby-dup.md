---
title: "Converge AttributeSet#deep_dup and reverse_merge! onto Ruby's dup semantics"
status: claimed
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: "2026-08-25T12:22:53Z"
assignee: "converge-attribute-deep-dup-onto-ruby-dup"
blocked-by: null
closed-reason: null
---

## Context

`AttributeSet#deep_dup` is one line in Rails
(`vendor/rails/activemodel/lib/active_model/attribute_set.rb:72-74`):

```ruby
def deep_dup
  AttributeSet.new(attributes.transform_values(&:deep_dup))
end
```

`Attribute` defines no `deep_dup`, so `&:deep_dup` is ActiveSupport's
`Object#deep_dup` — `duplicable? ? dup : self` — which reaches
`Attribute#initialize_dup` (`vendor/rails/activemodel/lib/active_model/attribute.rb:155-157`),
and that dups **only `@value`**. `@original_attribute` is carried into the copy
**by reference**.

`packages/activemodel/src/attribute-set.ts` instead routes `deepDup` through a
private `cloneAttribute(attr, cache)` that walks and rebuilds the whole
`originalAttribute` chain, memoised through a `Map<Attribute, Attribute>`. Two
divergences fall out:

- The original-attribute chain is deep-copied where Rails shares it, so a
  dup'd set's `original_value` / `changed?` read from a different object graph
  than Rails'.
- `reverseMergeBang` (`attribute_set.rb:96-98`) also clones through
  `cloneAttribute`, where Rails' `attributes.reverse_merge!(target.attributes)`
  copies references and clones nothing at all.

RFC 0115's `retire-attribute-set-map-adapter-surface` (PR #7021) took the file
to 0 novel / 0 moved on `parity:api:extra` and left `cloneAttribute` in place —
it is `private`, so unscored, but it is the last member of the file with no
counterpart in `attribute_set.rb`.

Related but distinct file: `converge-attribute-set-builder-residue` covers
`dupAttribute` / `cloneAttr` in `attribute-set/builder.ts`, the same
`Attribute#dup` concept spelled twice more. Converging both onto one real
`Attribute#deepDup` would retire all three.

## Acceptance criteria

- `Attribute` gets a `deepDup()` mirroring `Object#deep_dup` +
  `Attribute#initialize_dup` (attribute.rb:155-157): dup the attribute, dup
  `@value` if duplicable, share `@original_attribute` by reference.
  `UserProvidedDefault`'s existing `dupForDeepClone` fold in as its
  `initialize_dup` override if Rails has one there; otherwise it is deleted.
- `AttributeSet#deepDup` is `transformValues(this.attributes(), (attr) => attr.deepDup())`
  with no cache and no chain walk.
- `reverseMergeBang` copies references, matching `reverse_merge!`.
- `cloneAttribute` is deleted from `attribute-set.ts`.
- Dirty-tracking behaviour is unchanged for the cases the suites cover, or the
  changed case is the one Rails actually produces — verify against the Ruby
  before adjusting any assertion.

## Verification

```bash
pnpm vitest run packages/activemodel/src/attribute-set.test.ts packages/activemodel/src/attribute.test.ts packages/activemodel/src/dirty.test.ts packages/activerecord/src/dirty.test.ts packages/activerecord/src/dup.test.ts packages/activerecord/src/clone.test.ts
```
