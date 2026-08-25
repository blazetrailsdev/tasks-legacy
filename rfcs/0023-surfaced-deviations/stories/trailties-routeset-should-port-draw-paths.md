---
title: "add_routing_paths should concat external paths onto RouteSet#draw_paths"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by the RFC 0106 `@missingRailsCall` permanence audit (PR #6855). A
CONVERGEABLE tag on `packages/trailties/src/engine.ts` could NOT be returned to
`call-mismatches-exclude/` as a baseline row, because the call-set gate does not
compare that pair — a row for a call that never flags is a STALE row by
construction. So it stays a call-site tag with nothing tracking the work. This
story is that tracker.

`railties/lib/rails/engine.rb:595-599` (the `add_routing_paths` initializer):

    routing_paths = paths["config/routes.rb"].existent
    external_paths = self.paths["config/routes"].paths
    routes.draw_paths.concat(external_paths)
    app.routes.draw_paths.concat(external_paths)

`ActionDispatch::Routing::RouteSet#draw_paths` is not ported, so trails records
the external paths on the routes reloader only. `draw_paths` is what lets `draw`
resolve a partial route file by a relative name, so an engine shipping
`config/routes/*.rb` fragments cannot resolve them.

## Converged shape

Port `RouteSet#draw_paths` (the array `draw` searches when resolving a relative
route-file name), then spell `engine.ts`'s `add_routing_paths` initializer as
`engine.rb:595-606` does: concat `externalPaths` onto both `routes.drawPaths`
and `app.routes.drawPaths` before the reloader registration. Drop the
`@missingRailsCall` tag once the calls are made.

## Acceptance criteria

- [ ] `RouteSet#drawPaths` is ported and consulted by `draw` when resolving a
      relative route-file name.
- [ ] `engine.ts`'s `add_routing_paths` concats onto both `drawPaths` arrays at
      Rails' call sites.
- [ ] The `@missingRailsCall` tag is deleted from `engine.ts`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
