---
title: "Engine::Updater injects a generator factory Rails builds inline (engine/updater.rb:10-13)"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down trailties' `@internal` tags for RFC 0121 (#7045).

`packages/trailties/src/engine/updater.ts` carries three static methods with no
Rails counterpart — `setGeneratorFactory`, `resetGenerator`, `reset` — which
exist to inject and tear down a generator factory.

Rails builds the generator inline and memoizes it in an ivar; there is no
factory, no setter, and no teardown:

```ruby
# vendor/rails/railties/lib/rails/engine/updater.rb:10-13
def generator
  @generator ||= Rails::Generators::PluginGenerator.new ["plugin"],
    { engine: true }, { destination_root: ENGINE_ROOT }
end
```

trails injects because it has neither `PluginGenerator` nor `ENGINE_ROOT` yet.
All three carry `@internal` plus a `@noRailsEquivalent CONVERGEABLE` receipt
pointing here.

## Converged shape

Once `Generators::PluginGenerator` and `ENGINE_ROOT` are ported, inline the
construction into `Updater.generator` exactly as `engine/updater.rb:10-13` does,
memoizing on a single field, and delete all three seam methods plus their
receipts. Tests that install a factory should construct the real generator (or
stub at the `PluginGenerator` boundary) instead.

## Acceptance criteria

- `Updater.generator` constructs `PluginGenerator` inline and memoizes it,
  mirroring `engine/updater.rb:10-13`.
- `setGeneratorFactory`, `resetGenerator`, `reset` deleted with their receipts.
- `pnpm parity:api:extra` reports no STALE tag.
- trailties suite green.
