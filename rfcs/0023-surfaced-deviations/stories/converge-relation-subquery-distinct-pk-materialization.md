---
title: "converge-relation-subquery-distinct-pk-materialization"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`relation-handler-distinct-pk-materialization` landed as #3383 (RFC 0022, now
closed), but the gap is still open at
`packages/activerecord/src/relation.ts:4719-4746`, which cites that landed story
as "the continuation story".

The site is the relation-as-subquery-value path. Rails' `apply_join_dependency`,
for a limit/offset over non-limitable (collection) reflections, replaces the
relation with `distinct_relation_for_primary_key` (`finder_methods.rb:463`):
it EXECUTES a query (`select_rows`, `schema_statements.rb:1434`) to materialize
the limited DISTINCT primary keys — honouring `columns_for_distinct(...,
order_values)` — rewrites the relation as `WHERE pk IN (ids)`, and clears
`limit_value` / `offset_value`. This avoids `IN (SELECT … LIMIT n)`, which
limits joined rows rather than parents and is unportable (MySQL rejects it).

trails cannot do that here because the predicate builder is synchronous, so
the branch throws `NotImplementedError` with a "materialize the ids first"
message and carries an `@nie disposition=TODO` marker.

This is the sibling of the composite-PK case in
`converge-composite-pk-distinct-relation-materialization`; the blocker here is
the sync predicate builder rather than composite-PK support, so they converge
separately.

## Acceptance criteria

- A relation with an eager-load + limit/offset over a collection association
  works as a subquery value, materializing the limited DISTINCT primary keys as
  Rails does, rather than raising `NotImplementedError`.
- The `@nie disposition=TODO` marker and the stale
  `relation-handler-distinct-pk-materialization` citation at relation.ts:4724
  are removed.
- `pnpm parity:test` delta non-negative.

## Absorbed: `converge-eager-tosql-distinct-pk-subquery-into-apply-join-dependency`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Converge or record the toSql eager distinct-pk inline subquery deviation"

### Context

Surfaced while converging the live eager path onto `build_arel` (#5909).

`_applyEagerJoinDependency` (`packages/activerecord/src/relation.ts`) now mirrors
Rails `apply_join_dependency` (`finder_methods.rb:456-488`), including the
limit/offset rewrite for non-limitable reflections. Rails resolves that case by
EXECUTING `distinct_relation_for_primary_key`
(`finder_methods.rb:463`, `schema_statements.rb:1429-1452`) and rewriting the
relation as `pk IN (ids)` with limit/offset cleared.

trails does exactly that on the async path (`_materializeLimitedIds`, fed in as
`limitedIds`). The SYNCHRONOUS `toSql` path cannot execute a query, so it nests
the same DISTINCT-pk query inline as a subquery — `pk IN (SELECT DISTINCT pk …
LIMIT n)` — via `_buildEagerIdSubquery`. The relation-rewrite shape matches Rails
but the operand does not: Rails never emits a nested LIMIT subquery here, and
MariaDB rejects `IN (SELECT … LIMIT n)` in some positions
(ER_NOT_SUPPORTED_YET), which is why the async path materializes instead.

This is the THIRD site implementing Rails' single `distinct_relation_for_primary_key`.
The other two are tracked by
`converge-eager-count-distinct-pk-materialization-into-apply-join-dependency`
(calculations' inline version) and `relation-handler-distinct-pk-materialization`
(the predicate-builder throw). Converging all three means giving
`applyJoinDependency` one async materialization path; this story covers the
`toSql`/`_buildEagerSql` caller, which is the one with no async context at all
and so may need `toSql` to stay approximate by construction — in which case the
deviation should be recorded as PERMANENT with a `@noRailsEquivalent`-style
justification rather than silently carried.

### Acceptance criteria

- Either `_buildEagerSql`'s eager limit/offset path routes through the same
  materialization the async path uses, or the inline-subquery operand is
  documented as a permanent, structurally-forced deviation with the Rails
  `file:line` it approximates.
- `_buildEagerIdSubquery` has exactly one remaining caller shape, or is removed.
- Coordinate with the two sibling stories above so the three sites converge on
  one implementation rather than three.
