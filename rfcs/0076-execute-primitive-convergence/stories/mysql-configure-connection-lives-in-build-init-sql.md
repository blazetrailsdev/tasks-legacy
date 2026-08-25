---
title: "AbstractMysqlAdapter#configure_connection is unported; its body lives in Mysql2Adapter#_buildInitSql as a driver initSql string"
status: draft
updated: 2026-08-25
rfc: "0076-execute-primitive-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced converging `strict_mode?` in PR #7059 (RFC 0112 story
`mysql-strict-mode-predicate-hardcoded-false`). That PR routed the sql_mode
decision through the Rails-named predicate, but the body it routed into is
still not `configure_connection`.

Rails puts the whole session bootstrap in one method,
`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:912-936`:

```ruby
def configure_connection
  variables = @config.fetch(:variables, {}).stringify_keys
  variables["wait_timeout"] = self.class.type_cast_config_to_integer(variables["wait_timeout"] || 2147483)
  defaults = [":default", :default].to_set
  if sql_mode = variables.delete("sql_mode")
    sql_mode = quote(sql_mode)
  elsif !defaults.include?(strict_mode?)
    ...
  end
  sql_mode_assignment = "@@SESSION.sql_mode = #{sql_mode}, " if sql_mode
  # NAMES ... , then the variable assignments, then:
  internal_execute("SET #{encoding} #{sql_mode_assignment} #{variable_assignments}", ...)
end
```

trails has no `AbstractMysqlAdapter#configureConnection` at all. The same logic
lives in `Mysql2Adapter#_buildInitSql`
(`packages/activerecord/src/connection-adapters/mysql2-adapter.ts:1639`), a
trails-only `private` method that RETURNS the SET statement as a string so it
can be handed to the driver as `initSql` at connect time
(`mysql2-adapter.ts:623`) rather than executed on an established connection.

Consequences beyond the name:

- The Rails method is a public, overridable seam (`reconfigure_connection`
  paths, subclass hooks); a `private _buildInitSql` is neither.
- It builds a string per connect instead of issuing `internal_execute`, so the
  statement never passes through the adapter's logging/instrumentation.
- `wait_timeout` is coerced by hand instead of by
  `type_cast_config_to_integer` (rb:914), and the `variables` read does not go
  through `@config.fetch(:variables, {})`.

## Converged shape

Port `AbstractMysqlAdapter#configureConnection` onto the abstract class at the
Rails name, mirroring rb:912-936 line for line, and let the mysql2 connect path
call it after the connection is established — the way
`postgresql-adapter.ts` already calls its `configureConnection`. `_buildInitSql`
then disappears, or shrinks to whatever genuinely must reach the driver before
the first statement (charset/`SET NAMES`), with a call-site citation for the
residue.

Coordinate with RFC 0076's PG `configure_connection` stories, which are
converging the same method on the other adapter.

## Acceptance criteria

- [ ] `AbstractMysqlAdapter#configureConnection` exists at the Rails name and
      mirrors `abstract_mysql_adapter.rb:912-936`, including
      `type_cast_config_to_integer` for `wait_timeout` (rb:914) and the
      `@config.fetch(:variables, {})` read (rb:913).
- [ ] The session SET is issued through the adapter's execute path, not handed
      to the driver as an `initSql` string, except for any residue that
      provably must precede the first statement — justified at the call site.
- [ ] `_buildInitSql` is deleted or reduced to that residue.
- [ ] `mysql2-adapter.build-init-sql.trails.test.ts` follows the method it
      covers; the strict-mode arms (default-true, stored-false, `":default"`)
      stay green.
- [ ] MySQL/MariaDB lanes green; `pnpm parity:api:calls` and
      `parity:api:extra --package activerecord` show no new rows.
