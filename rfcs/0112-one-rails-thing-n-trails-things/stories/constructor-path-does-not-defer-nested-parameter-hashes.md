---
title: "Model.new does not defer nested parameter hashes"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 180
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Core#initialize` (`vendor/rails/activerecord/lib/active_record/core.rb`,
`initialize(attributes = nil)`) reaches assignment through
`assign_attributes(attributes)`, i.e. through
`_assign_attributes`
(`vendor/rails/activerecord/lib/active_record/attribute_assignment.rb:6-22`).
That means the nested-parameter-hash deferral (:13, :21) applies to `Model.new`
and `Model.create` exactly as it does to `assign_attributes`.

trails' constructor (`packages/activerecord/src/base.ts:3060`) does not call
`assignAttributes`. It runs its own bespoke split: `hasMultiparameterKeys` →
`extractMultiparameterCallstack` (:3099, :3115) plus
`_withoutDeferredConstructionKeys` (`base.ts:796`), which drops
nested-attribute setter keys and a composite-PK `id` before `super()` and
re-applies them later (`persistence.ts#_reapplyNestedAttrSetters`). There is no
`nested_parameter_attributes` bucket, so a Hash-valued key that is NOT a
registered `*Attributes` nested-attribute setter key — e.g. a `store` accessor
hash, or an association name given a hash — is still assigned in raw literal
order on the `new` path.

PR #6003 ported the deferral into `persistence.ts#assignAttributes` only; the
constructor path was explicitly out of its scope.

## Converged shape

The `new` path buckets Hash-valued keys and assigns them after the scalar pass,
before the multiparameter pass (:21-22), the same as `assignAttributes` — ideally
by routing the constructor through the one `_assign_attributes` implementation
rather than growing a fourth bespoke split.

## Acceptance criteria

- [ ] `new Model({ <hashKey>: {...}, <scalarKey>: v })` assigns `scalarKey`
      first regardless of literal order.
- [ ] Ordering holds when multiparameter keys are also present (nested before
      multiparameter, :21-22).
- [ ] Regression test that fails on baseline.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
