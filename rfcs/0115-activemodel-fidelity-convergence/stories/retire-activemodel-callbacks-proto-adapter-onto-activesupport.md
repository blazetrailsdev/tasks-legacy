---
title: "Retire activemodel/callbacks.ts's proto-registration adapter onto ActiveSupport::Callbacks"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel", "activesupport"]
deps:
  - retire-model-set-callback-skip-callback-run-callbacks-passthrough
deps-rfc: []
est-loc: 340
pr: 6964
claim: "2026-08-24T01:51:39Z"
assignee: "retire-activemodel-callbacks-proto-adapter-onto-activesupport"
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/callbacks.rb` is **50 code lines**:
`define_model_callbacks` (`:109`) and its three private generators (`:129`,
`:136`, `:143`). Everything else it needs comes from
`include ActiveSupport::Callbacks`.

`packages/activemodel/src/callbacks.ts` is **342 code lines**, of which 72 map
onto `callbacks.rb` (the four faithful members, `:34`–`:193`) and **203 have no
Rails counterpart**:

`_resolveCallbackObject` `:204`, `_buildAfterModelIfConditions` `:265`,
`_registerCallbackOnProto` `:285`, `hasBeforeOrAroundCallbackOnProto` `:357`,
`beforeOrAroundCallbackSources` `:381`, `hasCallbackOnProto` `:409`,
`skipCallbackOnProto` `:424`, `snapshotCallbacksOnProto` `:447`,
`restoreCallbacksOnProto` `:459`, `runAllCallbacks` `:479`,
`runBeforeCallbacksOnProto` `:503`, `runAfterCallbacksOnProto` `:523`,
`isThenable` `:196`.

They are an adapter over `packages/activesupport/src/callbacks.ts`, which is a
**1,631-line faithful port** of `ActiveSupport::Callbacks` — `Callback` (`:626`),
`CallbackChain` (`:1094`), `CallbackSequence` (`:772`), `Before`/`After`/
`Around` (`:405`/`:547`/`:594`), and the five module functions
`defineCallbacks` (`:1525`), `setCallback` (`:1533`), `skipCallback` (`:1541`),
`resetCallbacks` (`:1549`), `runCallbacks` (`:1553`). `callbacks.ts:18-22`
already imports three of them under `as*` aliases and then wraps them.

Rails has no adapter because the module is `include`d. CLAUDE.md's "Module
mixins" section names the trails equivalent: `include()` / `Included<>` from
`@blazetrails/activesupport`.

Depends on `retire-model-set-callback-skip-callback-run-callbacks-passthrough`
and the two macro stories landing first — while `model.ts` still calls
`_registerCallbackOnProto` directly, this layer cannot be removed.

## Acceptance criteria

- `packages/activemodel/src/callbacks.ts` contains `defineModelCallbacks` and
  the three `_define*ModelCallback` generators, and nothing else without a
  Rails counterpart.
- Callers reach `set_callback` / `skip_callback` / `reset_callbacks` /
  `run_callbacks` through `@blazetrails/activesupport` directly.
- Where a wrapper carried real behaviour ActiveSupport's port lacks (the
  `skipAfterCallbacksIfTerminated` default at `callbacks.ts:312`, thenable
  handling in `runAllCallbacks`), that behaviour is pushed into the
  ActiveSupport port at its Rails-named site — not left in a shim.
- `pnpm parity:api:extra --package activemodel` shows `callbacks.ts` at ≤ 1
  novel; `activemodel/callbacks.json`'s 3 baseline rows shrink or hold.
- Parity deltas non-negative for activemodel **and** activesupport.

## Verification

```bash
pnpm vitest run packages/activemodel/src/callbacks.test.ts packages/activesupport/src/callbacks.test.ts packages/activesupport/src/callbacks.trails.test.ts
```
