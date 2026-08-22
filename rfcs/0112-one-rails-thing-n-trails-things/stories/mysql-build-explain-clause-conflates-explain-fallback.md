---
title: "AbstractMysqlAdapter#buildExplainClause conflates the Explain fallback header and invents option validation"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 160
pr: 6811
claim: "2026-08-21T11:40:36Z"
assignee: "hash-config-primary-resolves-via-global-configurations"
blocked-by: null
closed-reason: null
---

## Context

Rails' `MySQL::DatabaseStatements#build_explain_clause`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/database_statements.rb:36-46`):

```ruby
def build_explain_clause(options = [])
  return "EXPLAIN" if options.empty?
  explain_clause = "EXPLAIN #{options.join(" ").upcase}"
  if analyze_without_explain? && explain_clause.include?("ANALYZE")
    explain_clause.sub("EXPLAIN ", "")
  else
    explain_clause
  end
end
```

Trails has TWO implementations and they disagree:

1. `packages/activerecord/src/connection-adapters/mysql/database-statements.ts`
   (`buildExplainClause`, around line 69) is the faithful port: `join`,
   `include("ANALYZE")`, returns bare `"EXPLAIN"` when options are empty.
2. `packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts`
   (`buildExplainClause`, around line 1474) is the one the adapter actually
   exposes. It returns `"EXPLAIN for:"` for empty options and routes rendering
   through `_validateExplainOptions`, which rejects unknown flags/formats,
   enforces at most one FORMAT, and reorders FORMAT last.

The `" for:"` suffix belongs to `ActiveRecord::Explain#build_explain_clause`
(`vendor/rails/activerecord/lib/active_record/explain.rb:55-61`), which is the
FALLBACK for adapters that do not respond to `build_explain_clause`. Folding it
into the adapter's own method conflates the two Rails methods, and
`_validateExplainOptions` has no Rails counterpart at all.

Surfaced in #5374; the two wide-call entries (`build_explain_clause` dropping
`include?` and `join`) are baselined against
`scripts/api-compare/call-mismatches-wide-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`
with a reason rather than converged, because collapsing the adapter method onto
the module function would drop the option validation that
`packages/activerecord/src/adapters/abstract-mysql-adapter/mysql-explain.test.ts`
currently asserts.

Note PostgreSQL has the same shape at
`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:2458`, so
whatever separation is chosen should apply to both adapters.

### Update (2026-08-15, PR #6581)

The PostgreSQL half is **done**. #6581 converged
`PostgreSQLAdapter#buildExplainClause` onto
`postgresql/database_statements.rb:96-100` — it returns bare `"EXPLAIN"` /
`"EXPLAIN (...)"` with no `" for:"` — and deleted the invented
`_explainStatementClause` twin that existed only because the header and the
executed statement had drifted apart once the suffix was baked in. `explain`
now composes its SQL from `buildExplainClause` + `toSql` exactly as
`postgresql/database_statements.rb:8` does. PG keeps `_validateExplainOptions`,
so the "option validation has no Rails counterpart" half of this story is still
open on both adapters.

That leaves MySQL as the only remaining `" for:"` emitter among adapters that
define `build_explain_clause`, at
`abstract-mysql-adapter.ts:1303-1306`, with its `_explainClause` /
`_explainStatementClause` twin at `:1321` and `:1369`.

Also surfaced while fixing PG: **`AbstractAdapter#buildExplainClause`
(`connection-adapters/abstract-adapter.ts:939-951`) is itself invented
surface.** Rails' `AbstractAdapter` defines no `build_explain_clause` at all —
which is precisely why `explain.rb:56-61` probes with
`respond_to?(:build_explain_clause, true)`. Because trails' abstract defines
one, _every_ adapter answers the probe, the `"EXPLAIN for:"` fallback in
`packages/activerecord/src/explain.ts:96` is unreachable, and SQLite produces
the Rails-correct header only by accident rather than by taking the fallback.
Retiring the abstract member is what makes the third acceptance criterion below
("the `\" for:\"` header lives only in explain.ts") actually true rather than
nominally true.

## Acceptance criteria

- `AbstractMysqlAdapter#buildExplainClause` is the Rails method: no `" for:"`
  suffix, `join` and `include("ANALYZE")` visible in the body.
- The `" for:"` header lives only in `packages/activerecord/src/explain.ts`,
  matching explain.rb:55-61.
- Option validation either moves behind a clearly named non-Rails helper that
  is not on the `build_explain_clause` path, or is justified at its call site
  as a deliberate addition.
- The single module-level `buildExplainClause` in
  `connection-adapters/mysql/database-statements.ts` is the only body; the
  adapter delegates, as `write_query?` / `returning_column_values` already do.
- `AbstractAdapter#buildExplainClause` is removed so `explain.rb`'s
  `respond_to?` probe means what it means in Rails, and SQLite reaches the
  fallback by the Rails path; `parity:api:extra --package activerecord` falls.
- Both `build_explain_clause` entries drop out of the wide exclude file;
  `pnpm parity:api:calls` baseline strictly shrinks.
- `Relation#explain` output is unchanged on MySQL and MariaDB (the
  `analyze_without_explain?` rewrite still applies).

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
