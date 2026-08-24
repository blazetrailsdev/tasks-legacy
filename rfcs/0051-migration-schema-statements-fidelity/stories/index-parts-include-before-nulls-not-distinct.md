---
title: "SchemaDumper#index_parts emits nullsNotDistinct before include"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
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

# `SchemaDumper#index_parts` emits `nullsNotDistinct` before `include`

## Context

Rails' `index_parts` (`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:265-280`)
appends in this order: `unique` → `length` → `order` → `opclass` → `where` →
`using` → `include` → `nulls_not_distinct` → `type` → `comment`.

The port (`packages/activerecord/src/schema-dumper.ts:977-987`) emits
`nullsNotDistinct` **before** `include`, inverting the two. Surfaced by the
review of #6698 (`schema-dumper-emit-table-and-underscored-callee-convergence`),
which converged the adjacent `using:` line but left the ordering alone as
out of scope.

The pair only both appear on PostgreSQL covering indexes that are also
`NULLS NOT DISTINCT`, so the emitted `add_index` line differs from Rails'
only in that case — but the dump is byte-compared in parity work, so it matters.

## Acceptance criteria

- `indexParts` appends `include` before `nullsNotDistinct`, matching
  schema_dumper.rb:265-280 line for line.
- A PostgreSQL test covers an index that is both covering (`include`) and
  `nulls_not_distinct`, asserting the Rails order.
