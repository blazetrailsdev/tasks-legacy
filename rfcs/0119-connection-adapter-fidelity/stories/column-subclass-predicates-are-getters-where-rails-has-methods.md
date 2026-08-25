---
title: "PG Column predicates are getters where Rails (and the sqlite3 sibling) have methods"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Column subclass predicates are getters where the sqlite3 sibling is a method

## Context

Filed under RFC 0119 rather than RFC 0096 (where the originating story lived):
0096 is closed and will not accept a draft story.

Surfaced while landing
`pg-column-serial-identity-fields-are-public-where-rails-has-ivars` (PR #7061).
That story's own text flagged it and deferred it: "Note `isSerial` / `isIdentity`
are currently getters, not methods, unlike the `isAutoIncrement()` method
PR #7058 added on the sqlite3 sibling; decide one spelling for the pair while
converging." PR #7061 kept the getters, because every caller already reads them
as properties and flipping them was out of that story's scope — so the two
adapter Column subclasses now spell the same kind of Rails predicate two
different ways.

In Rails every one of these is an ordinary predicate METHOD:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/column.rb:16-22`
  — `identity?`, `serial?`
- `postgresql/column.rb:41` — `alias :array? :array`
- `postgresql/column.rb:43-45` — `enum?`
- `sqlite3/column.rb` — `auto_increment?`, which trails already ports as the
  method `isAutoIncrement()`.

trails today, `packages/activerecord/src/connection-adapters/postgresql/column.ts`:
`isSerial`, `isIdentity`, `isEnum` and `array` are `get` accessors, while
`isArray()` and `isAutoIncrementedByDb()` beside them are methods.

Note this is NOT the generated-attribute-reader case CLAUDE.md ratifies
("Generated attribute readers are properties"). That section is about
`define_method_attribute`-generated readers for user attributes; a hand-written
predicate on a Column subclass has no such constraint, and the sqlite3 sibling
already proves the method spelling works.

## Converged shape

One spelling across both subclasses, matching Rails' predicate methods:
`isSerial()`, `isIdentity()`, `isEnum()` alongside the existing `isArray()` /
`isAutoIncrement()` / `isAutoIncrementedByDb()`. `array` (the non-bang reader at
`postgresql/column.rb:37-39`) stays a property — Ruby's `array` is a reader and
only `array?` is the alias.

Repoint every caller. Known field-shaped reads to fix:

- `packages/activerecord/src/schema-dumper.ts:356` — `(col as any).isSerial === true`
- `packages/activerecord/src/connection-adapters/postgresql/schema-dumper.ts:123,129,156`
- `packages/activerecord/src/connection-adapters/abstract/schema-dumper.ts:140`
- `packages/activerecord/src/adapters/postgresql/serial.test.ts` (several)
- `packages/activerecord/src/primary-keys.test.ts:417,431`
- `packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.trails.test.ts:383`
- `packages/activerecord/src/adapters/postgresql/postgresql-adapter.trails.test.ts:955`
- `packages/activerecord/src/connection-adapters/postgresql/column.trails.test.ts:43`

Watch the `ColumnInfo` interface in `schema-dumper.ts:79`, which has its own
plain `isSerial?: boolean` DATA field — that one is a struct member, not a
predicate, and must not be converted.

## Acceptance criteria

- [ ] PG `Column`'s `isSerial` / `isIdentity` / `isEnum` are methods, matching
      `postgresql/column.rb:16-22,43-45` and the sqlite3 sibling's
      `isAutoIncrement()`.
- [ ] Every caller listed above calls rather than reads; `ColumnInfo.isSerial`
      stays a data field.
- [ ] parity:api / parity:test delta non-negative; `parity:api:extra:gate` not
      raised. SQLite, PostgreSQL and MySQL/MariaDB lanes green.
