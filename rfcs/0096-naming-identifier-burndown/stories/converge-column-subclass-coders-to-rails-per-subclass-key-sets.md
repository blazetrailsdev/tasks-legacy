---
title: "Converge the adapter Column coders to Rails' per-subclass key sets (delegate oid/fmod, derive MySQL extra)"
status: claimed
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: "2026-08-25T12:58:54Z"
assignee: "split-model-mixin-surface-to-active-model-model"
blocked-by: null
closed-reason: null
---

## Context

Re-specification of `converge-column-subclass-state-out-of-encode-with`, whose
premise ("Rails persists **no subclass state**") is contradicted by the vendored
source. Rails' adapter `Column` subclasses each override `encode_with` /
`init_with` and DO persist their own ivars:

- `postgresql/column.rb:50-61` — `serial`, `identity`, `generated`, then `super`.
  `oid` / `fmod` are NOT ivars: they `delegate … to: :sql_type_metadata` (`:7`),
  which the base `encode_with` already persists. `array` is derived —
  `sql_type_metadata.sql_type.end_with?("[]")` (`:37-39`).
- `sqlite3/column.rb:35-42` — `auto_increment` only, then `super`. `rowid` and
  `@generated_type` are genuinely dropped by a round-trip.
- `mysql/column.rb` — **no** `encode_with` / `init_with` at all. `unsigned?`,
  `case_sensitive?`, `auto_increment?` and `virtual?` (`:9-24`) all derive from
  `sql_type` / `collation` / `sql_type_metadata.extra`, so nothing needs
  persisting.

trails diverges by storing as Column fields what Rails derives or delegates:

- `postgresql/column.ts:53-58` stores `oid` / `fmod` on the Column and builds a
  plain `SqlTypeMetadata`, where Rails wraps `PostgreSQL::TypeMetadata`
  (`postgresql/type-metadata.ts` exists and already carries `oid` / `fmod`, but
  the Column does not use it). It also stores `array` rather than deriving it.
- `mysql/column.ts:52-56` stores `unsigned` / `autoIncrement` / `virtual` /
  `extra` as fields, where Rails derives every one; `mysql/type-metadata.ts`
  exists and carries `extra`.

So the coder key sets cannot converge until the delegation/derivation converges
— dropping the keys today makes those fields `undefined` on a dump-loaded
column. Measured on this branch: three sqlite3/PG predicates are ported as
`x !== null` where Ruby writes `nil?` / `present?`
(`sqlite3/column.ts:55` `isVirtual`, `postgresql/column.ts:90` `isIdentity`,
`:100` `isVirtual`), so an `undefined` ivar flips them TRUE, every dump-loaded
sqlite3 column reports `virtual?`, and `primary-keys.test.ts` reds with
`NOT NULL constraint failed: subscribers.nick` (4 cases).

## Acceptance criteria

- `postgresql/column.ts` wraps `PostgreSQL::TypeMetadata` and delegates `oid` /
  `fmod` to it (`postgresql/column.rb:7`); `array` derives from the metadata's
  unstripped `sql_type` (`:37-39`); `encodeWith` / `initWith` carry `serial`,
  `identity`, `generated` and call `super` last, matching `:50-61`.
- `mysql/column.ts` derives `unsigned` / `autoIncrement` / `virtual` / `extra`
  per `mysql/column.rb:7-24` and drops both coder overrides (the `class` tag
  aside).
- `sqlite3/column.ts` carries `auto_increment` only (`sqlite3/column.rb:36-44`);
  `rowid` / `generatedType` are dropped by a round-trip, as in Rails.
- Dropping `rowid` / `generated_type` is the one arm that is load-bearing in
  trails and not upstream: Rails only ever compares a reflected cache against
  another reflected one, while trails' fixtures warm
  (`test-fixtures/with-transactional-fixtures.ts`, `replaySchemaCacheDump`)
  installs the boot dump and `base_test.rb`'s `test_clear_cache!` then compares
  it against a reflected cache. Converge that comparison too, so the sqlite3
  key set can shrink without reddening it.
- The three `nil?` / `present?` predicates above are ported with Ruby nil
  semantics (`!= null`), so a nil ivar is falsy.
- The `Column#encodeWith` deviation comment
  (`connection-adapters/column.ts:153-172`) is rewritten to describe Rails'
  actual per-subclass key sets, and the three `*/column.trails.test.ts`
  round-trip cases pin them.
- SQLite, PostgreSQL and MySQL/MariaDB lanes green.
