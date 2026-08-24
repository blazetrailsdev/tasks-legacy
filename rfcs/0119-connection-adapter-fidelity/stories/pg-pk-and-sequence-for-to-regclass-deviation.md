---
title: "Pin or retire the to_regclass deviation in PG pk_and_sequence_for"
status: closed
updated: 2026-08-24
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
closed-reason: "Already converged on main (152b2ebe9): postgresql/schema-statements-class.ts:1671-1694 uses Rails' `::regclass` cast (`dep.refobjid = ${quotedTable}::regclass`) inside an outer try mirroring Rails' bare `rescue nil`. `to_regclass` no longer appears in packages/activerecord/src at all, so there is no deviation left to pin or retire."
---

## Context

Surfaced while converging `pk_and_sequence_for` in PR #5389 (RFC 0072 story
`converge-pg-sequence-and-schema-qualified-name-helpers`).

Rails resolves the table with a hard cast —
`WHERE t.oid = #{quote(quote_table_name(table))}::regclass` — and relies on the
method-level `rescue nil` to turn an unknown table into `nil`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:359-410`).

trails emits `to_regclass(...)` instead
(`packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts`,
`pkAndSequenceFor`). Net result is the same — NULL yields no rows, hence nil —
but the two differ in one observable way: PG puts the enclosing transaction into
the aborted state when a `::regclass` cast raises, whereas Ruby's `rescue` leaves
it usable. Under trails' transactional fixtures, the Rails-literal form would
poison the surrounding transaction for every subsequent statement in the test.

The deviation is justified at the call site per CLAUDE.md, but it is a real
divergence in emitted SQL and should be either pinned as permanent (with the
transaction-abort rationale recorded in the deviation register) or retired if
the transaction handling ever makes the literal form safe.

## Acceptance criteria

- Decide and record: pin `to_regclass` as a permanent, documented deviation, or
  converge to Rails' `::regclass` cast plus the outer catch.
- If pinned, add it wherever the project tracks permanent SQL-shape deviations
  so it stops re-surfacing in call-set burndown reviews.
- If converged, prove the aborted-transaction behavior does not break
  transactional fixtures (`pkAndSequenceFor` is called from `resetPkSequence!`,
  `setPkSequence!` and `renameTable`).
