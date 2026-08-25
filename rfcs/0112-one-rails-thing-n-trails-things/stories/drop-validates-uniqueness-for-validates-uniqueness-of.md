---
title: "drop-validates-uniqueness-for-validates-uniqueness-of"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 7060
claim: "2026-08-25T18:47:56Z"
assignee: "converge-association-check-klass-onto-reflection-check-validity"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `converge-model-validates-onto-rails-generic-lookup`
(PR #7039), which deleted ActiveRecord's `validates` override so `validates :x,
uniqueness: true` reaches ActiveModel's generic macro and resolves
`UniquenessValidator` off the model's ancestry.

Rails has exactly ONE registrar for uniqueness
(`vendor/rails/activerecord/lib/active_record/validations/uniqueness.rb:291-293`):

```ruby
def validates_uniqueness_of(*attr_names)
  validates_with UniquenessValidator, _merge_attributes(attr_names)
end
```

trails has two. Beside the faithful `validatesUniquenessOf`
(`packages/activerecord/src/validations/uniqueness.ts`, which does mirror the
body above), there is a second, novel `validatesUniqueness(attribute, options)`
in the same file — singular-attribute, non-variadic, bypassing
`_mergeAttributes`, and threading `class: this` explicitly:

```ts
export function validatesUniqueness(this, attribute: string, options = {}): void {
  validateScopeOption(options.scope);
  this.validatesWith(UniquenessValidator, { ...options, attributes: [attribute], class: this });
}
```

It has no Ruby counterpart and is scored `novel` by `parity:api:extra` on
`base.ts`. The explicit `class: this` is redundant: `validates_with` already
sets `options[:class] = self` (`activemodel/lib/active_model/validations/with.rb:88-90`,
ported at `packages/activemodel/src/validations/with.ts`), which is where Rails'
`UniquenessValidator#initialize` reads `@klass = options[:class]` from
(`uniqueness.rb:16`).

PR #7039 removed one caller (the deleted AR `validates` override looped it once
per attribute). What remains is ~10 non-test call sites plus a wider set of test
models — `test-helpers/models/reply.ts:49`, `test-helpers/models/uuid-item.ts:8`,
`encryption/test-helpers.ts:455`, and the uniqueness/array/uuid/case-sensitivity
suites — and a `dx-tests/edge-cases.test-d.ts:64-67` type assertion naming it.

Note the per-attribute registration difference this hides: `validatesUniqueness`
registers one validator instance per attribute, where Rails' single
`validates_with` call registers ONE validator covering the whole `attributes`
array. #7039's review confirmed the generic path already has Rails' shape.

## Converged shape

Delete `validatesUniqueness`. Call sites move to either `validatesUniquenessOf`
(the Rails registrar, variadic + trailing options) or the generic
`validates(attr, { uniqueness: {...} })`, which resolves the AR
`UniquenessValidator` through the ancestry constants #7039 put on `Base`. Drop
the `declare static validatesUniqueness` on `base.ts:3132`, its re-exports in
`validations.ts:25,42,295`, and the `dx-tests` assertion.

Keep `validateScopeOption`'s eager `ArgumentError` only if
`UniquenessValidator`'s constructor does not already raise it — it does today
(same file), so the declaration-time call is likely droppable too.

## Acceptance criteria

- [ ] `validatesUniqueness` is deleted; no `@noRailsEquivalent` tag is added in
      its place.
- [ ] Every call site uses `validatesUniquenessOf` or `validates(..., {
uniqueness })`, matching `uniqueness.rb:291-293`.
- [ ] `pnpm parity:api:extra --package activerecord` loses the `validatesUniqueness`
      novel row on `base.ts`.
- [ ] AR uniqueness/encryption/adapter suites green on all three adapter lanes
      (uniqueness is adapter-sensitive: case-sensitivity, arrays, uuid).
