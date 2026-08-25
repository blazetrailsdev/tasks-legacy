---
title: "pluckCastTypeForKnownColumn consults the _serializedAttributes registry instead of attribute_types alone"
status: draft
updated: 2026-08-20
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

Surfaced while converging `pluckCastTypeForKnownColumn` off `_attributeDefinitions`
in PR #6769 (`converge-attribute-definitions-onto-default-attributes`, leaf-reader
tranche). That PR changed the presence check to `Object.hasOwn(model.attributeTypes(), name)`,
matching Rails' `attribute_types.fetch(name) { ... }` key semantics — but it left a
third divergence in the same method untouched, and it is not covered by
[[type-cast-pluck-values-maps-result-columns-and-drops-type-caster]] (which tracks the
`result.columns` mapping and the dropped `column.try(:type_caster)` alternative).

Rails (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:617`):

```ruby
model.attribute_types.fetch(name = result.columns[i]) do
  ...
end
```

`attribute_types` is derived from `_default_attributes`
(`vendor/rails/activemodel/lib/active_model/attribute_registration.rb`), so a
`serialize`d attribute is ALREADY an `ActiveModel::Type::Serialized` in that hash —
Rails consults exactly one source and gets the coder-aware cast type for free.

trails (`packages/activerecord/src/relation/calculations.ts`,
`pluckCastTypeForKnownColumn`) consults a second, parallel source first:

```ts
const coder = model._serializedAttributes?.get(name);
if (coder) return { deserialize: (value) => coder.load(value) };
return model.typeForAttribute?.(name) ?? null;
```

`_serializedAttributes` is a trails-side registry with no Rails counterpart, and the
ad-hoc `{ deserialize }` object it builds is not a `Type` — it bypasses whatever
decoration `attribute_types` carries for that attribute (normalizes, encrypts,
enum), so a serialized attribute that is ALSO decorated plucks through the raw
coder only.

`serialize.ts` already pushes a durable `decorateAttributes` decorator that installs
the `Serialized` cast type into the replayed attribute set, so the registry read is
redundant on top of being divergent.

## Converged shape

Drop the `_serializedAttributes` arm; return `model.typeForAttribute(name)` as the
single source, exactly as Rails' single `attribute_types` lookup does. Verify the
`Serialized` type is present in `attributeTypes()` for a `serialize`d attribute at
pluck time (the `serialize.ts` decorator replays on every `_defaultAttributes`
rebuild, so it should be); if it is not, the bug is in the decorator's replay, not
here, and fixing that is the convergence.

## Acceptance criteria

- [ ] `pluckCastTypeForKnownColumn` reads only `attribute_types` / `type_for_attribute`.
- [ ] Plucking a `serialize`d column still round-trips through its coder, covered by
      a test on canonical models/schema.
- [ ] A test plucks an attribute that is both `serialize`d and decorated and asserts
      the decoration is honored (fails on the current registry-first arm).
- [ ] If `_serializedAttributes` has no other readers left, it is deleted.
- [ ] `pnpm parity:api:calls` / `:args` green; SQLite, PostgreSQL and MySQL lanes green.
