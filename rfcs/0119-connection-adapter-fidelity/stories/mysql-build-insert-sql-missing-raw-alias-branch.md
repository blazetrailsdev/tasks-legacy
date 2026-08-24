---
title: "buildInsertSql never takes Rails' supports_insert_raw_alias_syntax? branch"
status: draft
updated: 2026-07-29
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AbstractMysqlAdapter#build_insert_sql` has two branches keyed on
`supports_insert_raw_alias_syntax?` (MySQL >= 8.0.19, never MariaDB):
`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:644-661`
emits `INSERT … VALUES (…) AS <table>_values` and references `<alias>.<column>`
in `ON DUPLICATE KEY UPDATE`; only the else-branch (line 662-678) uses the
deprecated `VALUES(<column>)` function.

trails implements the else-branch only —
`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1264`
always emits `col=VALUES(col)` — even though the predicate itself is ported at
`abstract-mysql-adapter.ts:1798` (`supportsInsertRawAliasSyntax`) and is
therefore dead for this purpose.

Observable failure: `insert-all.test.ts > InsertAllTest > upsert and db warnings`
fails on a MySQL 8.4 server with

```text
SQLWarning: 'VALUES function' is deprecated and will be removed in a future
release. Please use an alias (INSERT INTO ... VALUES (...) AS alias) and replace
VALUES(col) in the ON DUPLICATE KEY UPDATE clause with alias.col instead
```

because that test asserts the upsert raises no warning under
`withDbWarningsAction`. Reproduced on PR 5585 and confirmed identical on
`origin/main`, so it predates that PR. CI's mysql lane is a MariaDB stand-in
(`supports_insert_raw_alias_syntax?` false), which is why CI never sees it.

## Acceptance criteria

- `buildInsertSql` takes the raw-alias branch when `supportsInsertRawAliasSyntax()`
  is true, with the `<table_name>_values` alias Rails derives via `parameterize`
  and `ON DUPLICATE KEY UPDATE` assignments qualified by that alias, including
  the `raw_update_sql?` sub-branch that rebuilds the statement without the alias
  (abstract_mysql_adapter.rb:654-656).
- `touch_model_timestamps_unless` receives the alias-qualified comparison
  (`<quoted_table>.<col><=><alias>.<col>`), matching line 658.
- `InsertAllTest > upsert and db warnings` passes against a MySQL >= 8.0.19
  server; the MariaDB and MySQL < 8.0.19 paths keep the `VALUES(col)` form.
- No test renamed.
