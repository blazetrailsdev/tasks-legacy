---
title: "Drop the isInitialized filter from values_before_type_cast / values_for_database"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 7029
claim: "2026-08-25T12:46:55Z"
assignee: "converge-reverse-merge-bang-key-presence"
blocked-by: null
closed-reason: null
---

## Context

`values_before_type_cast` and `values_for_database`
(`vendor/rails/activemodel/lib/active_model/attribute_set.rb:28-34`) are plain
one-line `transform_values` over EVERY entry in the backing hash:

```ruby
def values_before_type_cast
  attributes.transform_values(&:value_before_type_cast)
end

def values_for_database
  attributes.transform_values(&:value_for_database)
end
```

`packages/activemodel/src/attribute-set.ts` writes both as a `for-of` that
**skips uninitialized attributes**:

```ts
for (const [name, attr] of this.attributes()) {
  if (attr.isInitialized()) result[name] = attr.valueBeforeTypeCast;
}
```

So where Rails returns a key for every attribute (an `Uninitialized` one
yielding `nil` — `attribute.rb:242-247`), trails omits the key entirely. A
caller distinguishing "present and nil" from "absent" sees a different answer,
and `attributes_before_type_cast` is exactly such a reader.

These are the last two rows in
`scripts/api-compare/call-mismatches-exclude/activemodel/attribute-set.json`.
RFC 0115's `retire-attribute-set-map-adapter-surface` (PR #7021) took that
shard from 7 rows to 2 and deliberately left these, because the fix is a
behaviour change in `attributesBeforeTypeCast` / `attributesForDatabase`
consumers rather than a call-shape rename.

## Acceptance criteria

- Both readers are `transformValues(this.attributes(), …)` over every entry,
  with no `isInitialized()` filter — the shape `attribute_set.rb:28-34` has.
- Consumers that relied on the omission (start from
  `attributesBeforeTypeCast` in `packages/activerecord/src/attribute-methods/before-type-cast.ts`
  and `attributesForDatabase`) are checked against the Rails behaviour and
  adjusted, not worked around at the AttributeSet level.
- The two remaining rows in
  `call-mismatches-exclude/activemodel/attribute-set.json` are hand-deleted and
  the shard's mark tightened with
  `pnpm parity:api:calls:tighten activemodel/attribute-set.json` — leaving the
  shard empty.
- Parity deltas non-negative for activemodel and activerecord.

## Verification

```bash
pnpm vitest run packages/activerecord/src/attribute-methods packages/activemodel/src/attribute-set.test.ts packages/activerecord/src/persistence.test.ts packages/activerecord/src/dirty.test.ts
```
