---
title: "Column coder carries a class tag and subclass state where Rails' writes seven keys"
status: ready
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Column#encodeWith` / `initWith` (`packages/activerecord/src/connection-adapters/column.ts`)
now carry Rails' exact seven keys (`vendor/rails/activerecord/lib/active_record/connection_adapters/column.rb:46-63`),
but two deviations remain around them, both cited at
`Column#encodeWith` and both rooted in the same cause — trails' schema-cache
dump is **JSON**, where Rails' `SchemaCache#dump_to` writes YAML
(`vendor/rails/activerecord/lib/active_record/connection_adapters/schema_cache.rb:406`):

1. **A `class` key in the coder.** YAML restores the adapter's Column subclass
   from the document's `!ruby/object:` tag; JSON has no tag, so
   `Column#encodeWith` writes `coder["class"] = "PostgreSQL::Column"` and
   `rehydrateColumn` (`connection-adapters/schema-cache.ts`) dispatches on it
   through a `COLUMN_CLASSES` map. Rails writes no such key.

2. **Subclass state in the coder.** Rails' `encode_with` writes seven keys and
   _no subclass ivars_ — `PostgreSQL::Column#array`, `#serial`, `#oid`,
   `SQLite3::Column#rowid`, `MySQL::Column#extra` & co. are simply lost, and
   `init_with` leaves them nil. trails' three subclasses override
   `encodeWith` / `initWith` to carry theirs.

Rails' data loss is invisible to Rails because it only ever compares a
reflected cache against another reflected one. trails' fixtures warm installs a
**dump-loaded** cache, so `base_test.rb`'s `test_clear_cache!`
(`packages/activerecord/src/base.test.ts`, "clear cache!") compares
dump-loaded against reflected — measured on CI in PR #6980: dropping the
subclass state reds it, because a dump-loaded PG column reports `array`/`serial`
undefined where the reflected one has real values.

## Converged shape

`Column#encodeWith` / `initWith` exist only on the base class, writing and
reading Rails' seven keys; the three subclass overrides and the `class` key are
gone. That needs both halves:

- the dump restores the Column subclass without a coder key — either a
  format that carries class identity natively, or a registry keyed off the
  **adapter** doing the loading (the adapter already knows which Column class
  it builds), which is the alternative the original story listed
- the dump-loaded-vs-reflected comparison is eliminated, so Rails' subclass
  data loss is as invisible here as it is upstream — e.g. the fixtures warm
  does not install a dump for the cache `test_clear_cache!` reads, matching
  Rails, where that test only ever sees reflected columns

## Acceptance criteria

- [ ] `Column#encodeWith` / `initWith` write and read exactly the seven keys at
      `column.rb:46-63`; no `class` key.
- [ ] `mysql/column.ts`, `postgresql/column.ts`, `sqlite3/column.ts` carry no
      `encodeWith` / `initWith` override.
- [ ] `rehydrateColumn` / `COLUMN_CLASSES` in `schema-cache.ts` are gone or
      reduced to an adapter-keyed lookup with no coder participation.
- [ ] `base_test.rb` "clear cache!" passes on sqlite3, postgresql AND mysql2 —
      each lane fails this differently, so all three must be run.
- [ ] `packages/activerecord/src/support/schema-cache-dump.trails.test.ts` passes.
