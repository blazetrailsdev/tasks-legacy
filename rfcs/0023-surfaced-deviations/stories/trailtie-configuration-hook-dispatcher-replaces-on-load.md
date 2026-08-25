---
title: "Trailtie::Configuration hand-rolls a hook dispatcher Rails routes through ActiveSupport.on_load (railtie/configuration.rb:54-77)"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

Surfaced while burning down trailties' `@internal` tags for RFC 0121 (#7045).

`packages/trailties/src/trailtie/configuration.ts` keeps a class-side
`_lifecycleBlocks` registry plus two methods with no Rails counterpart —
`runHook(hook, ...args)` and `lifecycleHooks()` — and a `LIFECYCLE_HOOKS` array
listing the five hook names.

Rails has no dispatcher and no list. Each hook is its own method that registers
through `ActiveSupport.on_load` with `yield: true`, and `run_load_hooks` fires
it; nothing enumerates them:

```ruby
# vendor/rails/railties/lib/rails/railtie/configuration.rb:54-77
def before_configuration(&block)
  ActiveSupport.on_load(:before_configuration, yield: true, &block)
end
# ... before_eager_load:60, before_initialize:65,
#     after_initialize:70, after_routes_loaded:75
```

The file's own header already admits the deviation: "Rails routes them through
`ActiveSupport.on_load` with `yield: true`; a direct class-side array matches
the observable behavior without coupling to a hook surface that doesn't expose
that semantic yet." Both methods carry `@internal` plus a
`@noRailsEquivalent CONVERGEABLE` receipt pointing here.

## Converged shape

Once activesupport's `on_load` / `run_load_hooks` supports `yield: true`, each
of the five hook methods registers through it directly
(`railtie/configuration.rb:54-77`), the callers fire via `run_load_hooks`, and
`_lifecycleBlocks`, `LIFECYCLE_HOOKS`, `runHook` and `lifecycleHooks` all go
away with their receipts.

Check `packages/activesupport/src/lazy-load-hooks.ts` for the current state of
`on_load` before starting — the `yield: true` arm is the blocking piece.

## Acceptance criteria

- The five lifecycle hook methods register via `ActiveSupport.on_load(..., yield: true)`,
  mirroring `railtie/configuration.rb:54-77`.
- `runHook`, `lifecycleHooks`, `LIFECYCLE_HOOKS` and `_lifecycleBlocks` deleted
  along with their receipts.
- `pnpm parity:api:extra` reports no STALE tag; trailties suite green.
