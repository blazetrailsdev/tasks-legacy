---
title: "Converge Column JSON serialization onto encode_with/init_with"
status: in-progress
updated: 2026-08-24
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 6980
claim: "2026-08-24T12:51:22Z"
assignee: "converge-schema-cache-install-onto-cache-replacement"
blocked-by: null
closed-reason: null
---

## Context

`Column` serialization in trails is a bespoke `toJSON()` / `static fromJSON()`
pair plus a `__mysql` / `__postgresql` / `__sqlite3` discriminator that
`rehydrateColumn` dispatches on
(`packages/activerecord/src/connection-adapters/column.ts:133-159`,
`mysql/column.ts:96-140`, `postgresql/column.ts`, `sqlite3/column.ts`,
`connection-adapters/schema-cache.ts:35-75`). PR #6972 added the PG and sqlite3
halves — the schema-cache dump it introduced was silently losing every
subclass field (`array`, `serial`, `oid`, `identity`, `generated`,
`autoIncrement`, `rowid`, `generatedType`), which `base_test.rb`'s
`test_clear_cache!` caught.

Rails' seam is `Column#encode_with(coder)` / `Column#init_with(coder)`
(`activerecord/lib/active_record/connection_adapters/column.rb:46-63`), the
YAML/Marshal protocol `SchemaCache#dump_to` goes through
(`schema_cache.rb:406`). Two divergences:

1. **Names.** `encodeWith` / `initWith` is what the conventions table produces
   from those Ruby names; trails spells them `toJSON` / `fromJSON`, which is
   why both are flagged by `parity:api:extra` and carry
   `@noRailsEquivalent PERMANENT` tags. `SchemaCache` itself already has
   `encodeWith` / `initWith` (`schema-cache.ts:194`, `:218`), so the
   inconsistency is inside one cluster.
2. **Key set.** Rails' `encode_with` writes exactly seven keys — `name`,
   `sql_type_metadata`, `null`, `default`, `default_function`, `collation`,
   `comment` — and _no subclass state_: `PostgreSQL::Column#array` and
   friends are NOT persisted, because YAML restores the class from its tag and
   `init_with` leaves the extra ivars nil. trails' JSON dump has no class tag,
   so it had to carry the discriminator, and #6972 chose to carry the subclass
   state as well rather than reproduce Rails' data loss.

The second point needs a decision, not a mechanical rename: reproducing Rails'
key set exactly would reintroduce the `test_clear_cache!` failure, because
trails' fixtures warm compares a dump-loaded cache against a reflected one
where Rails only ever compares reflected-vs-reflected.

## Acceptance criteria

- `Column#encodeWith` / `Column#initWith` replace `toJSON` / `fromJSON` on the
  base class and all three subclasses, at Rails' names and argument shape
  (a `coder` record), with `rehydrateColumn` / `serializeColumn` updated.
- The `@noRailsEquivalent` tags on the three subclass pairs go away with them.
- The class-discriminator question is settled explicitly in the code: either a
  `coder`-carried tag justified as the JSON stand-in for YAML's class tag, or a
  registry keyed off the adapter — cited at the call site either way.
- Whether the subclass state stays in the dump is decided with the
  `test_clear_cache!` constraint written down; if it stays, that is the one
  documented deviation and it cites this story.
- `pnpm parity:api:extra --package activerecord` shows the three pairs gone.
