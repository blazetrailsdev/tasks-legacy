---
title: "converge-create-schema-dumper-source-arity"
status: claimed
updated: 2026-08-24
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T22:45:55Z"
assignee: "converge-activesupport-temporal-receiver-chaining"
blocked-by: null
closed-reason: null
---

## Context

RFC 0096 wave-5 residual finding, split out of
`wave-5-residual-arg-shape-findings` (PR #6929).

Rails' `create_schema_dumper(options)` passes `self` to the dumper
(`activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb`,
`.../abstract/schema_statements.rb`). trails takes `(source, options)` — three
rows, two in `connection-adapters/postgresql-adapter.ts` and one in
`connection-adapters/sqlite3/schema-statements.ts`.

The extra leading parameter exists because
`packages/activerecord/src/schema-dumper.ts:573` deliberately passes a
**wrapped** source rather than the adapter itself, so rewiring the callee to
`this` alone breaks that call site. This is an arity convergence that has to
move the wrapping to the dumper side, not a mixin-receiver rewiring.

## Acceptance criteria

1. `createSchemaDumper` takes Rails' `(options)` and uses `this` as the source;
   the wrapping `schema-dumper.ts:573` needs is provided without an extra
   parameter on the Rails-matched signature.
2. The three rows are gone from `pnpm parity:api:calls:args:report`; no new
   `call-mismatches-exclude/` row and no new `arity-exclude.json` row.
3. `pnpm parity:api` arity delta is non-negative; both call gates stay green.
