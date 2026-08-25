---
title: "quote_default_expression has two bodies for one Ruby method"
status: done
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: 7047
claim: "2026-08-25T16:18:38Z"
assignee: "collection-proxy-association-seat-is-degenerate-for-singular-names"
blocked-by: null
closed-reason: null
---

## Context

Rails has exactly one `quote_default_expression` body in the abstract layer
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:157-164`),
which the PG and SQLite overrides reach with `super`.

trails has two copies of it: the standalone module function in
`packages/activerecord/src/connection-adapters/abstract/quoting.ts`
(`quoteDefaultExpression`) and a hand-written duplicate as a method on
`AbstractAdapter` (`abstract-adapter.ts`, `quoteDefaultExpression`). Both spell
the same three Rails lines — the Proc arm, the
`lookup_cast_type(column.sql_type).serialize(value)` at rb:161, and the
`quote(value)` at rb:162 — and they have already drifted from each other once
(the guard shapes around the cast-type lookup differed before PR #7035
converged them).

Two bodies for one Ruby method is the divergence risk itself: a future fix to
rb:161's semantics has to be made twice, and nothing enforces that.

Surfaced during PR #7035 (`quote-default-expression-column-is-required`), which
had to edit both copies in lockstep.

## Converged shape

`AbstractAdapter#quoteDefaultExpression` delegates to the standalone module
function (the settled trails shape for a module method mixed onto the adapter —
see the `this`-typed function convention in CLAUDE.md § "Module mixins"), so one
body implements `abstract/quoting.rb:157-164` and the PG/SQLite overrides' `super`
reaches it.

## Acceptance criteria

- [ ] One body implements `abstract/quoting.rb:157-164`; `AbstractAdapter`'s copy
      is gone or is a one-line delegation to it.
- [ ] PG (`postgresql/quoting.rb:156-167`) and SQLite
      (`sqlite3/quoting.rb:99-110`) overrides still reach it via `super`.
- [ ] `sql-default.trails.test.ts` and the adapter quoting suites pass unchanged.
- [ ] parity:api / parity:test delta non-negative; all three lanes green.
