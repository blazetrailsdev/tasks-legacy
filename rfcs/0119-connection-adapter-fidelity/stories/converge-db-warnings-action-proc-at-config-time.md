---
title: "Bake db_warnings_action into a callable at config time instead of branching in handle_warnings"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while converging `handle_warnings`' call arguments in PR #6418.

Rails turns `ActiveRecord.db_warnings_action = :raise` into a Proc ONCE, at
config time (`vendor/rails/activerecord/lib/active_record.rb:236-252`), so the
warning path is a single dispatch:

```ruby
# vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/database_statements.rb:216-222
def handle_warnings(sql)
  @notice_receiver_sql_warnings.each do |warning|
    next if warning_ignored?(warning)
    warning.sql = sql
    ActiveRecord.db_warnings_action.call(warning)
  end
end
```

trails stores the SYMBOL on the adapter class
(`packages/activerecord/src/connection-adapters/postgresql/database-statements.ts#handleWarnings`)
and re-derives the behaviour at every warning: an `ignore` / `raise` / `log` /
`report` if-ladder inline in the loop, with the Rails call reached only in the
`typeof action === "function"` arm. So the symbol→behaviour mapping lives in the
adapter instead of in the setter, and `handle_warnings` carries four branches
Rails' body does not have.

## Converged shape

`ActiveRecord.dbWarningsAction=` bakes the symbol into a callable once, the way
`active_record.rb:236-252` does, and `handleWarnings` keeps only Rails' three
lines: the `warning_ignored?` guard, the `warning.sql = sql` assignment, and the
single dispatch. The `:report` arm's `ActiveSupport.errorReporter.report(warning,
{ handled: true })` and the `:log` arm's message format move into that setter
verbatim.

## Acceptance criteria

- [ ] The symbol→behaviour mapping lives in the `db_warnings_action` setter, not
      in `handleWarnings`.
- [ ] `handleWarnings`' body mirrors `database_statements.rb:216-222` branch for
      branch.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green with no new
      rows.
- [ ] The PG warning suites stay green (this arm only runs on the postgres lane).
