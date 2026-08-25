---
title: "AttributeSet#deepDup invents cloneAttribute + identity cache instead of porting Attribute#deep_dup"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/attribute-set.ts:172` `deepDup` diverges from Rails
structurally, not just by identifier. Rails is two lines
(`vendor/rails/activemodel/lib/active_model/attribute_set.rb:73-75`):

```ruby
def deep_dup
  AttributeSet.new(attributes.transform_values(&:deep_dup))
end
```

trails instead builds a `newAttrs` map plus an identity `cache` and delegates to
an invented `protected cloneAttribute(attr, cache)` helper
(`attribute-set.ts:413-431`), which reflects over the prototype
(`Object.assign(Object.create(Object.getPrototypeOf(attr)), attr)`), probes for a
`dupForDeepClone` method, and walks `getOriginalAttribute()` /
`setOriginalAttribute()` — two more names with no Ruby counterpart. `deepDup`'s
Rails call, `Attribute#deep_dup`
(`vendor/rails/activemodel/lib/active_model/attribute.rb`), is not ported at all.

Surfaced by RFC 0096 wave-5 (`wave-5-naming-tail`) as a `naming` call-arg row —
ruby `ref:transformValues` vs ts `ref:newAttrs` — but it is an a3 (invented
helper) finding, not a rename, so per that RFC's `## Design` it was re-filed
rather than renamed away. The row stands in
`scripts/api-compare/output/call-arg-mismatches.json` until this lands.

## Acceptance criteria

- `Attribute#deepDup` is ported from `active_model/attribute.rb` and
  `AttributeSet#deepDup` is Rails' one-line `new AttributeSet(transform of
attributes)`.
- `cloneAttribute`, the identity `cache`, `dupForDeepClone` and the
  `getOriginalAttribute` / `setOriginalAttribute` pair are removed or traced to
  a Ruby counterpart.
- The `activemodel/attribute-set.ts deep_dup` row disappears from
  `pnpm parity:api:calls:args:report` with no baseline row added.
