---
title: "_shouldPrepare should be Rails' without_prepared_statement? on the abstract adapter"
status: done
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 1
pr: 7035
claim: "2026-08-25T14:10:32Z"
assignee: "attribute-dup-must-redup-mutable-value"
blocked-by: null
closed-reason: null
---

## Context

> **Premise corrected 2026-08-25 (PR #7035).** This story was written against
> `without_prepared_statement?` at
> `activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:1177`.
> That method **no longer exists in vendored Rails** — `grep -r
without_prepared_statement vendor/rails/**/lib` returns nothing; the name
> survives only in test fixture config (`test/config.yml:77`,
> `test/support/config.rb:29`) and two test method names. Adding it would have
> invented surface Rails does not have, so the converged shape and acceptance
> criteria below were rewritten to Rails' real gate before the story shipped.

Rails has no `should_prepare?`. The gate at this position is spelled inline at
the one place Rails decides it:

```ruby
prepare: prepared_statements && preparable,
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:74`,
in `to_sql_and_binds`.) Every downstream `perform_query` / `raw_execute` then
takes `prepare:` as a passed argument rather than re-deriving it.

trails instead carried a positive `_shouldPrepare` predicate duplicated on both
adapters — `mysql2-adapter.ts` and `postgresql-adapter.ts` — with a leading
underscore and no Ruby counterpart at all, called from ~10 sites.

Surfaced by `should-prepare-statement-limit-gate-is-invented` (PR #6293),
which converged the _body_ of both gates to Rails' condition but deliberately
left the name and the duplication alone to keep that PR scoped.

## Converged shape

Delete `_shouldPrepare` from both adapters and spell the gate the way
`database_statements.rb:74` does at the call sites. There is no method to add:
Rails has no named predicate here, so introducing one would be novel surface,
which `parity:api:extra` measures.

## Acceptance criteria

- [x] `_shouldPrepare` is gone from both adapters, and no named replacement
      predicate is introduced (Rails has none).
- [x] Call sites read Rails' gate, `prepared_statements && preparable`
      (`abstract/database_statements.rb:74`).
- [x] `pnpm parity:api:extra --package activerecord` loses the two novel rows.
- [x] parity:api / parity:test delta non-negative; all three lanes green.

## Known remaining approximation

The mysql2 fallback in `mysql2/database-statements.ts` substitutes
`binds.length > 0` for Rails' `preparable`, which is really Arel's
`Collector#preparable` flag threaded down from `to_sql_and_binds`, not a bind
count. That substitution predates this story (it lived inside `_shouldPrepare`)
and is tracked separately by
`mysql2-raw-execute-preparable-is-a-bind-count-approximation`.
