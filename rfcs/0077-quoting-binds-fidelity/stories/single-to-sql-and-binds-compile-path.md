---
title: "One to_sql_and_binds compile path; retire exceedsBindParamsLimit and compileInlined"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps:
  - should-prepare-should-be-without-prepared-statement
deps-rfc: []
est-loc: 200
priority: 7
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails has exactly **one** arel→SQL compile path,
`DatabaseStatements#to_sql_and_binds`
(`activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:20-46`),
and it owns the whole decision: pick `collector()`, seed
`collector.retryable = true`, branch on `prepared_statements`, apply the
over-limit `unprepared_statement` fallback, then read back `preparable` and
`allow_retry`.

trails re-implements that decision in four places:

- `packages/activerecord/src/connection-adapters/abstract/database-statements.ts`
  `toSqlAndBinds` (the real port)
- `packages/activerecord/src/relation.ts` `_compileSelectSql`
- `packages/activerecord/src/relation.ts` `_compileAstWithBinds`
- `packages/activerecord/src/relation/calculations.ts` `compileManagerWithBinds`

PR #6291 (`unprepared-statements-inline-binds`) had to add Rails' non-prepared
branch to **all four** — the same edit four times, which is the tell. It also
routed `_compileSelectSql` through `conn.toSqlAndBinds` for the `allow_retry`
read (rb:45), proving the delegation works; the other three still compile
`visitor.compileWithBinds` directly and discard `retryable`/`preparable`.

Two invented helpers exist only to serve the duplication and should die with it:

- `exceedsBindParamsLimit` (`connection-adapters/abstract/database-limits.ts`) —
  Rails inlines `binds.length > bind_params_length` at rb:36. It is a shared
  helper purely because three call sites needed the same test.
- `compileInlined` (`abstract/database-statements.ts`) — Rails just calls
  `collector()`, which already returns `SubstituteBinds` when
  `prepared_statements` is false (`abstract_adapter.rb#collector`).

Also folded in: trails tests `preparedStatements === false` rather than Rails'
`if prepared_statements`, because bare visitor stand-ins in tests carry no flag
and Rails' default is `true`. With one compile path through the adapter, the
receiver is always a real adapter and the plain truthiness test applies.

## Converged shape

`_compileSelectSql`, `_compileAstWithBinds` and `compileManagerWithBinds` all
delegate to the connection's `toSqlAndBinds`, which becomes the sole place the
prepared/unprepared collector choice, the over-limit fallback, and the
`preparable` / `allow_retry` reads live — as rb:20-46 does. `exceedsBindParamsLimit`
and `compileInlined` are deleted.

## Acceptance criteria

- [ ] One compile path: the three relation/calculation sites call
      `toSqlAndBinds` rather than `visitor.compileWithBinds`.
- [ ] `exceedsBindParamsLimit` and `compileInlined` are gone; the over-limit test
      is inline at the rb:36 position.
- [ ] The `preparedStatements === false` guard becomes Rails' `if prepared_statements`.
- [ ] `unprepared-statements-inline-binds.trails.test.ts` (both cases) and
      `bind-parameter.test.ts` pass on all three lanes; `parity:api:extra` loses the two
      helper rows.
