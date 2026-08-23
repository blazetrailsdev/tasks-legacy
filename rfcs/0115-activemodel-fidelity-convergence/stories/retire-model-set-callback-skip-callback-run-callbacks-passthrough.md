---
title: "Retire model.ts's setCallback/skipCallback/resetCallbacks/runCallbacks passthrough"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - retire-model-transactional-and-find-callback-macros
deps-rfc: []
est-loc: 200
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/model.ts` carries four statics/methods that are pure
passthrough to `packages/activemodel/src/callbacks.ts`, which is itself a
passthrough to the faithful `ActiveSupport::Callbacks` port in
`packages/activesupport/src/callbacks.ts`:

- `setCallback` — three overload signatures plus one body, `:1365`, `:1372`,
  `:1381` (29 code lines total)
- `skipCallback` — three, `:1415`, `:1421`, `:1429` (21)
- `resetCallbacks` `:1449` (3)
- `runCallbacks` `:2790` (6)

59 code lines. Rails' equivalents are
`activesupport/lib/active_support/callbacks.rb:737` `set_callback`, `:786`
`skip_callback`, `:811` `reset_callbacks`, and `run_callbacks`, all obtained by
`include ActiveSupport::Callbacks` — `ActiveModel::Model` re-declares none of
them. trails already exports all four from `@blazetrails/activesupport`
(`packages/activesupport/src/callbacks.ts:1525,1533,1541,1549,1553`) and
`callbacks.ts` imports three of them under aliases (`asDefineCallbacks`,
`asSkipCallback`, `asResetCallbacks`, `callbacks.ts:18-22`).

The passthroughs are also **divergent**, which is the fidelity cost of keeping
them: `skipCallback`'s JSDoc (`model.ts:1400-1412`) states that Rails raises
when no callback matches unless `raise: false` and that trails returns a
boolean instead, and that Rails' conditional `skip_callback(..., if:)` is not
supported. Both are convergence work, not documentation.

## Acceptance criteria

- `Model` obtains `setCallback` / `skipCallback` / `resetCallbacks` /
  `runCallbacks` through `include()` / `Included<>` from
  `@blazetrails/activesupport` per CLAUDE.md "Module mixins" — no hand-written
  statics in `model.ts`.
- `skipCallback` raises where `callbacks.rb:786-808` raises, with Rails' error
  class and message, and honours `raise: false`; the boolean-return deviation
  and its JSDoc are gone.
- Conditional `skip_callback(..., if:)` works, or the story is re-scoped with a
  filed follow-up naming the missing `CallbackChain` capability — not a
  reworded justification.
- `pnpm parity:api:extra --package activemodel` loses `setCallback`,
  `skipCallback`, `resetCallbacks`, `runCallbacks` from `model.ts`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed.
