---
title: "Hoist ensureSchemaLoaded and deferred distinct-PK materialization out of ids/pluck/calculation bodies"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 250
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `ids` onto `with_connection` (PR #6570, RFC 0106).

`Relation#ids` (`packages/activerecord/src/relation.ts`) opens its
`with_connection` block with two awaits that Rails' body
(`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:388-401`)
has no counterpart for:

```ts
await this._model.ensureSchemaLoaded();
await relation._materializeDeferredDistinctPkPredicates();
```

Rails needs neither: Ruby resolves a model's columns lazily through
`method_missing` on the connection it already holds, and Rails materializes the
`distinct_relation_for_primary_key` subquery at `.where()`-build time
(`finder_methods.rb:463`) rather than at query time. PR #6570 moved both inside
the lease so they no longer flip it permanent, but they are still two extra
calls in a ported body, and the same pair recurs in `pluck`'s arm and in
`inQueryConnection` ([[retire-in-query-connection-calculation-decorator]]).

## Converged shape

Hoist both out of the ported bodies:

- `ensureSchemaLoaded` — the schema reflection belongs at the point trails
  already reflects for every other read path, not inside each calculation body.
  Check whether the query path can reach `loadSchema` through the connection it
  leases (`model-schema.ts` `loadSchemaFromAdapter`, whose `reflectionAdapter`
  already prefers `threadedConnectionFor`) so the ported bodies need no await.
- `_materializeDeferredDistinctPkPredicates` — materialize at `.where()`-build
  time, which is where Rails does it, so no ported body carries the call. The
  existing comment at `relation/calculations.ts`' decorator says exactly this
  ("Rails materializes these at `.where()`-build time") and then does it at
  query time anyway.

Both are cross-cutting, so size the story around `ids` + `pluck` +
`inQueryConnection` and file any further callers as their own rows rather than
widening this one.

## Acceptance criteria

- [ ] `ids`' body has no `ensureSchemaLoaded` / materialization await; it reads
      as `calculations.rb:388-401` line for line.
- [ ] `pluck` and the calculation entry points lose the same pair.
- [ ] Deferred distinct-PK predicates still resolve to a literal id list before
      any arel compiles (the `where with eager-loading limited collection
relation subquery materializes distinct primary keys at load time` test
      in `relation.trails.test.ts` stays green).
- [ ] A first query against a never-reflected model still reflects, and still
      does not permanently lease under
      `permanent_connection_checkout = :deprecated | :disallowed` (the
      `common APIs don't permanently hold a connection...` assertions in
      `connection-handling.test.ts` cover `ids` and `pluck`).

## Absorbed: `converge-ids-pluck-third-arm-schema-and-deferred-pk-preludes`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Drop the ensureSchemaLoaded / deferred-distinct-PK preludes from ids' and pluck's read arms"

### Context

`Relation#ids` (`packages/activerecord/src/relation.ts`) was ported arm for arm
from `activerecord/lib/active_record/relation/calculations.rb:371-405` in PR
6565, but its third arm is wrapped in two preliminaries the Rails body has no
counterpart for:

```ts
return this._withQueryConnection(async () => {
  await (this._model as unknown as { ensureSchemaLoaded(): Promise<void> }).ensureSchemaLoaded();
  await this._materializeDeferredDistinctPkPredicates();
  const columns = this.arelColumns(primaryKeyArray);
  ...
});
```

Rails' third arm is just `arel_columns` → `spawn` → `select_values =` →
`where_clause.contradiction?` → `skip_query_cache_if_necessary { model.with_connection { c.select_all(...) } }`
(`:394-405`). Nothing loads the schema and nothing materializes a deferred
predicate, because Ruby resolves `model.attribute_types` lazily on first read and
Rails has no deferred distinct-PK marker at all.

Both preliminaries were inherited from the `pluck` delegation `ids` replaced —
`pluck` carries the identical pair (`relation.ts`, `_pluckInner`) — so this is one
shape in two bodies, not an `ids`-only wart. They were kept in #6565 deliberately:
dropping them silently regresses a sibling's cases, since `type_cast_pluck_values`
reads `model.attribute_types` and a deferred distinct-PK marker must resolve to a
literal id list before the arel compiles.

`_withQueryConnection` itself is already owned by
`converge-with-query-connection-onto-with-connection` (RFC 0106, in-progress) —
this story is the other two.

### Converged shape

`ensureSchemaLoaded()` and `_materializeDeferredDistinctPkPredicates()` stop being
per-terminal preludes:

- schema reflection resolves where Rails resolves it (on the attribute-type read
  inside `type_cast_pluck_values`), so no read terminal has to pre-load it; and
- the deferred distinct-PK marker resolves at `where()`-build time as Rails
  materializes it (`finder_methods.rb:463`), not at each terminal that might
  compile it — see the sibling
  `route-apply-join-dependency-through-distinct-relation-for-primary-key`.

Then `ids`' third arm is `calculations.rb:394-405` with no wrapper, and `pluck`'s
tail loses the same two lines.

### Acceptance criteria

- [ ] `Relation#ids`' third arm has no `ensureSchemaLoaded` /
      `_materializeDeferredDistinctPkPredicates` prelude.
- [ ] `Relation#pluck` loses the same pair, or the story documents why one
      terminal still needs it.
- [ ] `calculations.test.ts` `ids*` and `pluck*` tests stay green on all three
      adapters, including the deferred distinct-PK cases.
- [ ] `pnpm parity:api:calls` green; no new baseline rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
