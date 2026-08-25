---
title: "Base's constructor splits store-accessor keys out of assign_attributes"
status: draft
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Base`'s constructor (`packages/activerecord/src/base.ts`, the consolidated
`constructor` — the `_storeKeys` / `_storeAttrs` block) pulls every
store-accessor key out of `attrs` before calling `super()`, then assigns them
afterwards through an inline prototype-descriptor walk, falling back to
`_writeAttribute`.

Rails' `Core#initialize` (`vendor/rails/activerecord/lib/active_record/core.rb:471-482`)
has no such split. It hands the whole hash to `assign_attributes`, and a
store accessor is reached by name like any other attribute, because
`store_accessor` generates ordinary `#{key}=` methods
(`vendor/rails/activerecord/lib/active_record/store.rb:96-120`). The
per-key dispatch is `_assign_attribute`'s `public_send("#{k}=", v)`
(`vendor/rails/activemodel/lib/active_model/attribute_assignment.rb:67-75`).

PR #7037 collapsed the constructor's other bespoke split — the
`hasMultiparameterKeys` branch — onto `_assign_attributes`. This block is the
remaining one. Its stated reason is dirty-state ordering: store attrs are
assigned after the clean re-snapshot so they read as changed on a new record.

## Converged shape

No split: `super(attrs)` takes the whole hash, and store accessors reach their
generated `#{key}=` setter through `_assign_attribute` like every other key. If
the dirty-baseline ordering is load-bearing, the fix belongs wherever the
snapshot is taken, not in a second assignment path in the constructor.

Related: [[store-accessor-assignment-still-needs-a-descriptor-walk]] covers the
`findPrototypeSetter` walk in `_assignAttribute`; this story is the constructor's
own inline copy of that walk plus the key split that feeds it.

## Acceptance criteria

- [ ] The `_storeKeys` / `_storeAttrs` split and its inline descriptor walk are
      gone from the constructor; `super()` receives the unsplit `attrs`.
- [ ] Store-accessor attrs still read as dirty on a new record (`store.test.ts`).
- [ ] `base.test.ts`, `base.trails.test.ts`, `store.test.ts`, `dirty.test.ts` green.
