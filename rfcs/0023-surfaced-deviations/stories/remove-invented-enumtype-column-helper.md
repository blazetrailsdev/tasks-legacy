---
title: "PostgreSQL::ColumnMethods#enumType is a trails invention with no Rails counterpart"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by review of PR #5624 (`pg-column-methods-on-change-table-proxy`, RFC 0005).

`PostgreSQL::ColumnMethods` in trails declares an `enumType` member:

```ts
// packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:96
enumType(name: string, enumName: string, options?: ColumnOptions): unknown;
```

implemented on `PostgreSQL::TableDefinition` (same file) as
`pgColumn(name, "string", enumName, options)`.

Rails' `PostgreSQL::ColumnMethods` has no such method. Its
`define_column_methods` list
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:186-191`)
is exactly 32 names and `enum_type` is an **option**, never a column helper. The
`(name, enumName, options)` arity has no Rails analogue either.

PR #5624 verified the 32 generated names against Rails programmatically; `enumType`
is the one member of the trails `ColumnMethods` interface with no counterpart. It
is not currently surfaced as extra API by `parity:api:extra` (the PG ColumnMethods bucket
is satisfied by the other names), so it is invisible to the ratchet.

## Acceptance criteria

- `enumType` is removed from `PostgreSQL::ColumnMethods` and
  `PostgreSQL::TableDefinition`, with callers moved to `t.enum(name, { enum_type })`
  — the Rails-shaped surface.
- If any caller genuinely needs the string-column-with-explicit-sql-type behaviour
  it provides, that need is named explicitly rather than kept as a Rails-shaped alias.
- `parity:api` / `parity:test` deltas non-negative; no new entries in the
  extra-surface allowlist.
