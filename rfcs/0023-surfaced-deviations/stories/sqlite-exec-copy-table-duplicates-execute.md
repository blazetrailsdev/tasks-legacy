---
title: "SQLite execCopyTable hand-rolls execute's instrumentation"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' SQLite `copy_table` / `move_table` issue their DDL through `execute`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb`,
the `alter_table` / `copy_table` pair), so instrumentation comes for free.

trails' `execCopyTable`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`) drives
`this.driver.exec` directly to keep control of savepoint nesting, which means it
sits outside `execute`'s notification path. Before PR #5571 the entire
alter_table rebuild was invisible to `assert_queries_count` — the block counted
2 queries where Rails counts 14 — so ported query-count assertions silently
became no-ops.

PR #5571 made it emit the notification itself:

```ts
const payload = this._notificationPayload(sql, [], "SQL");
await Notifications.instrumentAsync("sql.active_record", payload, async () => { ... });
```

That fixes the count, but it hand-rolls what `execute` already does and can
drift from it: `execute` also runs `preprocessQuery`, `materializeTransactions`,
and bind type-casting, none of which `execCopyTable` performs. Rails has one
path; trails now has two that must be kept in sync by hand.

Note Rails' own `name` for these statements is nil, whereas `execute` defaults
to `"SQL"`. Both are counted by `assert_queries_count` (which filters only
`"SCHEMA"`), so this does not affect counts, but it does affect log output.

Related, already filed: `copy-table-indexes-bypasses-add-index` and
`sqlite-copy-table-indexes-drops-order-and-expression`.

## Acceptance criteria

- [ ] `execCopyTable` routes through the same path `execute` uses, or the
      duplication is eliminated another way, so there is one instrumented
      statement path in the rebuild.
- [ ] Savepoint nesting behaviour is unchanged (the reason the bypass exists) —
      `alterTable` still works inside and outside an open transaction.
- [ ] `columns.test.ts`'s `test_remove_columns_single_statement` and
      `test_add_timestamps_single_statement` still count 14 on SQLite.
- [ ] Green on all three lanes.
