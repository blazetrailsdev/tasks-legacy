---
title: "converge-abstract-dumper-column-spec-bodies-onto-rails"
status: claimed
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: "2026-08-25T16:18:38Z"
assignee: "collection-proxy-association-seat-is-degenerate-for-singular-names"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `ConnectionAdapters::SchemaDumper` onto a real
subclass (PR #6140). The bodies were moved mechanically, so these divergences
predate that PR and were deliberately left untouched by it.

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_dumper.rb`:

- `:46-52` `schema_type_with_virtual` is
  `if @connection.supports_virtual_columns? && column.virtual?` — trails'
  `schemaTypeWithVirtual`
  (`packages/activerecord/src/connection-adapters/abstract/schema-dumper.ts`)
  drops the `supports_virtual_columns?` arm and tests `column.virtual` alone.
- `:62-65` `schema_limit` suppresses the limit when it equals
  `@connection.native_database_types[column.type][:limit]`. trails substitutes an
  `isSerial` guard and says so in a comment ("We don't have the
  native_database_types comparison available here") — the adapter is reachable
  through `_adapter()`, so the comparison is available now.
- `:85-93` `schema_default` opens with a plain `return unless column.has_default?`.
  trails opens with `if (!column.hasDefault && column.default === undefined) return`,
  a two-condition test with no Rails counterpart.

## Converged shape

Each of the three bodies reads as the Ruby does: the `supports_virtual_columns?`
conjunct restored, `schema_limit` comparing against
`native_database_types[column.type][:limit]` off `_adapter()`, and
`schema_default` guarding on `hasDefault` alone.

## Acceptance criteria

- [ ] `schemaTypeWithVirtual` consults the adapter's `supportsVirtualColumns()`.
- [ ] `schemaLimit` compares against the adapter's native type limit rather than
      `isSerial`.
- [ ] `schemaDefault`'s guard is `hasDefault` alone.
- [ ] Dumper suites stay green on all three lanes.
