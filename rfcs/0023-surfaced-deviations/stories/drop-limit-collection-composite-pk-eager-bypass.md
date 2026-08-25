---
title: "Teach columns_for_distinct/materializeLimitedIds composite keys; drop the LIMIT+collection composite-PK eager bypass"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

After PR #4917 and #4924, `_eagerLoadBypassesJoinDependency`
(`packages/activerecord/src/relation.ts:3109`) JOINs composite-PK eager loads
EXCEPT the LIMIT/OFFSET + collection case, which still degrades to preload
because `_materializeLimitedIds` / `_distinctSelectForLimitedIds`
(relation.ts:~5205-5250) project the primary key as a SINGLE column — a composite
PK would emit a bogus `cpk_orders."shop_id,id"`. The sibling story
`composite-pk-distinct-relation-materialization` was closed as "unreachable /
dead code" on the OLD premise that composite-PK joins always degrade to preload;
that premise no longer holds, so the limited-ids path is now the live residual.

Rails' `distinct_relation_for_primary_key`
(`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:463-488`)
uses `columns_for_distinct` over `Array(primary_key)`, projecting each key column
and re-filtering with a composite predicate. relation.ts:3597 already raises for
the pluck limit/offset composite case and points at this deviation.

## Acceptance criteria

- [ ] `_distinctSelectForLimitedIds` projects each column of a composite PK
      (mirroring `columns_for_distinct`); `_materializeLimitedIds` returns key
      tuples re-filtered via a composite `IN (tuples)` predicate.
- [ ] The pluck limit/offset composite path (relation.ts:~3582) uses it instead
      of raising through `applyJoinDependency()`.
- [ ] The `hasLimitOrOffset && !limitable` composite carve-out is removed from
      `_eagerLoadBypassesJoinDependency`; `EagerAssociationTest > preloading
has_many with cpk` JOINs (green).
- [ ] No regression in eager, CPK-eager, pluck, and through suites.
