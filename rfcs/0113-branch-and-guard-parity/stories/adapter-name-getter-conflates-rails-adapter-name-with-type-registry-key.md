---
title: "adapterName conflates Rails' ADAPTER_NAME with the type-registry key"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the mysql2 type-registration key (#6496, RFC 0099
story `converge-mysql2-type-registration-adapter-key`).

trails' concrete adapters overload ONE getter, `adapterName`, with two
distinct Rails concepts:

1. Rails' `AbstractAdapter#adapter_name`
   (`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:353`)
   returns the class's `ADAPTER_NAME` constant — a human-facing display name:
   `"PostgreSQL"` (`postgresql_adapter.rb:54`), `"Mysql2"`
   (`mysql2_adapter.rb:19`), `"SQLite"` (`sqlite3_adapter.rb:31`),
   `"Trilogy"` (`trilogy_adapter.rb:17`), `"Abstract"`
   (`abstract_adapter.rb:31`). It is what Rails interpolates into
   `NotImplementedError` messages and what `current_adapter?` matches on.
2. The type-registry key, which Rails derives SEPARATELY via
   `ActiveRecord::Type.adapter_name_from`
   (`vendor/rails/activerecord/lib/active_record/type.rb:49-51`) =
   `model.connection_db_config.adapter.to_sym` — the CONFIG string
   (`:mysql2`, `:postgresql`, `:sqlite3`), never `ADAPTER_NAME`.

trails collapses both onto `adapterName`, typed `AdapterName`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts:139`):
`AbstractMysqlAdapter#adapterName` returns `"mysql2"`
(`connection-adapters/abstract-mysql-adapter.ts:251`), `SQLite3Adapter`
returns `"sqlite"` (`sqlite3-adapter.ts:203`), `PostgreSQLAdapter` returns
`"postgres"` (`postgresql-adapter.ts:204`) — none of which is the Rails
`ADAPTER_NAME`. Only the base class is faithful (`"Abstract"`).

The visible consequence today is the `NotImplementedError` text in
`abstract/schema-statements.ts:1882` / `:1892`, which interpolates
`this.adapterName` and so reads `mysql2 does not support changing table
comments` where Rails reads `Mysql2 does not support ...`. The wider cost is
that ~20 call sites branch on `adapterName` as a dialect discriminator, so
the display name cannot be fixed without giving those sites the config-derived
key they actually want.

## Converged shape

- `adapterName` returns Rails' `ADAPTER_NAME` verbatim: `"Mysql2"`,
  `"PostgreSQL"`, `"SQLite"`, `"Trilogy"`, `"Abstract"`. Add the
  `ADAPTER_NAME` static to each concrete adapter and have the getter read it,
  as `abstract_adapter.rb:353` does.
- Every dialect branch that today reads `adapterName` reads the config-derived
  registry key instead — the same value `Type.adapterNameFrom` /
  `adapterNameFromConfig` produce. Call sites to move:
  `insert-all.ts`, `abstract/schema-statements.ts` (3 switches + 2 guards),
  `model-schema.ts`, `migration.ts` (`_adapterName`), `fixtures.ts`
  (`adapterName()`), `abstract/schema-definitions.ts` +
  `mysql/schema-definitions.ts` (`TableDefinition`), `abstract/
native-database-types.ts`, `support/{schema-file-generator,canonical-schema,
canonical-table-rebuild,drop-all-tables,load-schema-helper}.ts`,
  `test-helpers/test-schema.ts`, and `trailties/src/schema-source.ts`
  (`detectAdapter` substring-matches `adapterName.toLowerCase()`).
- The `NotImplementedError` strings then match Rails without a separate fix.

Depends on nothing, but overlaps
`type-adapter-name-normalization-collapses-rails-adapter-spellings`
(0023) — that story converges the `postgres`/`sqlite` arms of
`adapterNameFromConfig` to Rails' config spellings; this one separates the
display name from the key. Sequencing them the other way round is fine.

## Acceptance criteria

1. Each concrete adapter has an `ADAPTER_NAME` static matching Rails
   verbatim, and `adapterName` returns it.
2. No dialect branch reads `adapterName`; they read the config-derived
   registry key.
3. `NotImplementedError` messages from `schema-statements.ts:1882`/`:1892`
   match Rails' text.
4. `pnpm parity:api:calls`, `pnpm parity:api:calls:args` green; all three
   adapter lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
