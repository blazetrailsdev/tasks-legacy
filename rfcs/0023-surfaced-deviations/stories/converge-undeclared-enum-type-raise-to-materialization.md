---
title: "Raise Undeclared attribute type from the enum decorate_attributes block, not the serialize path"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails raises "Undeclared attribute type for enum" from inside the
`decorate_attributes` block, at `_default_attributes` materialization, when the
resolved subtype is `ActiveModel::Type.default_value`
(`vendor/rails/activerecord/lib/active_record/enum.rb:239-246`).

trails defers it. `assertEnumTypeDeclared`
(`packages/activerecord/src/enum.ts:981-1007`) is driven by a
`_enumsPendingTypeCheck` set, reads `_columnsHash` through `ownProp` rather
than the subtype it was handed, and is gated on `isDecoratorReplay()` so it
skips `decorateAttributes`' eager pass. The raise therefore surfaces from
`Base.typeForAttribute` / `enumTypeOf` on the serialize path instead of from
materialization — the behavior enshrined in
`enum.trails.test.ts` "castEnumValue raises for an enum with no column and no
explicit type".

The deferral was introduced because the guard used to call
`columnForAttribute`, which drove a `loadSchema`, which replayed the pending
decorators, which re-entered the guard until the stack blew (the cycle is
documented at `enum.ts:966-978`). PR #6784 removed the post-reflection replay
inside `applyColumnsHash`, so that cycle no longer exists: reflection resets the
cache and `_defaultAttributes` replays lazily.

## Converged shape

The check lives in the enum `decorate_attributes` block and keys off the
subtype it is handed — `subtype == Type.default_value` — exactly as
`enum.rb:239-246`. No `_enumsPendingTypeCheck` sidecar set, no `_columnsHash`
probe, no `isDecoratorReplay()` gate.

## Acceptance criteria

- The raise fires from the decorator during `_default_attributes`
  materialization, on the `Type.default_value` subtype condition.
- `_enumsPendingTypeCheck` and `assertEnumTypeDeclared` are deleted.
- Confirm no `loadSchema` re-entry: the decorator must not reach
  `columnForAttribute`.
- The Rails-named enum tests keep their names; the trails test asserting the
  serialize-path raise site is updated to the materialization site (or dropped
  if the Rails test already covers it).
- Likely depends on [[retire-eager-decorate-attributes-bake]] for the
  `isDecoratorReplay()` half.
