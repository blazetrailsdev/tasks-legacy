---
title: "Converge defineModelCallbacks' options handling onto callbacks.rb:109-127"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
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

`vendor/rails/activemodel/lib/active_model/callbacks.rb:109-127` is nine code
lines:

```ruby
def define_model_callbacks(*callbacks)
  options = callbacks.extract_options!
  options = {
    skip_after_callbacks_if_terminated: true,
    scope: [:kind, :name],
    only: [:before, :around, :after]
  }.merge!(options)

  types = Array(options.delete(:only))

  callbacks.each do |callback|
    define_callbacks(callback, options)

    types.each do |type|
      send("_define_#{type}_model_callback", self, callback)
    end
  end
end
```

`packages/activemodel/src/callbacks.ts:42-92` is a ~50-line hand-rolled
argument walk. PR #6916 retired `model.ts`'s twelve lifecycle macros onto this
function and converged the three generators (they were missing
`options.assert_valid_keys(:if, :unless, :prepend)`, callbacks.rb:131/:138/:145),
but left the outer body as it was. It diverges from the Ruby in five ways:

1. **No defaults hash, no `merge!`.** `skipAfterCallbacksIfTerminated: true` is
   hardcoded into the `asDefineCallbacks` call (`:88`) instead of being a
   default a caller can override, and `scope: [:kind, :name]`
   (callbacks.rb:113) is dropped entirely rather than forwarded.
2. **`define_callbacks` gets the wrong argument.** Rails passes the whole merged
   `options` (callbacks.rb:122); trails passes a fresh
   `{ skipAfterCallbacksIfTerminated: true }` literal.
3. **Invented `ArgumentError`s.** `Unknown option: ${key}` (`:60`),
   `Invalid callback type: ${t}` (`:65`) and `At least one event name must be
provided to defineModelCallbacks` (`:78`) have no Rails counterpart. Rails
   forwards any unrecognised option straight to `define_callbacks` (so
   `terminator:` and friends work), accepts any `only:` value `Array()` accepts,
   and simply iterates zero times for zero callbacks.
4. **`only:` order is ignored.** Rails drives the generators from `types` in the
   caller's order (callbacks.rb:124-126); trails hardcodes before → after →
   around (`:89-91`).
5. **`extract_options!` is inlined** as an index/typeof test (`:50-56`) rather
   than the ActiveSupport method.

## Acceptance criteria

- `defineModelCallbacks` builds the Rails defaults object, merges the caller's
  options over it, deletes `only` from it, and forwards the remainder to
  `defineCallbacks` — matching callbacks.rb:110-122 statement for statement.
- The generator dispatch is driven by `types` in the caller's order
  (callbacks.rb:124-126).
- The three invented `ArgumentError`s are deleted; no error is raised that
  Rails does not raise at that site.
- `scope` reaches `defineCallbacks`, or its absence is justified at the call
  site against `activesupport/lib/active_support/callbacks.rb`'s handling of
  it — `_resolveCallbackObject`'s camelCase lookup is today's implicit
  `scope: [:kind, :name]` and the two must not silently disagree.
- `pnpm vitest run packages/activemodel/src/callbacks.test.ts` and
  `packages/activerecord/src/callbacks.test.ts` pass; no test renamed or
  reworded.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed.

## Notes

Ordering: independent of
`retire-activemodel-callbacks-proto-adapter-onto-activesupport`, which retires
the surrounding `*OnProto` adapter but treats `defineModelCallbacks` itself as
one of the four already-faithful members. Landing this one first keeps that
assumption true.
