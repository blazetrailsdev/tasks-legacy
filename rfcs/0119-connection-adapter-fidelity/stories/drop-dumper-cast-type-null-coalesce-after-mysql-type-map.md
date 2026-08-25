---
title: "Drop the schema-dumper cast-type null coalesce once the MySQL type-map lookup converges"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps:
  - mysql-native-type-map-converges-onto-type-map
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Both `AdapterSchemaSource` bridges coalesce a `null` cast-type lookup to
`Type.default_value`, so that `schemaDefault` can call `type.deserialize(...)`
unguarded the way Rails does:

- `packages/activerecord/src/schema-dumper.ts` (`AdapterSchemaSource.lookupCastTypeFromColumn`)
  — `?? defaultValue()` from `@blazetrails/activemodel`
- `packages/trailties/src/schema-source.ts` (`AdapterSchemaSource.lookupCastTypeFromColumn`)
  — `?? Type.defaultValue()` via activerecord's re-export

Rails has no counterpart for either coalesce.
`lookup_cast_type_from_column` (`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:125`)
bottoms out at `TypeMap#lookup`, whose miss block yields `Type.default_value`
(`vendor/rails/activemodel/lib/active_model/type.rb:38-40`) — it can never
return nil, so nothing downstream coalesces.

The coalesce exists solely to absorb one trails invention:
`AbstractMysqlAdapter#lookupCastTypeFromColumn`
(`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1282-1288`)
returns `null` for a blank `sqlType`. That adapter flags itself as divergent at
`abstract-mysql-adapter.ts:1268-1276` and names
`mysql-native-type-map-converges-onto-type-map` as its convergence owner.

Landed in #6875, which converged `schemaDefault` onto `schema_default`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_dumper.rb:87-95`).
Reviewer-surfaced: the first cut cast the adapter's result `as Type`, which let
`null` reach `deserialize`. The coalesce is the correct interim seam fix, but it
is debt, not the end state.

## Converged shape

Once `mysql-native-type-map-converges-onto-type-map` lands and the MySQL
override returns a `Type` for every input (falling through to the type map's
own default on a miss, as Rails does), both bridges lose the coalesce and the
`| null` in their casts:

```ts
lookupCastTypeFromColumn(column: ColumnInfo): Type {
  return this._adapter.lookupCastTypeFromColumn(column as { sqlType: string | null }) as Type;
}
```

The `SchemaSource.lookupCastTypeFromColumn` interface signature is already
non-nullable and stays as it is; only the implementations shrink.

## Acceptance criteria

- [ ] `mysql-native-type-map-converges-onto-type-map` has landed (this story is
      a no-op before that — verify the MySQL override can no longer return null).
- [ ] The `?? defaultValue()` coalesce is gone from
      `packages/activerecord/src/schema-dumper.ts`, along with the
      `@blazetrails/activemodel` `defaultValue` import if it has no other use.
- [ ] The `?? Type.defaultValue()` coalesce is gone from
      `packages/trailties/src/schema-source.ts`, along with the
      `import { Type } from "@blazetrails/activerecord"` if it has no other use.
- [ ] The Rails-cite comments justifying both coalesces are deleted with them.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; MySQL/MariaDB
      lane green (schema dumping over a blank-`sqlType` column is the case that
      motivated the null return).

## Notes

`packages/activerecord/src/schema-dumper.ts` must NOT import `./type.js` to
reach `defaultValue` — that edge closes an import cycle and a plain-node import
of the built `dist/schema-dumper.js` throws
`Cannot access 'BaseSchemaDumper' before initialization`. The vitest suite stays
green through it. Import from `@blazetrails/activemodel` (a leaf) instead, as
PR #6875 does. Relevant only if this story ends up moving the import rather than
deleting it.
