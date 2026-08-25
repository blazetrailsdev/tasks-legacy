---
title: "Abstract TableDefinition declares jsonb/char/array that Rails does not have"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7024
claim: "2026-08-25T01:44:12Z"
assignee: "retire-attribute-set-narrow-to"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `table-definition-enum-bypasses-enum-type-validation`
(PR #7018), which removed abstract `TableDefinition#enum` because `:enum` is a
`PostgreSQL::ColumnMethods` name, not an abstract one. Three siblings on the same
class have the same defect and were left in place as out of scope.

`pnpm parity:api:extra --package activerecord` lists them as novel on
`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts`:

- **`jsonb`** (`schema-definitions.ts:1455-1457`) — PG-only in Rails: it is in
  PG's `define_column_methods` list
  (`postgresql/schema_definitions.rb:186-189`, `:jsonb`) and appears nowhere in
  `abstract/schema_definitions.rb`. It is the last PG column method still
  declared on the abstract definition after #7018 removed `enum`.
- **`char`** (`schema-definitions.ts:1463-1465`) — no Rails counterpart. There is
  no `char` column method on any Rails `TableDefinition`; `abstract_mysql_adapter.rb:301`
  defines `charset`, which is unrelated.
- **`array`** (`schema-definitions.ts:1467-1469`) — no Rails counterpart as a
  column method. In Rails `array` is an _option_ (`array: true`) and a
  `PostgreSQL::Column` reader (`postgresql/column.rb:37`), never a
  `TableDefinition` DSL method.

## Converged shape

- `jsonb` moves to PG's `TableDefinition`
  (`connection-adapters/postgresql/schema-definitions.ts`), beside the other
  `define_column_methods` names, with the same two-overload
  `(...names)` / `(...names, options)` shape they use — exactly the move #7018
  made for `enum`.
- `char` and `array` are removed. Call sites use `column(name, type, options)`
  with the Rails option spelling (`{ array: true }` for the array case), which is
  what Rails itself writes.

## Acceptance criteria

- Abstract `TableDefinition` declares none of `jsonb`, `char`, `array`.
- `pnpm parity:api:extra --package activerecord` drops all three from
  `connection-adapters/abstract/schema-definitions.ts` (6 novel → 3).
- A PG `createTable` block still resolves `t.jsonb(...)`; note this interacts
  with `create-table-callback-yields-abstract-table-definition` (same RFC), which
  is what makes adapter-specific names typecheck inside a callback — sequence the
  two so the dumped-schema path is never left uncompilable.
- `pnpm typecheck` clean; `parity:api` / `parity:test` deltas non-negative.
