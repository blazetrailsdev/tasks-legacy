---
title: "converge-journey-mapping-onto-ported-mapper-mapping"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/actionpack/src/action-dispatch/journey/routes.ts:11` declares a local
`interface Mapping { makeRoute(name, index) }` as a stand-in for
`ActionDispatch::Routing::Mapper::Mapping`
(`vendor/rails/actionpack/lib/action_dispatch/routing/mapper.rb:83`), which
trails has not ported — `packages/actionpack/src/action-dispatch/routing/mapper.ts`
declares no `Mapping` class. Rails' `Journey::Routes#add_route` receives a real
`Mapper::Mapping` instance (`journey/routes.rb`), so the interface is deferred
porting work, not a coincidental name collision.

Found by the RFC 0080 audit of `moved` interface declaration names
(`audit-moved-interface-declaration-names`), which tagged it
`@noRailsEquivalent CONVERGEABLE (story: <this story>)`.

## Acceptance criteria

- `ActionDispatch::Routing::Mapper::Mapping` is ported into
  `action-dispatch/routing/mapper.ts` in Rails layout.
- `journey/routes.ts` types `addRoute` on the ported class and the local
  `Mapping` interface plus its `@noRailsEquivalent` tag are deleted.
- `pnpm parity:api:extra` exits 0 (no stale tag).
