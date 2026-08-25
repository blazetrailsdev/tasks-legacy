---
title: "Retire PredicateBuilder#normalizeQueryValue — Rails has no normalization step in build"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

`PredicateBuilder#build` in Rails is five lines
(`vendor/rails/activerecord/lib/active_record/relation/predicate_builder.rb:57-65`):

```ruby
def build(attribute, value, operator = nil)
  value = value.id if value.respond_to?(:id)
  if operator ||= table.type(attribute.name).force_equality?(value) && :eq
    bind = build_bind_attribute(attribute.name, value)
    attribute.public_send(operator, bind)
  else
    handler_for(value).call(attribute, value)
  end
end
```

There is no normalization step, because there is nothing to special-case: the
attribute's cast type IS the `NormalizedValueType` decorator
(`normalization.rb:89-93`), so `table.type(column_name)` at
`predicate_builder.rb:68` already normalizes on the query path.

trails carries a bespoke pre-pass instead —
`packages/activerecord/src/relation/predicate-builder.ts:268` calls the private
`normalizeQueryValue` (`:317`), which gates on
`klass.normalizedAttributes.has(columnName)` and re-casts the value early, only
to decide nil-routing (`where(col: "")` must reach `IS NULL` when a normalizer
maps the scalar to nil). Neither the method nor the gate exists in Rails.

Introduced by `normalizes-query-and-in-place-type-decoration` (RFC 0030); the
`normalizedAttributes` read was rewired from the retired `_normalizations` map
in #6834 without touching the shape.

## Acceptance criteria

- `normalizeQueryValue` is gone from `predicate-builder.ts`, along with the
  `normalizedAttributes` gate — `PredicateBuilder` learns nothing about
  normalization, exactly as `predicate_builder.rb` does not.
- Whatever nil-routing `where(col: "")` needs comes from the decorated type on
  the path Rails uses (`table.type(...)` / `buildBindAttribute`,
  predicate_builder.rb:67-69), not from a caller-side pre-cast.
- `pnpm vitest run packages/activerecord/src/normalized-attribute.test.ts` stays
  green — `normalizes value in query` and the `where`/`exists?` cases in
  `vendor/rails/activerecord/test/cases/normalized_attribute_test.rb` are the
  behavioural contract.
- `pnpm parity:api:extra --package activerecord` loses the `normalizeQueryValue`
  row; `parity:api:calls` / `:args` clean, no reseed.
