---
title: "Wire Locking::Optimistic#_touch_row/_update_row into the touch/update persistence path"
status: ready
updated: 2026-06-26
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps:
  - persistence-test-canonical-wave15
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

Follow-up surfaced by PR #3949 (thread-attribute-names-through-create-record),
which wired `LockingOptimistic._createRecord` into the create path
(`base.ts._performInsert`). That story's acceptance listed `_touchRow`/`_updateRow`
as "ideally" — deferred as out of create-path scope.

The locking mirrors `_touchRow` (optimistic.ts:197-208, Mirrors
`Locking::Optimistic#_touch_row`) and `_updateRow` (optimistic.ts:214-243,
Mirrors `Locking::Optimistic#_update_row`) remain **dead code** — not present in
`LockingOptimistic.InstanceMethods` (optimistic.ts:316-320) and not called from
the touch/update persistence paths. As with the pre-#3949 create path, the
touch/update paths (`base.ts._performUpdate`, `callbacks.ts._updateRecord`,
the `_touchRow`/`updateRow` equivalents) likely apply locking logic inline
rather than threading through the optimistic.rb mirrors.

Rails: `Locking::Optimistic#_touch_row` unions the locking column into
`touch_attribute_names` before `super`; `#_update_row` builds the
`WHERE locking_column = ?` constraint, bumps the in-memory version, and raises
`StaleObjectError` on `affected_rows != 1` (optimistic.rb:62-99).

## Acceptance criteria

- [ ] `_touchRow`/`_updateRow` mirrors are wired into the touch/update
      persistence path (analogous to how #3949 wired `_createRecord`), carrying
      the locking-column union / stale-object check from the locking layer.
- [ ] Any inline locking logic in the generic touch/update path is removed in
      favor of the optimistic.ts mirrors, matching Rails' layering.
- [ ] `locking.test.ts` + `persistence.test.ts` + `touch-later.test.ts` stay
      green; no parity:api / parity:test regression.
