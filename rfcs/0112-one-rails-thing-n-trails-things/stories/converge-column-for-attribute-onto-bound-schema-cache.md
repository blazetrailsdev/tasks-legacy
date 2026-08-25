---
title: "Converge columnForAttribute onto the bound schema cache"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 60
pr: null
claim: "2026-08-25T14:26:33Z"
assignee: "consolidate-three-assign-attributes-implementations"
blocked-by: null
closed-reason: null
---

## Context

Left over from PR #5906 (story
`converge-schema-cache-getter-onto-bound-reflection`), which converged
`AbstractAdapter#schemaCache` onto the pool-bound `BoundSchemaReflection` and
moved the DDL cache-busts, `tableExists`, `insertAll` and the uniqueness
validator onto Rails' one-arg forms.

`AbstractAdapter#columnForAttribute`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`, the
`columnForAttribute` getter) was NOT moved. It still reads the raw cache
through the two-arg form, and builds a `FakePool` by hand to do it:

```ts
const pool = this.pool == null || this.pool instanceof NullPool ? new FakePool(this) : this.pool;
const hash = await (this.internalSchemaCache as any).columnsHash(pool, tableName);
```

Rails is `schema_cache.columns_hash(table_name)` — one arg, no pool plumbing,
because `BoundSchemaReflection` already carries the pool (and
`for_lone_connection` already wraps a `FakePool` for the NullPool case, so the
hand-rolled branch above is re-implementing what the bound handle does).

## Acceptance criteria

- `columnForAttribute` reads `this.schemaCache.columnsHash(tableName)`.
- The hand-built `FakePool` branch and the `as any` cast go away.
- No new `internalSchemaCache` readers; the count of raw-cache call sites in
  `abstract-adapter.ts` drops to zero.
- `pnpm parity:api:calls` and `pnpm parity:api:calls` stay green (this is one of the
  call sites the wide ratchet scores).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
