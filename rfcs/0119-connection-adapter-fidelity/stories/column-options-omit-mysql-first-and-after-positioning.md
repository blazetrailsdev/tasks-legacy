---
title: "ColumnOptions declares no first/after column-positioning keys"
status: draft
updated: 2026-07-29
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Surfaced while shipping PR #5614 (`port-column-methods-primary-key-helper`,
RFC 0005).

`ColumnOptions`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:488`)
declares no `first` or `after` key, so MySQL's column-positioning options are
unrepresentable anywhere in the schema-definition surface. Rails carries them
through `add_column`/`change_column` into
`MySQL::SchemaStatements#add_column_position!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb`),
and `ColumnMethods#primary_key` passes them straight through `**options`.

Concretely: `t.primary_key :id, first: true` — the exact call in Rails'
`change_table_test.rb:121` — does not typecheck in trails. #5614's port of the
helper had to swap `first: true` for `comment:` in its change_table assertion
to keep the same `add_column` shape under test.

This is the root gap under the existing draft
`reference-definition-polymorphic-options-forwards-first-and-after`
(0023): that story asks `polymorphicOptions` to forward `options.slice(:null,
:first, :after)`, but there is nothing to forward while the keys do not exist
on the type. Whichever ships first should note the other.

## Acceptance criteria

- `ColumnOptions` declares `first?: boolean` and `after?: string`, matching the
  keys Rails' MySQL adapter consumes.
- The MySQL adapter honours them when emitting `ADD COLUMN` / `CHANGE COLUMN`
  (`add_column_position!`); other adapters ignore them exactly as Rails does
  (they are MySQL-only in Rails too — confirm before generalising).
- A test asserts `t.primary_key("id", "primary_key", { first: true })` reaches
  `addColumn` with the option intact, restoring the Rails-literal form
  `change_table_test.rb:121` uses.
- Note the relationship to
  `reference-definition-polymorphic-options-forwards-first-and-after` in
  whichever story lands second.
