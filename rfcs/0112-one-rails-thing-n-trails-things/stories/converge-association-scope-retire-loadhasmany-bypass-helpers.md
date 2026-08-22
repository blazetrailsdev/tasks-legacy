---
title: "Un-fuse loadHasMany so association.scope replaces the three bypass helpers"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 200
pr: 6757
claim: "2026-08-20T01:56:44Z"
assignee: "consolidate-duplicated-through-association-module"
blocked-by: null
closed-reason: null
---

## Context

trails has three exported functions in
`packages/activerecord/src/associations.ts` that have no Rails counterpart and
exist for one shared reason: `loadHasMany` fuses relation building with target
caching, strict-loading enforcement and `inverse_of` wiring, so there is no way
to obtain an unexecuted, side-effect-free association relation.

- `buildHasManyRelation` (already `@internal`)
- `buildThroughJoinScope` (already `@internal`)
- `countHasMany` (tagged `@internal` in #5359)

In Rails all three are the same one thing: `Association#scope`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:107`)
delegating to `AssociationScope#scope`
(`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb:21`).
`HasManyAssociation#count_records`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_association.rb:84`)
just calls `scope.count(:all)` on it, and
`CounterCache::ClassMethods#reset_counters`
(`vendor/rails/activerecord/lib/active_record/counter_cache.rb:57`) counts with
`object.send(counter_association).count(:all)` — through the proxy, no bypass
helper anywhere.

Surfaced while retiring `resolveCounterColumn` in #5359. That PR converged
`resetCounters` onto the Rails body but had to keep calling `countHasMany`,
because the faithful `object.send(counter_association).count(:all)` is not
reachable while the fusion stands. Note `HasManyAssociation#countRecords`
already exists at
`packages/activerecord/src/associations/has-many-association.ts` and is a
correct, separate port (counter cache + target trim + `limit_value` clamp) —
this story must not collapse the two.

Un-fusing `loadHasMany` also removes the strict-loading bypass counter
`_strictLoadingBypassCount`, which only exists to let these helpers run against
strict-loading models.

## Acceptance criteria

- A side-effect-free association scope is reachable without the three helpers
  (caching, strict-loading and `inverse_of` split out of relation building).
- `buildHasManyRelation`, `buildThroughJoinScope` and `countHasMany` are
  deleted from `associations.ts`; `associations.ts` novel extra count drops by
  three.
- `resetCounters` counts via the association/proxy, matching
  `object.send(counter_association).count(:all)`.
- `_strictLoadingBypassCount` is gone, or its remaining need is justified at
  the declaration.
- `HasManyAssociation#countRecords` is left intact and still passes.
- Counter-cache, has-many and through association suites pass with no test
  renames.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
