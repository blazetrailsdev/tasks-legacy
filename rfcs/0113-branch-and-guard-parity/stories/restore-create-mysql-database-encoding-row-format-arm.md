---
title: "Restore the dropped row_format_dynamic_by_default? arm in create mysql database with encoding"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activerecord/test/cases/adapters/abstract_mysql_adapter/active_schema_test.rb:125-134`
(`test_create_mysql_database_with_encoding`) opens with an adapter-conditional arm
our port drops entirely:

```ruby
if ActiveRecord::Base.lease_connection.send(:row_format_dynamic_by_default?)
  assert_equal "CREATE DATABASE `matt` DEFAULT CHARACTER SET `utf8mb4`", create_database(:matt)
else
  error = assert_raises(RuntimeError) { create_database(:matt) }
  ...
end
```

The trails port
(`packages/activerecord/src/adapters/abstract-mysql-adapter/active-schema.test.ts`,
`it("create mysql database with encoding")`) only asserts the `charset:` and
`collation:` cases, so neither branch of `AbstractMysqlAdapter#createDatabase`'s
no-options path (`abstract_mysql_adapter.rb:275-284`) is covered. PR #5958 made
that branch call `await isRowFormatDynamicByDefault.call(this)`; the arm is now
easy to exercise because the helper resolves the version lazily.

## Acceptance criteria

- The ported test restores the dropped arm, gated the same way Rails gates it
  (dynamic-by-default → `CREATE DATABASE ... DEFAULT CHARACTER SET utf8mb4`,
  otherwise the "Configure a supported :charset..." error).
- Test name is unchanged.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
