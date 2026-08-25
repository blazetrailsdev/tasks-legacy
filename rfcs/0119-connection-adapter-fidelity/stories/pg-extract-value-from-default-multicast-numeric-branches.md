---
title: "resolve the non-Rails multi-cast numeric branches in PG extractValueFromDefault"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #5409 (RFC 0072
`converge-pg-remove-index-and-new-column-from-field-call-sets`).

Rails' `PostgreSQLAdapter#extract_value_from_default`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:755`)
matches numerics with `/\A\(?(-?\d+(\.\d*)?)\)?(::bigint)?\z/` — an optional
paren pair and an optional `::bigint` cast, nothing more. Anything else falls
to `nil`, and `extract_default_function` then reflects the expression as a
_function_ default.

trails' port
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts#extractValueFromDefault`)
adds two branches Rails does not have:

    /^\((-?\d+(?:\.\d+)?)\)(?:::[\w"\s.]+)+$/    // (150.55)::numeric::money
    /^(-?\d+(?:\.\d+)?)(?:::[\w"\s.]+)+$/        // 150.55::numeric

They exist because two ported tests assert a _literal_ default where Rails'
own code would produce a function default:

- `packages/activerecord/src/adapters/postgresql/money.test.ts` "default"
  (Rails `money_test.rb` `test_default`) — `depth` defaults to `"150.55"`.
- `packages/activerecord/src/adapters/postgresql/schema.test.ts`
  "decimal defaults in new schema when overriding domain" — expects
  `3.14159265358979323846`.

One of two things is true and neither is established: either the PG version
CI runs emits a default expression shape Rails' regex does handle (making the
extra branches unnecessary and the tests passable without them), or Rails
genuinely reflects these as function defaults and the trails tests encode the
wrong expectation. Resolve it against real `pg_get_expr` output.

## Acceptance criteria

- Capture the literal `pg_get_expr` output PG emits for a `money` column with
  `DEFAULT 150.55` and for a decimal domain column, on the PG version CI uses.
- Either delete the two extra branches (Rails' regex suffices) and keep the
  tests green, or confirm Rails reflects a function default and correct the
  two test expectations to match Rails.
- `extractValueFromDefault` ends up byte-faithful to Rails, or the residual
  deviation is documented with the captured PG output at the branch.
