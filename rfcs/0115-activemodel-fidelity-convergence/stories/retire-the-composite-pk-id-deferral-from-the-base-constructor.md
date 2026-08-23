---
title: "Retire the composite-PK id deferral from the Base constructor"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: 60
pr: 6933
claim: "2026-08-23T18:20:09Z"
assignee: "sweep-trails-only-test-files-associations"
blocked-by: null
closed-reason: null
---

# Retire the composite-PK `id` deferral from the `Base` constructor

## Context

Surfaced landing PR #6819 (`retire-reapply-nested-attr-setters-onto-constructor-assign`).
That story deleted the nested-attribute half of the constructor's deferred-key
machinery, leaving `_withoutDeferredConstructionKeys`
(`packages/activerecord/src/base.ts:755-775`) and `_applyCompositePrimaryKey`
(`:735-752`) holding exactly ONE key out of `super()`'s assignment loop and
re-dispatching it afterwards: a composite-PK `id`.

Rails has no such pass. `Core#initialize`
(`vendor/rails/activerecord/lib/active_record/core.rb:390-400`) routes the whole
hash through `assign_attributes` (:394), so `id: [a, b]` is dispatched by
`_assign_attribute`'s `public_send("id=", v)`
(`vendor/rails/activemodel/lib/active_model/attribute_assignment.rb:68`) inline
with every other key, reaching `CompositePrimaryKey#id=`
(`vendor/rails/activerecord/lib/active_record/core.rb:766-768` /
`primary_key.rb`). One pass, no bucketing, no re-dispatch.

The stated reason for the deferral — "`id=` touches key columns that aren't
wired until after `super()`" — is the same class of ordering problem that PR
PR #6819 fixed for `new_record?`: a JS subclass field initializer runs after
`super()`, so state declared as a `Base` class field is absent while ActiveModel
assigns. #6819 raised `_newRecord` at the head of `Core#initInternals`
(`packages/activerecord/src/core.ts`), which is Rails' own order
(`@new_record = true` is `initialize`'s first line, two lines above
`init_internals`). Whatever `id=` actually needs may be seatable the same way,
which would let the `id` bucket go and take both helpers with it.

## Converged shape

`_withoutDeferredConstructionKeys` and `_applyCompositePrimaryKey` are deleted;
a composite-PK `id` dispatches through `_assignAttributes` on the single
constructor pass like every other key, with whatever state `id=` reads seated
before ActiveModel assigns (the `initInternals` seat, as `_newRecord` now is)
rather than after `super()` returns.

First step is to establish what `id=` actually reads that is unavailable during
`super()` — `_applyCompositePrimaryKey` names only "the key columns" — and
whether it is a `Base` class field (same fix as `_newRecord`) or genuinely
constructed later.

## Acceptance criteria

- [ ] `base.ts` no longer defines `_withoutDeferredConstructionKeys` or
      `_applyCompositePrimaryKey`; the constructor passes the whole hash to
      `super()`.
- [ ] `new Model({ id: [a, b] })` on a model-level composite PK still zips
      across the key columns, and a scalar `id` still raises `TypeError` as
      `CompositePrimaryKey#id=` does.
- [ ] `composite-primary-key.test.ts` and `base.test.ts` stay green on SQLite,
      PostgreSQL and MySQL/MariaDB.
- [ ] `pnpm parity:api:extra --package activerecord` loses two novel names.
