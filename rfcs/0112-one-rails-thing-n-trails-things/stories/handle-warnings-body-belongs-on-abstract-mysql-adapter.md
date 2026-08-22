---
title: "handle_warnings body sits on Mysql2Adapter while its warning_ignored? guard sits on AbstractMysqlAdapter, where Rails puts both"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 130
pr: 6772
claim: "2026-08-20T01:54:44Z"
assignee: "handle-warnings-body-belongs-on-abstract-mysql-adapter"
blocked-by: null
closed-reason: null
---

## Context

Baselined in PR #6577 (RFC 0106 wave 3b): four rows —
`handle_warnings | query`, `| new`, `| warning_ignored?`, `| call` — in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`.

Rails defines `handle_warnings` on `AbstractMysqlAdapter`
(`abstract_mysql_adapter.rb:770-784`), reading `@raw_connection` directly:

    def handle_warnings(sql)
      return if ActiveRecord.db_warnings_action.nil? || @raw_connection.warning_count == 0

      warning_count = @raw_connection.warning_count
      result = @raw_connection.query("SHOW WARNINGS")
      result = [["Warning", nil, "Query had warning_count=... "]] if result.count == 0
      result.each do |level, code, message|
        warning = SQLWarning.new(message, code, level, sql, @pool)
        next if warning_ignored?(warning)
        ActiveRecord.db_warnings_action.call(warning)
      end
    end

trails leaves `AbstractMysqlAdapter#handleWarnings`
(`abstract-mysql-adapter.ts:1590`) as a two-line delegation to a `_handleWarnings`
hook, and puts the whole body on `Mysql2Adapter#handleWarnings`
(`mysql2-adapter.ts:1767`), because the raw mysql2 connection is reached there
as `this._client`.

`warning_ignored?` (rb:786-788) IS correctly on the abstract class in trails
(`isWarningIgnored`, abstract-mysql-adapter.ts:1600), so the file is
inconsistent with itself: the guard is at the Rails location and the body it
guards is one class down.

Related, already done: `mysql2-handle-warnings-drops-conn-parameter` (RFC 0076).

## Converged shape

The `handle_warnings` body lives on `AbstractMysqlAdapter` at the Rails
location, reading the raw connection through whatever seam RFC 0076 settles for
`@raw_connection` on this class. `Mysql2Adapter` keeps only what is genuinely
driver-specific (the `warning_count` read, if the mysql2 client exposes it
differently), and the `_handleWarnings` indirection — which has no Rails
counterpart — is retired.

Also fold in the two TODOs left in the current body: `Rails.error.report(warning,
handled: true)` is unwired (PG's `handleWarnings` already does it), and the
`ActiveRecord.db_warnings_action` dispatch is spelled as a local `action`
variable with string arms rather than Rails' single `.call`.

## Acceptance criteria

- [ ] `handleWarnings` body sits on `AbstractMysqlAdapter`, mirroring rb:770-784
      line for line, same early return and same `next if warning_ignored?` guard.
- [ ] `_handleWarnings` is gone (it has no Rails counterpart).
- [ ] The four `handle_warnings` rows deleted by hand from the shard (no reseed).
- [ ] `db_warnings_action` `:raise` / `:log` / proc arms keep their existing tests.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Absorbed: `mysql-drop-table-temporary-body-belongs-on-abstract-mysql-adapter`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Move MySQL dropTable TEMPORARY body to AbstractMysqlAdapter so the mixed-in companion dropTable isn't dead"

### Context

Rails puts the `DROP TEMPORARY TABLE` handling in `AbstractMysqlAdapter#drop_table`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb`),
so every MySQL-family adapter inherits it. trails instead defines the whole
`dropTable` body — including the `TEMPORARY` keyword, `IF EXISTS`, `CASCADE`,
and the schema-cache invalidation loop — in the concrete `Mysql2Adapter` class
body (`connection-adapters/mysql2-adapter.ts:1450-1473`), and has
`MysqlSchemaStatements#dropTable`
(`connection-adapters/mysql/schema-statements.ts:79-95`) delegate back to
`this.adapter.dropTable(...)` for the `temporary: true` case.

PR #5490 made `MysqlSchemaStatements` a real mixin on `AbstractMysqlAdapter`
(`include MySQL::SchemaStatements`, `abstract_mysql_adapter.rb:19`). Verified at
runtime after that change: of the companion's four own prototype members
(`schemaCreation`, `addIndex`, `removeColumn`, `dropTable`), `dropTable` is the
one that is mixed onto `AbstractMysqlAdapter.prototype` but permanently shadowed
by `Mysql2Adapter`'s class-body `dropTable`, so the mixed-in copy — and its
`temporary` delegation branch — is dead code on every instantiated adapter.

That dead branch is also a latent recursion trap: if a future MySQL-family
adapter is added without its own class-body `dropTable`, the delegation would
dispatch back to itself. #5490 considered a defensive `adapter !== this` guard
there and deliberately dropped it, because the guard would have silently emitted
a non-`TEMPORARY` `DROP TABLE` rather than fail loudly — fixing the layout is the
right resolution, not guarding the dead path.

Related: [[converge-schema-statements-companion-onto-mixin]] tracks retiring the
`schemaStatements()` companion accessor generally; this story is the narrower
MySQL `dropTable` layout move and can land independently.

### Acceptance criteria

- [ ] `dropTable`'s MySQL-family body (TEMPORARY/IF EXISTS/CASCADE + schema-cache
      invalidation) lives where Rails puts it — `AbstractMysqlAdapter`, mirroring
      `abstract_mysql_adapter.rb#drop_table` — instead of `Mysql2Adapter`.
- [ ] The `temporary`-only delegation back to `this.adapter.dropTable(...)` in
      `MysqlSchemaStatements#dropTable` is removed or reduced to the Rails shape,
      with no path that can dispatch to itself.
- [ ] Existing `dropTable`/temporary-table coverage still passes on mysql2, and
      sqlite3/postgresql are unaffected.
- [ ] `pnpm parity:test --package activerecord --gates --check` exits 0.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
