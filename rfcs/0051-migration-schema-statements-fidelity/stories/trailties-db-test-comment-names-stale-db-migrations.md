---
title: "trailties-db-test-comment-names-stale-db-migrations"
status: claimed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 5
priority: null
pr: null
claim: "2026-08-24T14:19:54Z"
assignee: "website-vitest-missing-activesupport-subpath-alias"
blocked-by: null
closed-reason: null
---

## Context

`packages/trailties/src/commands/db.test.ts:2088` still says
"Primary migrations in db/migrations; animals in db/migrate_animals." after
PR #6977 converged the default migration directory to Rails' `db/migrate`
(`Migrator.migrations_paths`,
`vendor/rails/activerecord/lib/active_record/migration.rb:1419`). The setup the
comment describes now lays migrations under `db/migrate`, so the comment names
a directory the test no longer uses.

## Acceptance criteria

- [ ] The comment names the directory the test actually writes (`db/migrate`).
- [ ] No `db/migrations` left in `packages/trailties/src` except the
      deliberately-custom `migrationsPaths: "db/migrations_test_env"` at
      `db.test.ts:1831`.
- [ ] Test names unchanged.
