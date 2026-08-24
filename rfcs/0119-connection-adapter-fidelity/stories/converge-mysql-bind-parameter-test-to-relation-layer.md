---
title: "Converge MySQL bind-parameter test to Rails' canonical fixtures and relation-layer quoting"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/adapters/abstract-mysql-adapter/bind-parameter.test.ts`
diverges from its Rails counterpart
`vendor/rails/activerecord/test/cases/adapters/abstract_mysql_adapter/bind_parameter_test.rb`
on two linked points. Surfaced while converting the file to
`leaseMysqlAdapter()` in #5327 (story
`mysql-tests-self-built-adapter-burndown-batch-2`); #5327 changed only the
connection, not the test bodies, so both deviations are pre-existing.

1. **Bespoke table.** The trails file `CREATE TABLE`s `bind_param_items`
   (`bind-parameter.test.ts:30`, dropped at `:36`) via raw `adapter.exec`.
   That table exists nowhere in Rails — `grep bind_param_items vendor/rails/`
   is empty. Rails declares `fixtures :topics, :posts` (`bind_parameter_test.rb:11`)
   and drives everything through the canonical `Topic` and `Post` models. This
   violates the canonical-tables-only rule in CLAUDE.md, and now that the file
   leases the ambient connection the bespoke DDL runs on the shared connection
   every other suite in the worker uses.

2. **Wrong layer.** Rails' `assert_quoted_as` private helper
   (`bind_parameter_test.rb:74-87`) pins a _relation-layer_ property:
   `Post.where("title = ?", value).to_sql` must quote a typed value against a
   string column as a string literal (`0` → `'0'`, `0.0` → `'0.0'`,
   `false` → `'0'`), so the six `test_where_with_*_for_string_column_using_bind_parameters`
   cases assert both the emitted SQL and that the relation matches 0 rows. The
   trails port tests the raw `?`-bind / driver layer instead, which has no
   column-type knowledge, so it asserts MySQL's numeric _coercion_ — the
   opposite of Rails' intent. The file's own header comment
   (`bind-parameter.test.ts:1-18`) documents this and calls the faithful
   assertion "unreachable at this layer". It is reachable at the relation
   layer, which is where Rails puts it.

Also missing entirely: `test_update_question_marks`, `test_create_question_marks`,
`test_update_null_bytes`, `test_create_null_bytes` (`bind_parameter_test.rb:14-48`)
— `foo?bar` and `foo\0bar` round-trips through `Topic#save!` / `Topic.create!`.

## Acceptance criteria

- [ ] `bind_param_items` is gone; the file uses `fixtures({ ... })` with the
      canonical `topics` and `posts` tables and the official `Topic` / `Post`
      models, matching `fixtures :topics, :posts`.
- [ ] The six `where with ... for string column using bind parameters` cases go
      through a relation-layer `assert_quoted_as` equivalent — asserting
      `Post.where("title = ?", value).toSql()` string-quotes the value and that
      the relation matches the expected row count — not driver-level coercion.
- [ ] The four question-mark / null-byte round-trip tests are ported.
- [ ] The header comment explaining the adapter-layer compromise is deleted,
      since the compromise is gone.
- [ ] Test names match Rails verbatim.
