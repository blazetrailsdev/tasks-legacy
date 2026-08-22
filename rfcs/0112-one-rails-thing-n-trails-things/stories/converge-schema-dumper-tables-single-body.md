---
title: "Collapse SchemaDumper#tables' duplicated sync fast path into Rails' single body"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 230
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing #6369. Rails' `SchemaDumper#tables` has ONE body
(`schema_dumper.rb:134-155`). The trails port has two: an async branch (taken
when `SchemaSource#tables()` returns a Promise, i.e. every real adapter) and a
synchronous fast path for mock sources, which re-implements the table walk
inline — reading `columns`/`indexes`/`fetchTableOptions`, setting `tableName`,
and calling `emitTable` directly instead of going through `table()`.

`packages/activerecord/src/schema-dumper.ts` `tables(stream)`:

- the async branch was converged by #6369 (`sortedTables`, `notIgnoredTables`,
  the `tbl`/`foreignKeysStream` FK loop);
- the sync branch below it still carries the duplicated walk, never calls
  `table()`, never dumps foreign keys, and throws two bespoke `TypeError`s
  ("returned a Promise while tables() was synchronous") that have no Rails
  counterpart.

So every fix to the dump has to be made twice, and the sync path silently skips
`filterIndexesForDump`, `gatherInlineConstraints` and `foreignKeys`.

## Acceptance criteria

1. `tables()` has a single body matching `schema_dumper.rb:134-155`; the
   sync-only walk and its two bespoke `TypeError`s are gone.
2. The mock `SchemaSource` implementations that motivated the sync path either
   return promises or are driven through the async path (Rails' `@connection`
   is always the real thing, so a sync source is a trails-only shape).
3. `schema-dumper.test.ts` stays green, including the tests that construct a
   dumper over a plain object source.

## Absorbed: `converge-schema-dumper-section-blank-lines`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Emit Rails' inter-table and pre-foreign-key blank lines instead of table()'s trailing push"

### Context

Surfaced while landing #6369 (`naming-burndown-ar-schema-dumper-stream`), which
renamed the dumper's accumulator to `stream` and split the foreign-key loop.
Rails separates the dumped sections with explicit blank lines that the port does
not emit; instead `table()` unconditionally pushes a trailing `""` after every
table, which happens to produce a blank line in roughly the right places.

`schema_dumper.rb:139-155`:

```ruby
not_ignored_tables.each_with_index do |table_name, index|
  table(table_name, stream)
  stream.puts if index < not_ignored_tables.count - 1
end

if @connection.supports_foreign_keys?
  foreign_keys_stream = StringIO.new
  not_ignored_tables.each do |tbl|
    foreign_keys(tbl, foreign_keys_stream)
  end

  foreign_keys_string = foreign_keys_stream.string
  stream.puts if foreign_keys_string.length > 0

  stream.print foreign_keys_string
end
```

trails (`packages/activerecord/src/schema-dumper.ts`, `tables`/`table`):

- the inter-table separator is a trailing `stream.push("")` at the end of
  `table()` (so the LAST table also gets one, where Rails emits none);
- the `stream.puts if foreign_keys_string.length > 0` guard before the FK block
  is absent entirely — the FK lines are appended straight after the last table's
  trailing blank.

The two deviations currently cancel out for the common case, which is why the
dump looks right; they diverge for a single-table dump and for a dump whose FK
stream is empty.

### Acceptance criteria

1. `table()` no longer pushes a trailing `""`; `tables()` emits the separator
   between tables per `schema_dumper.rb:141`, and the
   `foreign_keys_string.length > 0` guard per `:150`.
2. `pg`/`mysql` `table()` overrides, which currently pop and re-push that
   trailing `""` (`postgresql/schema-dumper.ts`), are adjusted with it.
3. Dumped output is unchanged for the multi-table canonical schema, and gains
   the Rails-correct shape for the single-table and empty-FK cases.
4. `schema-dumper.test.ts`, `adapters/sqlite3/virtual-table.test.ts` and
   `connection-adapters/mysql/schema-dumper.test.ts` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
