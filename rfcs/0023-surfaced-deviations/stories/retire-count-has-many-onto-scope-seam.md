---
title: "Retire countHasMany now that the scope seam exists"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`countHasMany` (`packages/activerecord/src/associations.ts`) is a trails-only
helper with no Rails counterpart. Its own JSDoc says so: Rails' `reset_counters`
counts with `object.send(counter_association).count(:all)`, where the proxy
delegates to `association.scope` — side-effect-free by construction.

It existed because trails had no `scope` seam, so a caller could not get an
unexecuted relation without going through the caching / strict-loading /
inverse_of fusion in `findTarget`. PR #5939 removed that reason: `scope()`
(`packages/activerecord/src/associations/has-many-association.ts`) now returns
exactly the relation `findTarget` runs, and `countHasMany` was reduced to a thin
wrapper over it —

```ts
const rel = scope(record, assocName, options);
if (!rel) return 0;
const result = await rel.count();
```

— plus a `_strictLoadingBypassCount` bump and a `_hmtNotFound` guard for the
through arm.

Remaining work is to retire the wrapper: route its callers at
`scope(...).count()` directly, and find each piece a Rails home — the
strict-loading bypass belongs with the association's `skip_strict_loading`
(`association.rb`), and the misconfigured-through raise belongs on the through
reflection's validity check, not in a counting helper.

## Acceptance criteria

- `countHasMany` is deleted from `packages/activerecord/src/associations.ts`,
  with every caller routed through the `scope` seam.
- The strict-loading bypass and the `_hmtNotFound` through guard are preserved at
  Rails-named homes rather than dropped; cite the Rails `file:line` for each at
  the call site.
- `resetCounters` still works on strict-loading models (its original reason for
  the bypass) and misconfigured-through associations still fail loudly rather
  than counting 0.
- `pnpm parity:api:extra --package activerecord --novel-only` does not regress.
- No test renames.
