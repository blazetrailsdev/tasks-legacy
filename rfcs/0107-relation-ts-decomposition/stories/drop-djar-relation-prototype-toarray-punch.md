---
title: "DJAR#execQueries punches Relation.prototype.toArray where the plain chain now suffices"
status: done
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: 6949
claim: "2026-08-23T21:02:49Z"
assignee: "drop-djar-relation-prototype-toarray-punch"
blocked-by: null
closed-reason: null
---

## Context

`DisableJoinsAssociationRelation#execQueries`
(`packages/activerecord/src/disable-joins-association-relation.ts:584`) calls
`Relation.prototype.toArray.call(merged)` to load the composed chained-state
relation while bypassing DJAR's own override chain.

That explicit prototype call was meaningful only while `Relation#toArray`
carried the query itself. PR #6943 inverted the chain onto Rails' direction
(`to_ary` is `records.dup`, relation.rb:337-339; the query and assignment live
in `load`, relation.rb:1179-1186), so `Relation.prototype.toArray.call(merged)`
now just routes to `merged.records()` -> `merged.load()` and dispatches to
DJAR's `load` override anyway — identical to the plain `merged.toArray()` on
the line below it (`:589`). The indirection is residue with no remaining
effect.

Rails has no such prototype-punching: `disable_joins_association_relation.rb`
overrides `load` and lets the ordinary chain run.

## Converged shape

Replace `(await Relation.prototype.toArray.call(merged)) as T[]` with
`await merged.toArray()`, confirming with the disable-joins suites
(`packages/activerecord/src/associations/disable-joins-*.trails.test.ts`,
`has-many-through-disable-joins-associations.test.ts`) that the in-memory
limit/offset slice still applies after the id-order regroup.

## Acceptance criteria

- No `Relation.prototype.toArray.call` remains in
  `disable-joins-association-relation.ts`.
- All disable-joins + collection-proxy suites stay green.
- `pnpm parity:api:calls` / `:args` add zero rows.
