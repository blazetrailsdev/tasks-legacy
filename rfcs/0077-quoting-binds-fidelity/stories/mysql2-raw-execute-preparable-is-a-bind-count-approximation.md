---
title: "mysql2-raw-execute-preparable-is-a-bind-count-approximation"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails decides prepared-statement routing exactly once, in `to_sql_and_binds`:

```ruby
prepare: prepared_statements && preparable,
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:74`).
`preparable` is Arel's `Collector#preparable` flag — set by
`Arel::Collectors::Composite`/`Bind` while the statement is built, i.e. "was
this SQL produced through Arel with real bind params". Everything downstream
(`raw_execute`, `perform_query`, `internal_exec_query`) receives `prepare:` as a
passed argument and never re-derives it.

trails' `packages/activerecord/src/connection-adapters/mysql2/database-statements.ts`
`performQuery` carries a fallback for callers that do not state `prepare:`:

```ts
const prepare = options.prepare ?? (this.preparedStatements === true && binds.length > 0);
```

`binds.length > 0` is a stand-in for `preparable`, not the same predicate: a
hand-built SQL string with binds is preparable-false in Rails but true here, and
Rails has no fallback at this position at all — the argument is always stated.

The substitution predates PR #7035, which only deleted the `_shouldPrepare`
method the expression used to live in (surfaced by that PR's review).

## Acceptance criteria

- [ ] Every trails caller of `performQuery` / `rawExecute` in the mysql2 adapter
      states `prepare:` explicitly, as Rails' callers do
      (`abstract/database_statements.rb:552-556`,
      `mysql2/database_statements.rb:41`).
- [ ] The `options.prepare ?? ...` bind-count fallback is deleted; `prepare` is a
      required argument threaded from `to_sql_and_binds`' gate.
- [ ] If a caller genuinely cannot know `preparable`, thread Arel's collector
      flag rather than approximating it with a bind count.
- [ ] parity:api / parity:test delta non-negative; all three lanes green.
