---
title: "converge-pg-type-map-initializer-onto-ported-type-map"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

`packages/activerecord/src/connection-adapters/postgresql/oid/type-map-initializer.ts:12`
declares a local `interface TypeMap` (oid-keyed `registerType` / `aliasType` /
`lookup`) as the store `TypeMapInitializer` writes into. Rails passes a real
`ActiveRecord::Type::HashLookupTypeMap`
(`vendor/rails/activerecord/lib/active_record/type/hash_lookup_type_map.rb`,
subclass of `Type::TypeMap` in `type/type_map.rb:7`).

Trails ports `Type::TypeMap` at `packages/activerecord/src/type/type-map.ts`,
but it is keyed `string | RegExp` and returns `Type`, so the PG initializer
cannot type against it as-is; `HashLookupTypeMap` — the oid-keyed subclass this
call site actually needs — is not ported.

Found by the RFC 0080 audit of `moved` interface declaration names
(`audit-moved-interface-declaration-names`), which tagged it
`@noRailsEquivalent CONVERGEABLE (story: <this story>)`.

## Acceptance criteria

- `ActiveRecord::Type::HashLookupTypeMap` is ported in Rails layout
  (`type/hash-lookup-type-map.ts`).
- `type-map-initializer.ts` types its store on the ported class and the local
  `TypeMap` interface plus its `@noRailsEquivalent` tag are deleted.
- `pnpm parity:api:extra` exits 0 (no stale tag).
