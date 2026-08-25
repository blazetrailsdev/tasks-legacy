---
title: "composed_of takes a class-name String and forwards its own options hash to Reflection.create"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# composed_of cannot forward its options hash — ComposedOfOptions holds a class, not a class-name String

## Context

Shipped as a reviewed call-**argument** baseline row by PR #6722
(`wave-4c-ar-core-residue-model`, RFC 0106). That PR converged
`composed_of`'s missing `Reflection.create` / `add_aggregate_reflection` calls,
but could not converge the argument list, so the deviation is now recorded as
the single `kind: "args"` row in
`scripts/api-compare/call-mismatches-exclude/activerecord/aggregations.json`.
It is debt, not a settled decision.

Rails, `activerecord/lib/active_record/aggregations.rb:250-266`:

```ruby
def composed_of(part_id, options = {})
  options.assert_valid_keys(:class_name, :mapping, :allow_nil, :constructor, :converter)
  ...
  class_name  = options[:class_name]  || name.camelize
  ...
  reflection = ActiveRecord::Reflection.create(:composed_of, part_id, nil, options, self)
  Reflection.add_aggregate_reflection self, part_id, reflection
end
```

Rails forwards its own `options` hash — the caller's, verbatim — and
`AggregateReflection` resolves `class_name` to a constant lazily.

trails, `packages/activerecord/src/aggregations.ts` (`ComposedOfOptions` and
`composedOf`): `className` is typed as the value-object CLASS
(`new (...args: any[]) => any`), not a Ruby class-name String, so `composedOf`
has to rebuild the reflection options at the call site:

```ts
const reflection = create(
  "composedOf",
  name,
  null,
  {
    className: options.className.name,
    mapping: options.mapping,
    anonymousClass: options.className,
  },
  modelClass,
);
```

Three separate divergences hide in that rebuild:

1. `class_name` is required in trails; Rails defaults it to `name.camelize`.
2. Rails' `:allow_nil` / `:constructor` / `:converter` never reach the
   reflection at all in trails — they are read straight off `options` by
   `readerMethod`/`writerMethod` instead, so
   `reflect_on_aggregation(name).options` answers a different hash than Rails'.
3. `assert_valid_keys` is not called, so a typo'd option key is silently
   ignored rather than raising `ArgumentError`.

This is a data-shape divergence in the public `composed_of` signature, NOT a
TypeScript language shortcoming — TS can hold a class-name string and resolve it
through the same registry `stiClassFor` / `computeType` already use. It is
therefore ratifiable only until someone converges it.

## Acceptance criteria

- [ ] `ComposedOfOptions` accepts Rails' option set with Rails' spelling and
      semantics: `className` as a String (defaulting to `name.camelize` when
      absent), `mapping`, `allowNil`, `constructor`, `converter`. Decide
      explicitly whether the class-valued form stays as an accepted extra arm —
      if it does, it is an additive overload, not the only shape.
- [ ] `composed_of` calls `assert_valid_keys` on the incoming options with
      Rails' key list and raises `ArgumentError` on an unknown key
      (aggregations.rb:251).
- [ ] `composed_of` forwards its own `options` to
      `Reflection.create("composedOf", partId, null, options, this)` — the whole
      hash, not a rebuilt one — so `reflectOnAggregation(name).options` answers
      what Rails answers.
- [ ] `AggregateReflection` resolves `className` to the value-object class at
      call time (the way Rails' `klass` does), rather than requiring the class
      up front.
- [ ] Delete the `composed_of` → `create` `kind: "args"` row from
      `scripts/api-compare/call-mismatches-exclude/activerecord/aggregations.json`
      by hand (`serializeBaseline`), then
      `pnpm parity:api:calls:tighten activerecord/aggregations.json`. No
      `--write`, no reseed.
- [ ] Existing `composed_of` call sites in
      `packages/activerecord/src/test-helpers/models/` are updated to the Rails
      spelling; test names match the Rails counterparts in
      `vendor/rails/activerecord/test/cases/aggregations_test.rb`.
- [ ] `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
      `pnpm parity:api:extra --package activerecord` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
