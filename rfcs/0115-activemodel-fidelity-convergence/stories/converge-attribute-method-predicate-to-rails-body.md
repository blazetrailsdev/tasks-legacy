---
title: "converge-attribute-method-predicate-to-rails-body"
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::AttributeMethods#attribute_method?`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:541-543`) is:

```ruby
def attribute_method?(attr_name)
  respond_to_without_attributes?(:attributes) && attributes.include?(attr_name)
end
```

trails' port, `packages/activemodel/src/attribute-methods.ts:231`, is:

```ts
export function isAttributeMethod(this: InstanceHost, attrName: string): boolean {
  return this._attributes?.has(attrName) ?? false;
}
```

It makes neither Rails call: no `respond_to_without_attributes?(:attributes)`
guard, and it consults the attribute SET rather than the `attributes` hash.

This was carried by a `call-mismatches-exclude` row
(`activemodel/attribute-methods.json`, `attribute_method?` →
`respond_to_without_attributes?`). PR #7000 landed `respond_to?`
(attribute_methods.rb:528-539) in the same file, which put
`isRespondToWithoutAttributes` into that file's call set and made the row go
stale; the only-shrink contract required deleting it, so the divergence is now
carried by nothing. This story is the actual convergence.

## Why it is not a mechanical swap

`attributes` is a getter over `_attributes.toHash()`, which COMPUTES every
attribute value, where `_attributes.has()` does not read any. Routing
`attribute_method?` through `attributes` would mark every attribute as read on
a path `matched_attribute_method` takes on every unmatched send — and
`accessed_fields` being empty on a freshly built record is pinned by
`attribute_methods_test.rb:1308`. Rails does not have this problem because
`ActiveRecord::AttributeMethods` overrides `attribute_method?`
(`activerecord/lib/active_record/attribute_methods.rb`) so the ActiveModel body
never runs for an AR record.

So the convergence likely needs the AR override ported first, or the ActiveModel
body converged with the accessed-fields consequence measured.

## Acceptance criteria

- `isAttributeMethod` (attribute-methods.ts) makes both calls Rails' body makes,
  in Rails' order, or the AR override is ported so the divergence is confined.
- `accessed_fields` behaviour stays as `attribute_methods_test.rb:1308` pins it.
- No new `call-mismatches-exclude` row; `pnpm parity:api:calls` and
  `pnpm parity:api:calls:args` clean.
