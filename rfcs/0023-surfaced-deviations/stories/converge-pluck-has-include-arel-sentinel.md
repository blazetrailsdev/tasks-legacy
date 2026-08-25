---
title: "pluck passes column_names.first to has_include?, dropping the invented \\0arel sentinel"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6619 (RFC 0096 `wave-4-naming-ar-relation`). The naming row
`calculations.ts` / `has_include?` (`ref:first` -> `ref:firstColumnName`) is
not a rename — trails passes a synthetic sentinel where Rails passes the value.

Rails (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:309`):

```ruby
if has_include?(column_names.first)
```

trails (`packages/activerecord/src/relation/calculations.ts`, `pluck`):

```ts
const firstColumnName =
  columnNames.length === 0 ? null : typeof columnNames[0] === "string" ? columnNames[0] : "\0arel";
if (hasInclude(this as any, firstColumnName)) {
```

The `"\0arel"` string is a trails invention: a non-string first column (an
`Arel::Attribute`, `NamedFunction` or `SqlLiteral`) is replaced by a sentinel
that `hasInclude` is expected to treat as "present but not a column name".
Rails just hands `has_include?` the object and lets
`eager_loading? || (includes_values.present? && (column_name || references_eager_loaded_tables?))`
read it for truthiness.

A sentinel that leaks into a comparison is exactly the kind of invented surface
CLAUDE.md rules out, and it makes the pluck/has_include contract unreadable
next to the Ruby.

## Acceptance criteria

- [ ] `pluck` passes `columnNames[0]` (Rails' `column_names.first`) to
      `hasInclude`, with no sentinel.
- [ ] `hasInclude` mirrors calculations.rb's `has_include?` truthiness on
      whatever object it is handed, including a non-string Arel node.
- [ ] The `"\0arel"` literal is gone from the package.
- [ ] The `has_include?` naming row clears in
      `pnpm parity:api:calls:args:report`, with no new `shape` row.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green (pluck-with-includes
      tests).
