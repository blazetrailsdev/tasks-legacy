---
title: "Wrap nil-able column lists with Ruby Array() semantics in the FK visitor"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging PG unique-constraint DDL (#5383). Rails wraps
possibly-nil column lists with Ruby's `Array()`, which maps `nil` to `[]`;
trails uses the `Array.isArray(x) ? x : [x]` idiom, which maps `null` to
`[null]` and then throws inside the identifier quoter before the statement
ever reaches the database.

PR #5383 fixed the unique-constraint visitor by routing through `wrap()` from
`@blazetrails/activesupport` (the exact `Array()` analogue: null/undefined →
`[]`, arrays pass through, scalars wrap). The same idiom is still live in the
abstract foreign-key visitor:

- `packages/activerecord/src/connection-adapters/abstract/schema-creation.ts:338-341`
  — `(Array.isArray(o.column) ? o.column : [o.column])` and the same for
  `o.primaryKey`.
- Rails: `abstract/schema_creation.rb:84-85` —
  `Array(o.column).map { |c| quote_column_name(c) }` and
  `Array(o.primary_key).map { ... }`.

Currently unreachable from `addForeignKey`, which always fills `column` and
`primaryKey` through `foreign_key_options` before the visitor runs — so this is
latent, not a live bug. It becomes reachable for any caller that constructs a
`ForeignKeyDefinition` directly.

## Acceptance criteria

- Route both list wraps in `visitForeignKeyDefinition` through `wrap()`.
- Sweep for the remaining `Array.isArray(x) ? x : [x]` sites that stand in for a
  Rails `Array()` call and convert them; leave sites where the input is
  statically non-nullable.
- A test constructing a `ForeignKeyDefinition` with a nil column renders the
  Rails-equivalent SQL instead of throwing in the quoter.
