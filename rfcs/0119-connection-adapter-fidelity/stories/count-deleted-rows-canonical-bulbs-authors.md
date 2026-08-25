---
title: "count_deleted_rows_with_lock port uses bespoke test_bulbs/test_authors tables"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/adapters/abstract-mysql-adapter/count-deleted-rows-with-lock.test.ts`
builds two invented tables with raw DDL in `beforeEach`:

```ts
await adapter.execute(
  "CREATE TABLE `test_bulbs` (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255), color VARCHAR(255))",
);
await adapter.execute(
  "CREATE TABLE `test_authors` (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255))",
);
```

and then issues raw `INSERT`/`DELETE` against them.

Rails uses the canonical tables and models —
`activerecord/test/cases/adapters/abstract_mysql_adapter/count_deleted_rows_with_lock_test.rb:4-27`
requires `models/bulb`, `models/car`, `models/author` and works through
`Bulb.unscoped.delete_all`, `Bulb.create!(name: "Jimmy", color: "blue")` and
`Author.create!(name: "Tommy")`. `bulbs` and `authors` are both in the canonical
schema (`vendor/rails/activerecord/test/schema/schema.rb`), so there is no
reason for the bespoke pair. This is the "canonical tables only — no bespoke
tables" rule in CLAUDE.md; the raw-DDL residue survived the RFC 0059
`defineSchema` burndown because it never went through `defineSchema`.

The concurrency shape stays: Rails runs the competing INSERT on a second
thread's own pool connection, and the trails port's second in-test adapter is
the faithful analogue — only the table/model layer converges.

## Acceptance criteria

- The test drops the `test_bulbs` / `test_authors` raw-DDL `beforeEach` and
  `afterEach` and gets `bulbs` + `authors` from the canonical schema via
  `fixtures({ ... })`.
- The body works through the canonical `Bulb` / `Author` models
  (`packages/activerecord/src/test-helpers/models/`), mirroring
  `count_deleted_rows_with_lock_test.rb:11-26` line by line, including
  `Bulb.unscoped().deleteAll()`.
- The single `assert_equal 1, delete_thread.value` assertion is unchanged; the
  file keeps its 0/0/0 assertion-parity standing from PR #6736.
- No new rows in `scripts/parity/unported-files/`; MySQL lane green.
