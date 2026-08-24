---
title: "Converge dump installation onto Rails' cache replacement"
status: ready
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails installs a loaded schema-cache dump by **replacing the cache object**:
`SchemaReflection#load_schema` assigns `@cache = load_cache(pool)`, and
`load_cache` returns the freshly built `SchemaCache`
(`activerecord/lib/active_record/connection_adapters/schema_cache.rb:116-140`).
There is no "load into an existing cache" path in Rails at all — `marshal_load`
exists only for `Marshal.load` to call on a newly allocated object
(`schema_cache.rb:420`).

trails has no equivalent seam exposed to a caller holding an adapter. PR #6972's
fixtures warm therefore does the opposite: it calls
`sc.marshalLoad(dumped.marshalDump())` on the cache the pool already handed out
(`packages/activerecord/src/test-fixtures/with-transactional-fixtures.ts`,
`replaySchemaCacheDump`), because trails' adapter-side consumers hold
`internalSchemaCache` by identity — `AbstractAdapter#schemaCache` and
`TypeCaster::Connection` read the instance, so swapping `poolConfig.schemaCache`
underneath them is only safe where `ConnectionPool` does it itself
(`connection-adapters/abstract/connection-pool.ts`, the lazy-load and
eager-warm blocks, which both assign `this.poolConfig.schemaCache = loaded`).

The deviation is cited at that call site. It is debt: the marshal round-trip is
pure overhead (serialize a cache we hold, rehydrate it into another), and the
two installation paths — the pool's assignment and the fixtures warm's
marshal-load — should be one.

## Acceptance criteria

- One way to install a loaded `SchemaCache` into a pool, at Rails' shape
  (replace the object, `schema_cache.rb:116-140`), reachable by a caller that
  holds an adapter rather than the pool internals.
- `replaySchemaCacheDump` uses it and drops the `marshalDump()`/`marshalLoad`
  round-trip and its call-site deviation note.
- Identity-holding consumers (`AbstractAdapter#schemaCache`,
  `TypeCaster::Connection`) still see the loaded cache — the reason the
  deviation was taken in the first place, so this is the part to prove, not
  assume.
- `packages/activerecord/src/support/schema-cache-dump.trails.test.ts` still
  passes on all three lanes.
