---
title: "Retire model.ts's lifecycle callback macros onto define_model_callbacks"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel", "activerecord"]
deps:
  - move-ar-save-side-dirty-surface-out-of-model
deps-rfc: []
est-loc: 380
priority: null
pr: 6916
claim: "2026-08-23T14:42:26Z"
assignee: "retire-model-lifecycle-callback-macros-onto-define-model-callbacks"
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/callbacks.rb:109-127`
`define_model_callbacks(*callbacks)` **generates** `before_<event>`,
`around_<event>` and `after_<event>` for each event, via the three private
generators `_define_before_model_callback` (`:129`),
`_define_around_model_callback` (`:136`) and `_define_after_model_callback`
(`:143`). `vendor/rails/activerecord/lib/active_record/callbacks.rb` then does
`define_model_callbacks :initialize, :find, :touch` and
`define_model_callbacks :save, :create, :update, :destroy`. `ActiveModel::Model`
itself has **no** `before_save`.

`packages/activemodel/src/callbacks.ts:34-193` already ports
`defineModelCallbacks` and all three generators faithfully. Nothing in the
lifecycle path calls it.

Instead `model.ts` hand-writes the macros, 13–16 code lines each:

`beforeSave` `:1082`, `afterSave` `:1097`, `beforeCreate` `:1112`,
`afterCreate` `:1127`, `beforeUpdate` `:1142`, `afterUpdate` `:1157`,
`beforeDestroy` `:1172`, `afterDestroy` `:1187`, `aroundSave` `:1202`,
`aroundCreate` `:1219`, `aroundUpdate` `:1236`, `aroundDestroy` `:1253`.

**196 code lines**, twelve of them among `model.ts`'s 20 `novel` names in
`pnpm parity:api:extra --package activemodel`. Each body is the same three
statements: `_rejectOnOption(conditions)` then `_registerCallbackOnProto(
this.prototype, timing, event, fn, conditions)`.

`packages/activerecord/src/callbacks.ts:82-173` **already exports**
`beforeSave`, `afterSave`, `beforeCreate`, `afterCreate`, `beforeUpdate`,
`afterUpdate`, `beforeDestroy`, `afterDestroy` as free functions — they are
shadowed by the `Model` statics.

Scope note: `afterCommit` / `afterRollback` / `afterInitialize` / `afterFind` /
`afterTouch` and `setCallback` / `skipCallback` / `resetCallbacks` /
`runCallbacks` are the next two stories; leave them here.

## Design note the implementer must settle

Rails runs `define_model_callbacks` at include time. JS has no `inherited`
hook. `model.ts:1453` `_ensureOwnValidators` documents trails' settled
workaround for exactly this shape — copy-on-first-write. State in the PR body
which the callback registration uses and why; do not invent a third mechanism.

## Acceptance criteria

- `Base` (or whichever class Rails' `include ActiveRecord::Callbacks` maps to)
  obtains `beforeSave` … `aroundDestroy` by calling `defineModelCallbacks` with
  the Rails event list, not by hand-written statics.
- The twelve `Model` statics at `model.ts:1082-1268` are deleted.
- `packages/activerecord/src/callbacks.ts`'s free functions are either the
  generated result or are deleted as redundant — one spelling, not two
  (RFC 0112's rule).
- `defineModelCallbacks` in `packages/activemodel/src/callbacks.ts` is
  unchanged except where a real divergence from `callbacks.rb:109-127` is
  found; if one is, converge it and say so.
- `pnpm parity:api:extra --package activemodel` loses twelve `novel` rows from
  `model.ts`.
- `pnpm vitest run packages/activerecord/src/callbacks.test.ts` and the
  activemodel callbacks suites pass unchanged; no test renamed or reworded.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed; stranded `call-mismatches-exclude/activemodel/callbacks.json` rows
  hand-deleted then tightened.
