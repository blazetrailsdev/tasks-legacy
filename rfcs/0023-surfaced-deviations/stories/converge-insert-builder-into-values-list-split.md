---
title: "Builder#into bundles values_list; Rails interpolates the two fragments separately"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`InsertAll::Builder` in Rails exposes `into` and `values_list` as two separate
fragments, and every adapter's `build_insert_sql` interpolates both:
`"INSERT #{insert.into} #{insert.values_list}"`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:647`,
`:662`; likewise `postgresql_adapter.rb` / `sqlite3_adapter.rb`, and
`abstract_adapter.rb:843`). Rails' `Builder#into` is only
`"INTO #{model.quoted_table_name} (#{columns_list})"` (`insert_all.rb`
`Builder#into`), and `values_list` compiles the `VALUES (...)` node.

trails' `Builder.into()` (`packages/activerecord/src/insert-all.ts:660-675`)
bundles the compiled VALUES list into its own return value, so all four TS
adapters emit `INSERT ${insert.into()}` alone. `Builder.valuesList()` exists
but is not part of the `InsertBuilder` contract. This is documented at
`insert-all.ts:576-583`, i.e. a ratified deviation rather than a converged one.

Surfaced while porting the MySQL raw-alias arm in PR #6580: the alias arm's
`INSERT #{insert.into} #{insert.values_list} AS #{values_alias}` had to be
written as `INSERT ${insert.into()} AS ${valuesAlias}`, which happens to
produce the same string but does not read as the same method.

## Converged shape

`Builder.into()` returns only `INTO <quoted_table> (<columns>)` (or the
`DEFAULT VALUES` / `() VALUES ()` no-column forms Rails handles), `valuesList()`
joins the `InsertBuilder` contract, and each adapter body interpolates both in
the Rails order. Verify the no-column arms still emit identical SQL on all
three dialects.

## Acceptance criteria

- [ ] `InsertBuilder` exposes `into()` and `valuesList()` as separate members.
- [ ] All four `buildInsertSql` bodies interpolate `${insert.into()} ${insert.valuesList()}`.
- [ ] The bundling note at `insert-all.ts:576-583` is deleted, not reworded.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
