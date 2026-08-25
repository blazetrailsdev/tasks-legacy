---
title: "update_columns raises through the attribute set, not an invented pre-check"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`updateColumns` in `packages/activerecord/src/persistence.ts` runs an invented
pre-check before writing any value:

```ts
const known = Object.hasOwn(attributeTypes, key);
if (!known && !pkCols.includes(key)) throw new UnknownAttributeError(this, key);
```

plus a primary-key allowance ("PK columns are implicit on Base and aren't always
in `attribute_types`").

Rails has neither. `update_columns`
(`activerecord/lib/active_record/persistence.rb:604-627`) transforms the keys
for aliases and readonly, then writes each straight through
`@attributes.write_cast_value(k, v)` (`:617`). An unknown key surfaces from
`AttributeSet#write_cast_value` → `self[name]`
(`activemodel/lib/active_model/attribute_set.rb:64-66, 16-18`), where a missing
key returns `default_attribute(name)` — `Attribute.null` on a plain AttributeSet
— and `Attribute::Null#with_cast_value` raises `ActiveModel::MissingAttributeError`.
So the error class, message and raise site all differ from trails', and the PK
allowance means trails silently accepts a key Rails would reject (or vice versa,
when the PK genuinely is in the set).

## Converged shape

Drop the pre-check and the `pkCols` allowance; write through the attribute set
and let `Attribute::Null` raise, matching Rails' class and message. That
requires trails' `AttributeSet` write path to raise the same way — check
`writeCastValue` / `getAttribute` in
`packages/activemodel/src/attribute-set.ts` before deleting the guard, and fix
the PK gap at its real cause (the PK not being in `attribute_types` for some
classes) rather than by exempting it here.

## Acceptance criteria

- [ ] `updateColumns` has no unknown-key pre-check and no primary-key exemption.
- [ ] An unknown key raises the Rails error class with the Rails message, from
      the attribute-set write.
- [ ] `persistence` suites pass, including the existing unknown-attribute cases
      (retarget them at the Rails error if they assert `UnknownAttributeError`).
