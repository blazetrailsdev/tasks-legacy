---
title: "LazyRouteSet injects a reload hook Rails names inline (engine/lazy_route_set.rb:12-104)"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down trailties' `@internal` tags for RFC 0121 (#7045).

`packages/trailties/src/engine/lazy-route-set.ts` exports a module-level
injection seam:

```ts
let reloadHook: ReloadHook = () => undefined;
export function setReloadRoutesHook(fn: ReloadHook): void {
  reloadHook = fn;
}
export function resetReloadRoutesHook(): void {
  reloadHook = () => undefined;
}
```

Rails has no setter and no stored hook. `LazyRouteSet` names the constant
directly at each of its 15 call sites:

```ruby
# vendor/rails/railties/lib/rails/engine/lazy_route_set.rb:12-104
Rails.application&.reload_routes_unless_loaded
```

Ruby resolves `Rails.application` when the method RUNS, so no seam is needed.
trails injects instead because `Trails.application` does not exist yet. Both
functions carry `@internal` plus a `@noRailsEquivalent CONVERGEABLE` receipt
pointing here.

Related but distinct: `0104-twitter-app-full-stack-integration/complete-set-routes-reloader-hook`
covers `Application#reloaders` + `reloader.to_run` in `finisher.ts`. This story
is only about deleting the module-level injection seam in `lazy-route-set.ts`.

## Converged shape

Once `Trails.application` exists, replace every `reloadHook()` call with
`Trails.application?.reloadRoutesUnlessLoaded()` named inline, matching
`lazy_route_set.rb:12-104`, and delete both exported functions along with their
receipts. Tests that currently assert "each routing op consults the hook exactly
once" should assert against `Trails.application` instead.

## Acceptance criteria

- `setReloadRoutesHook` / `resetReloadRoutesHook` deleted.
- Every routing op names `Trails.application?.reloadRoutesUnlessLoaded()`
  inline, once per op, as Rails does.
- Both `@noRailsEquivalent` receipts gone; `pnpm parity:api:extra` reports no
  STALE tag.
- `pnpm exec eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs "packages/trailties/src/**/*.ts"`
  still clean.
