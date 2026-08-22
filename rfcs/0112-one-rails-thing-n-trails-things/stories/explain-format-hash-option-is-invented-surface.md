---
title: "The { format } EXPLAIN option arm is invented surface Rails' join cannot express"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ExplainOption` in
`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:66`
is `string | { format: string }`. The object arm has no Rails counterpart.

Rails' `Relation#explain(*options)` forwards its options to the adapter's
`build_explain_clause`, and both adapter bodies render them with a bare join:

- `mysql/database_statements.rb:39` — `"EXPLAIN #{options.join(" ").upcase}"`
- `postgresql/database_statements.rb:99` — `"EXPLAIN (#{options.join(", ").upcase})"`

A Ruby Hash passed there would render as `{:format=>"json"}.to_s.upcase`, i.e.
garbage — so Rails' options are only ever Symbols/Strings, and the way a Rails
user asks for a format is `explain(:analyze, :verbose)` style flags, with
`FORMAT` spelled as one of them where the server supports it.

trails special-cases the object arm in both adapters instead:

- `connection-adapters/mysql/database-statements.ts` (`buildExplainClause`)
  maps `{ format }` to `FORMAT=JSON` rather than joining it.
- `connection-adapters/postgresql-adapter.ts:2364` (`_validateExplainOptions`)
  maps it to `FORMAT JSON` and reorders it last.

This is invented public API: it widens the signature of `Relation#explain`
beyond what Rails accepts, and it is the reason both adapters needed a
per-option `map` where Rails has a single `join`.

Surfaced while converging the MySQL half in PR #6811
(`mysql-build-explain-clause-conflates-explain-fallback`). #6811 deliberately
left the module body's `{ format }` arm alone — it is pre-existing and the
story scoped to the header/fallback conflation — so it is filed here rather
than propagated.

## Converged shape

`ExplainOption` becomes `string` (a Ruby Symbol or String, per the
repo's Symbol convention), both adapter bodies drop the per-option `map` in
favour of Rails' `join`, and a caller wanting a format passes it as a flag
string. `inspectExplainOption`
(`abstract/database-statements.ts:72`) exists only to render the object arm in
error messages and goes with it.

Sequence with
`pg-explain-option-validation-has-no-rails-counterpart`: that story deletes
PG's `_validateExplainOptions`, which is where PG's object arm lives, so
landing it first shrinks this one to the MySQL body plus the type.

## Acceptance criteria

- [ ] `ExplainOption` no longer admits an object arm.
- [ ] `mysql/database-statements.ts` `buildExplainClause` renders options with
      a plain `join(" ").toUpperCase()`, matching
      `mysql/database_statements.rb:39`.
- [ ] The PG body matches `postgresql/database_statements.rb:99`.
- [ ] `inspectExplainOption` is deleted if it has no remaining caller.
- [ ] Explain suites updated for the flag spelling; no test renames.
- [ ] `pnpm parity:api:extra --package activerecord` total falls.
