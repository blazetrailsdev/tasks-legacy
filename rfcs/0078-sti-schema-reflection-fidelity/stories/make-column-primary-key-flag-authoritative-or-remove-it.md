---
title: "make-column-primary-key-flag-authoritative-or-remove-it"
status: ready
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
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

`Column#primaryKey` (`packages/activerecord/src/connection-adapters/column.ts`)
has no Rails counterpart: Rails' `Column` carries no `@primary_key` ivar, and
primary-key membership lives in `SchemaCache`'s own `@primary_keys` slot
(`schema_cache.rb:416`), which `marshal_dump` carries separately.

Because trails' Column carries it, `Column#encodeWith` has to write an eighth
key beyond Rails' seven (`column.rb:55-63`). PR #6980 tried to drop that key and
derive the flag from `_primaryKeys` on load instead; it reds on CI, because the
flag is **adapter-dependent** and measured so on all three lanes:

| adapter          | `columns()` reflects `primaryKey` for a real PK              |
| ---------------- | ------------------------------------------------------------ |
| sqlite3          | `true`                                                       |
| postgresql       | `false` (verified: raw adapter returns false for `posts.id`) |
| mysql2 / MariaDB | `false` (documented at `reconcilePrimaryKeyFlags`)           |

So a dump-loaded cache must reproduce whichever answer the reflecting adapter
gave, or `base_test.rb`'s `test_clear_cache!` reds — it compares a dump-loaded
cache against a reflected one, where Rails only ever compares
reflected-vs-reflected:

- derive from `_primaryKeys` → `true` on every lane → reds PG + MariaDB
  (measured, run 32735888743)
- omit entirely → `false` on every lane → reds sqlite3 (`test_clear_cache!`
  plus the schema dumper, which reads `col.primaryKey` at
  `schema-dumper.ts:316,973`)

`reconcilePrimaryKeyFlags` (`schema-cache.ts`) is clear-only for the same
reason: it can drop MySQL's promoted-unique false positive but must not invent
a `true` the adapter did not report.

## Converged shape

Either:

1. **Make the flag authoritative on both paths.** `setColumns` derives it from
   `_primaryKeys` rather than reconciling against it, so every adapter agrees
   on `true` for a real PK. Needs `columns()`'s cache-miss path to warm
   `_primaryKeys` first — today it does not (verified: after `cache.clear()`,
   `getCachedPrimaryKeys("posts")` is `undefined` while `columns()` repopulates
   `_columns`), so this changes the query profile and must be checked against
   the "add() issues four queries per table" budget in
   `test-fixtures/with-transactional-fixtures.ts`.
2. **Remove `Column#primaryKey` entirely** and route its two readers
   (`schema-dumper.ts:316`, `:973`) through the cache's `primaryKeys`, which is
   where Rails keeps it. This is the closer mirror; `resolvePrimaryKeyColumns`
   is already documented as consulting the authoritative key for dialects whose
   per-column flag over-reports, so the seam exists.

Either way `Column#encodeWith` then writes exactly Rails' seven keys and the
`primary_key` deviation note at `column.ts` goes away.

## Acceptance criteria

- [ ] `Column#encodeWith` / `initWith` carry exactly Rails' seven keys
      (`column.rb:46-63`); the `primary_key` key and its deviation note are gone.
- [ ] `base_test.rb` `test_clear_cache!` passes on sqlite3, postgresql AND
      mysql2 — all three, since each lane fails a different way here.
- [ ] `SchemaDumperTest` passes on all three lanes.
- [ ] If option 1: the per-table query count for `SchemaCache#add` does not grow.
