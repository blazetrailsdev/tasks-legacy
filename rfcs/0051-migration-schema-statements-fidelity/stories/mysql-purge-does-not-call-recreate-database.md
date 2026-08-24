---
title: "mysql-purge-does-not-call-recreate-database"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`MySQLDatabaseTasks#purge` (`mysql_database_tasks.rb:29-31`) is:

```ruby
def purge
  establish_connection(configuration_hash.merge(database: nil))
  connection.recreate_database(db_config.database, creation_options)
end
```

trails' `packages/activerecord/src/tasks/mysql-database-tasks.ts:105-113` instead
reads the live charset/collation (`savedCharset()`), then calls `drop()` and
`create(saved)` — no `recreate_database` call at all, and a creation-options set
computed from the live database rather than from `creation_options`
(`mysql_database_tasks.rb:79-85`, which is `{ charset:, collation: }` off the
configuration hash).

Surfaced while enrolling `mysql2_rake_test.rb` in `parity:test` (PR for
`enroll-pg-and-mysql-rake-tests-in-test-compare`). All three `MySQLPurgeTest`
tests assert on `recreate_database` and are therefore parked as `it.skip` in
`packages/activerecord/src/adapters/mysql2/mysql2-rake.test.ts`:

- `establishes connection without database` (`mysql2_rake_test.rb:186`)
- `recreates database with no default options` (`:196`)
- `recreates database with the given options` (`:204`)

The in-file comment on the trails deviation says the live-charset read exists so
purge does not silently change collation under MySQL 8 defaults; that rationale
has to be re-established against Rails' shape (Rails re-creates from
`creation_options`), not preserved by assumption.

## Acceptance criteria

- [ ] `MySQLDatabaseTasks#purge` converges on `mysql_database_tasks.rb:29-31`:
      `establishConnection(configurationHash without database)` then
      `recreateDatabase(db_config.database, creationOptions())`.
- [ ] `recreateDatabase` exists on the mysql2 adapter at the Rails name
      (`abstract_mysql_adapter.rb#recreate_database`).
- [ ] The three `MySQLPurgeTest` `it.skip` stubs in
      `packages/activerecord/src/adapters/mysql2/mysql2-rake.test.ts` become
      real tests at their Rails names; `pnpm parity:test` gate-mismatch stays 0.
- [ ] Green on the MariaDB lane — in particular the case-sensitivity tests the
      current charset-preservation comment names.
