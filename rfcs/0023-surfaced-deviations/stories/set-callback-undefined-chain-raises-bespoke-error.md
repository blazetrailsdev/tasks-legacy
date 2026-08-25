---
title: "set_callback on an undefined chain raises a bespoke Error, not Ruby's NoMethodError"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby reaches a callback chain through the reader `define_callbacks` generates:
`get_callbacks name` is `send "_#{name}_callbacks"`
(activesupport/lib/active_support/callbacks.rb:823-841, :690-695). Calling
`set_callback :save, :before, …` on a class that never ran
`define_callbacks :save` therefore raises `NoMethodError` for
`_save_callbacks`, from Ruby itself.

trails' `Callbacks.setCallback` raises a bespoke
`new Error('No callback chain "save" defined. Call defineCallbacks first.')`
(`packages/activesupport/src/callbacks.ts:1467-1471`) — wrong class, wrong
message, and a sentence of trails-specific advice Rails does not print.

PR #6951 made this reachable where it had been masked: `Model.setCallback` used
to route through ActiveModel's `_registerCallbackOnProto`, which called
`defineCallbacks` first and so silently auto-defined any chain. Removing that
passthrough surfaced two genuinely-unported `included do define_callbacks`
blocks (both fixed in #6951) and left this error as the thing callers now hit.

## Converged shape

Raise `NoMethodError` with Ruby's message shape for the missing generated
reader — `undefined method '_save_callbacks' for class Thing` — from the same
place `get_callbacks` would have. trails already has `NoMethodError` and
already models this exact translation in `defineModelCallbacks`
(`packages/activemodel/src/callbacks.ts:71-77`), which raises `NoMethodError`
for the `send "_define_#{type}_model_callback"` that Ruby would have missed.

Check `packages/activesupport/src/callbacks.test.ts` for ported Rails tests
asserting the current string before changing it, and check the
`rails-error-parity` grandfather list — this error may be on it.

## Acceptance criteria

- [ ] `setCallback` on an undefined chain raises `NoMethodError`, not `Error`,
      with Ruby's `undefined method '_<name>_callbacks' for class <X>` message.
- [ ] The bespoke "Call defineCallbacks first." advice is gone.
- [ ] If the string is on the `rails-error-parity` grandfather list, the row
      comes off in the same PR.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
