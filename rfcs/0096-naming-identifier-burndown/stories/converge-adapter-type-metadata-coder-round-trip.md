---
title: "Round-trip the PG/MySQL TypeMetadata through the schema-cache coder so the Column subclasses can delegate oid/fmod/extra"
status: in-progress
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: 7026
claim: "2026-08-25T09:46:54Z"
assignee: "missing-rails-call-tag-inert-on-non-rails-class-member"
blocked-by: null
closed-reason: null
---

## Context

Found while working #7014 (`converge-column-subclass-coders-to-rails-per-subclass-key-sets`
was released for exactly this reason).

Rails' adapter `Column` subclasses do NOT persist `oid` / `fmod` / `extra`
themselves — they delegate to `sql_type_metadata`
(`activerecord/lib/active_record/connection_adapters/postgresql/column.rb:7`,
`.../mysql/column.rb:7`), and the base `encode_with`
(`.../abstract/column.rb`, trails `connection-adapters/column.ts:180-190`)
persists `sql_type_metadata` for them. YAML tags that object with its class, so
`PostgreSQL::TypeMetadata` round-trips with its own ivars.

trails cannot do that yet:

- `connection-adapters/sql-type-metadata.ts:55,65` — `toJSON` / `fromJSON` carry
  the five base keys and reconstruct a bare `SqlTypeMetadata`; there is no class
  tag, so a PG or MySQL metadata object cannot be recovered.
- `connection-adapters/postgresql/type-metadata.ts` and
  `.../mysql/type-metadata.ts` do not extend `SqlTypeMetadata` and have no
  `toJSON` at all, so `Column#encodeWith`'s `this.sqlTypeMetadata?.toJSON()`
  would throw if a Column ever held one.
- Consequently `postgresql/column.ts:53-58` and `mysql/column.ts:52-56` store
  `oid` / `fmod` / `extra` as Column fields and encode them into the Column
  coder, which is the divergence the released story is about.

This is the prerequisite: until the metadata round-trips, the Column key sets
cannot shrink to Rails'.

## Acceptance criteria

1. `PostgreSQL::TypeMetadata` and `MySQL::TypeMetadata` round-trip through the
   schema-cache coder with their own state (`oid` / `fmod`;
   `extra`) — a class tag on the serialized `sql_type_metadata` payload plus
   dispatching reconstruction, mirroring how `rehydrateColumn`
   (`schema-cache.ts`) already dispatches the Column `class` tag.
2. `PostgreSQL::Column` / `MySQL::Column` hold that metadata object, so
   `oid` / `fmod` / `extra` are delegations (`postgresql/column.rb:7`,
   `mysql/column.rb:7`) rather than Column fields.
3. SQLite, PostgreSQL and MySQL/MariaDB lanes green.
4. `converge-column-subclass-coders-to-rails-per-subclass-key-sets` is
   unblocked: its key-set shrink becomes a pure deletion.
