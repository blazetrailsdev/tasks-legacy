---
title: "Port Rails' timestamped_migrations=false cases for nextMigrationNumber/isValidateTimestamp"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

PR #5564 (RFC 0081, `convert-ar-config-accessors-internal-flags`) implemented two
`timestamped_migrations` reads that trails was missing, but shipped them
without tests — the wide calls ratchet was the only thing exercising them.

Implemented in that PR:

- `packages/activerecord/src/migration.ts` `nextMigrationNumber` — added Rails'
  `else "%.3d" % number.to_i` sequential branch
  (`vendor/rails/activerecord/lib/active_record/migration.rb:1128-1134`).
  Before, the UTC timestamp was returned unconditionally and the flag was never
  consulted, so `timestamped_migrations = false` had no effect at all.
- `packages/activerecord/src/migration.ts` `isValidateTimestamp` — added the
  missing `timestamped_migrations &&` conjunct
  (`migration.rb:1377-1379`).

Neither path has a trails test with the flag flipped to false. Rails covers
both in `vendor/rails/activerecord/test/cases/migration_test.rb` — see
`:1600`, `:1606`, `:1627`, `:1711` and
`test_migration_succeeds_despite_future_timestamp_if_timestamped_migrations_is_false`
at `:1869-1879`.

Port the relevant cases verbatim (test names must match Rails). A regression
test here must fail against the pre-#5564 implementation — the sequential
branch and the conjunct are both silent no-ops without it.

## Acceptance criteria

- Rails' `timestamped_migrations = false` cases from `migration_test.rb` are
  ported under their Rails names, covering both `nextMigrationNumber`'s
  sequential branch and `isValidateTimestamp`'s conjunct.
- Each new test fails when the corresponding guard is reverted.
- `pnpm parity:test` matched count does not drop.
