---
title: "PostgreSQLAdapter#buildExplainClause invents option validation Rails does not have"
status: done
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
pr: 6890
claim: "2026-08-22T22:50:07Z"
assignee: "wave-5d-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Rails' `PostgreSQL::DatabaseStatements#build_explain_clause`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/database_statements.rb:96-100`)
is three lines and validates nothing:

```ruby
def build_explain_clause(options = [])
  return "EXPLAIN" if options.empty?

  "EXPLAIN (#{options.join(", ").upcase})"
end
```

trails' `PostgreSQLAdapter#buildExplainClause`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:2331`)
instead routes rendering through `_validateExplainOptions`
(`postgresql-adapter.ts:2364`), which rejects unknown flags against
`EXPLAIN_FLAGS` (`:2344`), rejects unknown formats against `EXPLAIN_FORMATS`
(`:2362`), enforces at most one FORMAT, and reorders FORMAT last. None of
that has a Rails counterpart — Rails hands whatever the caller passed
straight to `join(", ").upcase` and lets the server reject it.

PR #6811 converged the MySQL half of exactly this shape: it deleted
`AbstractMysqlAdapter._validateExplainOptions` and its `EXPLAIN_FLAGS` /
`EXPLAIN_FORMATS` allowlists and delegated to the single module body in
`connection-adapters/mysql/database-statements.ts`, which is
`mysql/database_statements.rb:36-46` line for line. PG is the remaining half,
and the parent story
(`mysql-build-explain-clause-conflates-explain-fallback`, closed by #6811)
explicitly noted the separation chosen should apply to both adapters.

PR #6581 had already converged PG's _header_ (no `" for:"` suffix, and the
invented `_explainStatementClause` twin deleted); the option validation is
what is left.

## Converged shape

`buildExplainClause` becomes the Rails body: `return "EXPLAIN"` when options
are empty, otherwise `` `EXPLAIN (${options.join(", ").toUpperCase()})` ``.
`_validateExplainOptions`, `EXPLAIN_FLAGS` and `EXPLAIN_FORMATS` are deleted.

Note `packages/activerecord/src/adapters/postgresql/` explain tests currently
assert the rejection behaviour; those assertions cover invented surface and go
with it, the way the MySQL `buildExplainClause rejects unknown format` case
was removed in #6811.

## Acceptance criteria

- [ ] `PostgreSQLAdapter#buildExplainClause` is the Rails body from
      `postgresql/database_statements.rb:96-100`, with `join` and `upcase`
      visible.
- [ ] `_validateExplainOptions`, `EXPLAIN_FLAGS`, `EXPLAIN_FORMATS` are gone
      from `postgresql-adapter.ts`.
- [ ] `pnpm parity:api:extra --package activerecord` total falls; no novel
      surface added.
- [ ] `pnpm parity:api:calls` / `:args` clean, no baseline rows added.
- [ ] PG lane green, in particular the explain suites.
