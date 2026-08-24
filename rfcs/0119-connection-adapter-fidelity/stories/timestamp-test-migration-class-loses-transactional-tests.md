---
title: "PostgresqlTimestampMigrationTest runs non-transactionally; Rails' class is transactional"
status: draft
updated: 2026-07-29
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activerecord/test/cases/adapters/postgresql/timestamp_test.rb`
holds four test classes. The first three (`PostgresqlTimestampTest`,
`PostgresqlTimestampWithAwareTypesTest`, `PostgresqlTimestampWithTimeZoneTest`)
each set `self.use_transactional_tests = false`; the fourth,
`PostgresqlTimestampMigrationTest` (timestamp_test.rb:165-209), does NOT — it
runs transactionally, which is what rolls back the
`add_column :postgresql_timestamp_with_zones, :times, :datetime` its three
tests each perform. Rails' `ensure` blocks only restore `$stdout`.

trails' mirror `packages/activerecord/src/adapters/postgresql/timestamp.test.ts`
sets `fixtures([...], { useTransactionalTests: false })` once at file scope, so
the migration describe is non-transactional too. PR #5551 (which repointed the
file at the boot-laid `postgresql_timestamp_with_zones`) compensated by adding
an explicit `adapter.removeColumn(..., "times")` in each test's `finally` —
correct behavior, but not Rails' mechanism.

The deviation is structural: trails' `fixtures()` applies
`useTransactionalTests` per file, whereas Rails sets it per test class. Any
Rails file mixing transactional and non-transactional classes hits this.

## Acceptance criteria

- The `PostgreSQLTimestampMigrationTest` describe runs transactionally, as
  Rails' class does, and the hand-rolled `removeColumn` cleanups are dropped.
- The other three describes keep `use_transactional_tests = false`.
- Either `fixtures()` gains per-describe scoping or the file is split so each
  scope gets the right setting; whichever is chosen, note it at the call site.
