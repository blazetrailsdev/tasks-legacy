---
title: "Engine#app should build the middleware stack through MiddlewareStackProxy#merge_into"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by the RFC 0106 `@missingRailsCall` permanence audit (PR #6855). Two
CONVERGEABLE tags on `packages/trailties/src/application.ts` could NOT be
returned to `call-mismatches-exclude/` as baseline rows, because the call-set
gate does not compare that pair — a row for a call that never flags is a STALE
row by construction. So they stay as call-site tags with nothing tracking the
work. This story is that tracker.

`railties/lib/rails/engine.rb:519` (`Engine#app`):

    config.middleware = build_middleware.merge_into(stack)

`MiddlewareStackProxy` is not ported (`trailtie/configuration.ts:87` —
`appMiddleware()` returns undefined), so trails has no queued middleware
operations to merge and assigns the default stack to `config.middleware`
directly. Both halves of the line — `build_middleware` and `merge_into` — are
dropped for the same reason.

`Engine#build_middleware` is `config.app_middleware + config.middleware`, both
`MiddlewareStackProxy`s; `MiddlewareStackProxy#merge_into` replays its queued
`use` / `insert_before` / `swap` / `delete` operations onto the passed stack.

## Converged shape

Port `ActionDispatch::MiddlewareStack::MiddlewareStackProxy` (its recorded
operation queue and `merge_into`), have `TrailtieConfiguration#appMiddleware`
and `#middleware` return proxies, and then spell `application.ts`'s `app` builder
as `engine.rb:517-522` does: `config.middleware = buildMiddleware().mergeInto(stack)`.
Drop both `@missingRailsCall` tags once the calls are made.

## Acceptance criteria

- [ ] `MiddlewareStackProxy` is ported with its operation queue and `mergeInto`.
- [ ] `appMiddleware()` returns a proxy rather than undefined.
- [ ] `application.ts` calls `buildMiddleware()` and `mergeInto()` at Rails' call
      site, so queued middleware operations actually reach the default stack.
- [ ] Both `@missingRailsCall` tags are deleted from `application.ts`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
