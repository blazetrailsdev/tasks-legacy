---
title: "Accept PostgreSQL::Name in quoteTableName and the pk-sequence callers"
status: draft
updated: 2026-08-02
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `pkAndSequenceFor` onto Rails' two-query
`PostgreSQL::Name` shape (#5892, RFC 0072).

Rails' sequence consumers pass a `PostgreSQL::Name` straight into
`quote_table_name`:

- `set_pk_sequence!` — `quoted_sequence = quote_table_name(sequence)`
  (`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:319`)
- `reset_pk_sequence!` — same call at
  `schema_statements.rb:341`, where the `sequence` parameter itself may arrive
  as a `Name` from `pk_and_sequence_for`.

That works because PG's `quote_table_name` (`postgresql/quoting.rb`) delegates
through the abstract implementation, which reaches the value via `name.to_s`
(`abstract/quoting.rb#quote_table_name` -> `quote_column_name` ->
`Utils.extract_schema_qualified_name(name.to_s)`), so any object with a
`to_s` — `Symbol`, `String`, `Name` — is accepted.

trails types the method `quoteTableName(name: string): string`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:3461`),
so #5892's callers had to insert an explicit `seq.toString()` that Rails does
not write:

- `setPkSequenceBang` —
  `packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts:1897`
- `resetPkSequenceBang` — same file, `:1916`, which also types its own
  `sequence` parameter as `string | null` where Rails accepts a `Name`.

The behavior is correct — `Name#toString()` produces the same
`schema.identifier` string Ruby's `to_s` does — but the call shape diverges,
and `resetPkSequenceBang`'s narrowed parameter type means a caller holding a
`Name` cannot pass it through the way Rails' can.

## Acceptance criteria

- `quoteTableName` (and the `quoteColumnName` / `extractSchemaQualifiedName`
  path behind it) accepts `PostgreSQL::Name` alongside `string`, matching
  Rails' `name.to_s` reach.
- `resetPkSequenceBang`'s `sequence` parameter accepts a `Name`.
- `setPkSequenceBang` / `resetPkSequenceBang` drop the explicit `.toString()`
  and pass the `Name` through as Rails does.
- Existing PG sequence tests keep passing under PostgreSQL.
