---
title: "PG Column#generated and SQLite3 Column#autoIncrement are public where Rails keeps ivars behind predicates"
status: draft
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
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

Surfaced while landing `converge-column-subclass-coders-to-rails-per-subclass-key-sets`
(PR #7032). With the coder key sets converged, the remaining divergence in the
adapter `Column` subclasses is that trails exposes as **public fields** state
Rails keeps in ivars behind predicates.

Two concrete instances, both measured:

**1. `PostgreSQL::Column#generated` is public surface Rails does not have.**
`pnpm parity:api:extra --package activerecord` reports
`connection-adapters/postgresql/column.ts — 1 novel`, and the novel name is
`generated`. In Rails, `@generated` is written by `initialize` / `init_with`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/column.rb:9-13`,
`:50-55`) and read by exactly one method, `virtual?` (`:27-30`):

```ruby
def virtual?
  # We assume every generated column is virtual, no matter the concrete type
  @generated.present?
end
```

There is no `generated` reader. `serial` and `identity` do not show up as novel
because `serial?` / `identity?` exist (`:16-22`); `generated` has no such
counterpart, so the public field is invented surface.

**2. `SQLite3::Column` has no `auto_increment?`, and exposes `autoIncrement`
instead.** Rails defines the predicate
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/column.rb:18-20`):

```ruby
def auto_increment?
  @auto_increment
end
```

and only `rowid` gets a reader (`attr_reader :rowid`, `:7`). trails'
`packages/activerecord/src/connection-adapters/sqlite3/column.ts` has a public
`autoIncrement` field and no `isAutoIncrement()`. Callers read the field
directly — `packages/activerecord/src/schema-dumper.ts` (the `autoIncrement`
projection) and `packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`'s
sqlite3 paths — which is why the MySQL half of that projection needed a
`typeof col.isAutoIncrement === "function"` fallback in PR #7032: the two
subclasses disagree about whether the flag is a field or a predicate.

Note the MySQL subclass is already converged here — PR #7032 removed its
`unsigned` / `autoIncrement` / `virtual` fields entirely and derives all three
(`mysql/column.rb:9-24`), so this story is only about PG and sqlite3.

## Converged shape

- `postgresql/column.ts`: `generated` becomes a non-public ivar (`_generated`,
  matching the file's existing `_generatedType` spelling in the sqlite3
  sibling), written by the constructor and `initWith`, read by `isVirtual()`
  and `encodeWith`. `parity:api:extra` for that file goes to 0 novel.
- `sqlite3/column.ts`: add `isAutoIncrement()` per `sqlite3/column.rb:18-20`,
  make the ivar non-public, and repoint every reader at the predicate.
  `isAutoIncrementedByDb()` (`:22-24`) then reads `auto_increment? || rowid`
  through the predicate as Rails does. `rowid` stays a public reader
  (`attr_reader :rowid`, `:7`).
- Once both subclasses answer `isAutoIncrement()`, the duck-typed
  `typeof col.isAutoIncrement === "function"` fallback in
  `schema-dumper.ts`'s `autoIncrement` projection can collapse to a plain call.
  (The wider projection layer is its own story —
  `adapter-schema-source-column-flag-duck-typing` under 0023 — so only that one
  branch is in scope here.)

## Acceptance criteria

- `pnpm parity:api:extra --package activerecord` reports
  `connection-adapters/postgresql/column.ts` at 0 novel, and the
  `extra-surface-mark.json` novel/total marks are TIGHTENED (never raised).
- `SQLite3::Column#isAutoIncrement()` exists and mirrors
  `sqlite3/column.rb:18-20`; no caller reads a public `autoIncrement` field.
- The `autoIncrement` branch of `schema-dumper.ts`'s projection no longer
  duck-types on `typeof … === "function"`.
- `pnpm parity:api:calls` / `:args` clean; SQLite, PostgreSQL and
  MySQL/MariaDB lanes green.
