---
title: "Base.findGlobalId/findSignedGlobalId[!] are uncalled trails inventions"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "globalid"
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

`Base.findGlobalId`, `Base.findSignedGlobalId` and `Base.findSignedGlobalIdBang`
(`packages/activerecord/src/base.ts:4152-4185`) are trails inventions: Rails apps
call `GlobalID::Locator.locate` / `locate_signed` directly, and globalid's
railtie injects only the instance-side `GlobalID::Identification` (`to_gid`
family) onto `ActiveRecord::Base`. There is no `find_global_id` `def` anywhere in
`vendor/rails` or the vendored globalid gem.

Until #5368 they were hidden from `parity:api:extra` by the `methods` arm of
`AMBIENT_RAILTIE_MIXINS["ActiveRecord::Base"]`
(`scripts/api-compare/extra-surface.ts`); that PR moved the justification onto
the declarations as `@noRailsEquivalent` (RFC 0080), which made the deviation
visible — and visible is where it should be decided, not permanently tagged.

The convergence case is that they have **no callers and no tests**: a grep over
`packages/**/*.ts` for `findGlobalId|findSignedGlobalId` outside `src/base.ts`
and `dist/` returns only `packages/globalid/src/wire.ts:1-2` (a comment) and a
prose mention in `signed-global-id.ts:321`. The repo has precedent for deleting
unused invented surface rather than justifying it: `AbstractAdapter::Version`'s
`major`/`minor`/`patch` readers and the base-class `createRange`/`dropRange`
stubs were both deleted for exactly this reason (see their neighbours' reasons in
`extra-surface-allow.json`).

Secondary cleanup in the same area: `packages/globalid/src/wire.ts:1-5` describes
a GID-4 design ("will register the Locator and findGlobalId/findSignedGlobalId
class methods onto AR Base here via a registration callback") that was never
built — `base.ts` declares them statically. Either the comment or the design is
stale.

## Acceptance criteria

- Decide and record: delete the three finders, or keep them with a caller/test
  that justifies the surface. Deleting is the default given zero consumers.
- If deleted: the three `@noRailsEquivalent` tags in `base.ts` go with them, and
  `pnpm parity:api && pnpm parity:api:extra` stays green with activerecord `Allowed`
  dropping by 3 and no new extras.
- `packages/globalid/src/wire.ts`'s GID-4 comment is corrected to describe what
  the code actually does (or the registration path is built, if that is the
  decision).
- No behaviour change for `toGlobalId` / `toSgid` / `GlobalID::Locator`, which
  are faithful ports and stay exactly as they are.
