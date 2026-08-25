---
title: "arel-duplicates-activemodel-type-default-value"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "arel"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Noticed while converging `HomogeneousIn#procForBinds` in RFC 0096 wave 2
(`naming-burndown-2-arel-activemodel`, PR #6421).

`vendor/rails/activerecord/lib/arel/nodes/homogeneous_in.rb:49-51`:

```ruby
def proc_for_binds
  -> value { ActiveModel::Attribute.with_cast_value(attribute.name, value, ActiveModel::Type.default_value) }
end
```

`ActiveModel::Type.default_value` is a single memoized `Value.new` for the whole
process (`vendor/rails/activemodel/lib/active_model/type.rb`).

trails has that function — `defaultValue()` in
`packages/activemodel/src/type.ts:20-22`, memoizing into a module-level
`_defaultValue`. But `packages/activemodel/src/type.ts` is not re-exported from
`packages/activemodel/src/index.ts` at all, so `arel` cannot reach it, and
`packages/arel/src/nodes/homogeneous-in.ts:7-10` carries a byte-for-byte copy:

```ts
let _defaultValue: ValueType | null = null;
function defaultValue(): ValueType {
  return (_defaultValue ??= new ValueType());
}
```

The result is **two distinct `ValueType` singletons** where Rails has one. Any
code that compares types by identity (rather than by `==`/`type` name) sees the
arel-minted default as a different object from the activemodel one.

The naming row itself is converged (PR #6421 renamed arel's copy from
`defaultType` to `defaultValue`); this story is about the duplication.

## Acceptance criteria

- [ ] `packages/activemodel/src/type.ts` is reachable from the package's public
      surface under a shape that reads as Rails' `ActiveModel::Type` module —
      check `docs/ruby-ts-conventions.md` and `pnpm parity:api --package
activemodel` for whether `type.rb`'s module functions (`registry`,
      `register`, `lookup`, `default_value`) are currently counted as missing.
- [ ] `packages/arel/src/nodes/homogeneous-in.ts` imports `defaultValue` instead
      of redefining it; the local `_defaultValue` memo is deleted.
- [ ] Verify no import cycle / TDZ is introduced — arel already imports
      `Attribute` and `ValueType` from `@blazetrails/activemodel`, but confirm
      against the built `dist/**.js` with a plain-node import as entry module
      (CLAUDE.md "Call-time constant resolution"), not just a green vitest run.
- [ ] `pnpm vitest run packages/arel/src` and `packages/activemodel/src` pass.
