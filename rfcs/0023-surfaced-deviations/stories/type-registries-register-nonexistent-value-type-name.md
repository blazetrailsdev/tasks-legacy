---
title: "Both type registries register a :value name Rails has no registration for"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Neither Rails registry registers a `:value` type. `ActiveModel::Type`
(`vendor/rails/activemodel/lib/active_model/type.rb:43-53`) registers
big_integer, binary, boolean, date, datetime, decimal, float, immutable_string,
integer, string, time; `ActiveRecord::Type`
(`vendor/rails/activerecord/lib/active_record/type.rb:69-81`) registers the AR
set. `Type.default_value` returns `Type::Value.new` directly
(`activemodel/lib/active_model/type.rb`), it is not a registry entry, so
`attribute :foo, :value` raises `Unknown type :value` in Rails.

trails diverges: ActiveModel's `TypeRegistry` constructor registers `"value"`
(`packages/activemodel/src/type/registry.ts:36`) — a trails-only entry
predating this work — and PR #5196 mirrored it into ActiveRecord's registry
(`packages/activerecord/src/type.ts:80`,
`_registry.register("value", ValueType, { override: false })`) so that AR models
declaring `attribute(name, "value")` did not regress when `resolveTypeName`
switched them off the ActiveModel registry. The mirror was correct for that PR
(no name resolvable before may become "Unknown type") but propagates the
original deviation into a second registry.

## Acceptance criteria

- Establish whether any trails caller actually declares `attribute(x, "value")`
  by name; `grep -rn '"value"' packages/*/src --include=*.ts` over the
  `attribute(` and `lookup(` call sites.
- If nothing depends on it, drop the `"value"` registration from BOTH registries
  (activemodel `type/registry.ts` and activerecord `type.ts`) so the name raises
  as it does in Rails, and route internal `typeRegistry.lookup("value")` callers
  (`attribute-set.ts:147,171,260,308`, `attribute-registration.ts:233`,
  `attributes.ts`, `model-schema.ts:1161`, `attribute.ts:324`) to a
  `Type.defaultValue()` helper mirroring `ActiveModel::Type.default_value`.
- If something does depend on the name, keep it and justify at both call sites.
- Both registries stay in agreement either way — the two must not diverge again.
