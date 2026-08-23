---
title: "Retire activerecord/callbacks.ts's duplicate beforeValidation/afterValidation free functions"
status: draft
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
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

`packages/activerecord/src/callbacks.ts` still carries `beforeValidation` `:57`
and `afterValidation` `:71` as free functions taking the model class as their
first argument, plus the `registerCallback` `:124` helper and the
`ValidationCallbackOptions` / `CallbackOptions` / `CallbackFilter` types that
exist only to serve them.

Rails has **one** spelling. `ActiveRecord::Callbacks`'s `included do` block
(`activerecord/lib/active_record/callbacks.rb:413`) does
`include ActiveModel::Validations::Callbacks`, whose `ClassMethods`
(`activemodel/lib/active_model/validations/callbacks.rb:32-110`) is where
`before_validation` / `after_validation` live — and as of PR #6923 that is
exactly where the trails port lives too, reached on `Base` through
`Model`'s `extend(Model, ValidationsCallbacksClassMethods)`.

So these two free functions are now a second, divergent spelling of a macro
that already exists at the Rails name, in the Rails module. They are the last
survivors of the group: PR #6916 deleted the eight lifecycle free functions
(`beforeSave` … `afterDestroy`) and PR #6923 deleted `afterFind` /
`afterInitialize` for the same reason.

Note their options shape also diverges: `registerCallback` `:134` forwards `on:`
only when `event === "validation"`, and types it `"create" | "update"` — Rails
puts no such restriction on the value and converts it through
`set_options_for_callback` (`validations/callbacks.rb:99-109`).

## Converged shape

Delete `beforeValidation`, `afterValidation`, `registerCallback` and the types
left orphaned by their removal. Every caller spells `Model.beforeValidation(fn,
options)` — the Rails macro — the way `builder/belongs_to.rb`'s call sites were
converged onto `model.after_create` in #6916.

## Acceptance criteria

- `beforeValidation` / `afterValidation` / `registerCallback` no longer exist in
  `packages/activerecord/src/callbacks.ts`; callers use the ClassMethods macro.
- `pnpm parity:api:extra --package activerecord` loses the rows.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no reseed.
