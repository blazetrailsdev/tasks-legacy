---
title: "Rack::Utils settings are get*/set* pairs where Rack has reader/writer"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

`packages/rack/src/utils.ts:19-45` exposes the module-level Rack settings as
`get*`/`set*` pairs — `getDefaultQueryParser` / `setDefaultQueryParser`,
`getParamDepthLimit` / `setParamDepthLimit`, `getMultipartFileLimit`,
`getMultipartTotalPartLimit` — while Rack spells them as a reader and a writer:
`Rack::Utils.default_query_parser` / `default_query_parser=` and
`param_depth_limit` / `param_depth_limit=` (`vendor/rack/lib/rack/utils.rb:82-88`).
Per `docs/ruby-ts-conventions.md` those map to `defaultQueryParser()` and
`setDefaultQueryParser()`, so the `get`-prefixed spelling matches no Ruby name.

The file already carries the converged reader as a separate overloaded
`defaultQueryParser()` / `defaultQueryParser(parser)` accessor
(`utils.ts:47-58`, marked `@internal`) — i.e. BOTH spellings exist side by
side, and PR #6452 had to route `setParamDepthLimit` through the converged one
to satisfy the call gate (`utils.ts:29`).

## Converged shape

One accessor per Ruby name: `defaultQueryParser()` + `setDefaultQueryParser()`,
`paramDepthLimit()` + `setParamDepthLimit()`, `multipartFileLimit()` +
`setMultipartFileLimit()`, `multipartTotalPartLimit()` +
`setMultipartTotalPartLimit()`. Delete the `get*` duplicates and the
Ruby-less overloaded-getter/setter form, and update the callers
(`packages/rack/src/index.ts:52`, `packages/rack/src/request.ts:45,717`, and
`utils.test.ts`).

## Acceptance criteria

- [ ] No `get*`-prefixed accessor remains in `packages/rack/src/utils.ts`.
- [ ] `pnpm parity:api --package rack` delta non-negative; `parity:api:extra
--package rack` shows no new novel names (it should shrink).
- [ ] `pnpm parity:api:calls` stays clean.
