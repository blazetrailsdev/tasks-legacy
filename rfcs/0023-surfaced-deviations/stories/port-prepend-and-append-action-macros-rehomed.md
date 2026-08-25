---
title: "Port the six unported prepend_/append_ action callback macros"
status: draft
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Re-homed from `0072-api-compare-parity-burndown`, where
`port-prepend-and-append-action-callback-macros` was closed for scope, not for
lack of merit: _"Out of scope for AR-focused 0072 burndown:
actionpack/abstractcontroller is in the web/framework stack, not ActiveRecord's
dependency graph. Reopen/re-home under a web-stack parity RFC if desired."_
(`port-prepend-and-append-action-macros` was closed as a duplicate of it.)

**New evidence justifying the re-home:** PR #5435 taught the Ruby extractor to
record `define_method` / `alias_method` surface, so these six are no longer
invisible to tooling. `pnpm parity:api --package abstractcontroller` now scores
`callbacks.rb` at **15/23**, and the eight misses name them directly:

```text
- prepend_before_action → prependBeforeAction
- prepend_after_action  → prependAfterAction
- prepend_around_action → prependAroundAction
- append_before_action  → appendBeforeAction
- append_after_action   → appendAfterAction
- append_around_action  → appendAroundAction
```

Both closed stories predate that and had to argue the gap from Rails source;
it is now a standing, measurable parity miss.

`vendor/rails/actionpack/lib/abstract_controller/callbacks.rb:230-252` generates
twelve macros in one loop; trails ports six.

```ruby
[:before, :after, :around].each do |callback|
  define_method "#{callback}_action" do |*names, &blk| ... end       # ported
  define_method "prepend_#{callback}_action" do |*names, &blk|       # MISSING
    _insert_callbacks(names, blk) do |name, options|
      set_callback(:process_action, callback, name, options.merge(prepend: true))
    end
  end
  define_method "skip_#{callback}_action" do |*names| ... end        # ported
  alias_method :"append_#{callback}_action", :"#{callback}_action"   # MISSING
end
```

`prepend_*_action` passes `options.merge(prepend: true)` to `set_callback`;
`append_*_action` is a plain alias of the base macro. Ported surface lives in
`packages/actionpack/src/abstract-controller/callbacks.ts` (the six existing
macros are `beforeAction`, `afterAction`, `aroundAction`, `skipBeforeAction`,
`skipAfterAction`, `skipAroundAction`), installed onto the class in
`packages/actionpack/src/abstract-controller/base.ts`.

## Acceptance criteria

- `callbacks.ts` exports `prependBeforeAction` / `prependAfterAction` /
  `prependAroundAction`, threading `prepend: true` into the `set_callback`
  options, and `appendBeforeAction` / `appendAfterAction` /
  `appendAroundAction` as aliases of the base macros.
- All six are installed at the same `base.ts` site as the existing six.
- `pnpm parity:api --package abstractcontroller` shows `callbacks.rb` with
  those six no longer missing.
- Tests mirror the Rails ones; test names match Rails verbatim.
