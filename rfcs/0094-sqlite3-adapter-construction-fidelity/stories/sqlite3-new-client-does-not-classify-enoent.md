---
title: "SQLite3Adapter.new_client does not classify Errno::ENOENT at open time"
status: draft
updated: 2026-08-25
rfc: "0094-sqlite3-adapter-construction-fidelity"
cluster: null
packages: []
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

Rails classifies a missing database file at _open_ time, inside `new_client`:

```ruby
# activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:34-42
def new_client(config)
  ::SQLite3::Database.new(config[:database].to_s, config)
rescue Errno::ENOENT => error
  if error.message.include?("No such file or directory")
    raise ActiveRecord::NoDatabaseError
  else
    raise
  end
end
```

trails' `SQLite3Adapter.newClient`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`) constructs
an _adapter_ rather than opening a file, so there is no open-time errno to
classify; the missing-database mapping happens later, in `translateException`.
A caller that uses `newClient` directly therefore gets a different error class
than Rails hands back.

This is the fifth of the five sqlite3 call-set rows PR #7009 retired. The other
four are the construction-shape rows already owned by
`sqlite3-connection-parameters-never-built` and
`retire-sqlite3-positional-constructor-overload`; this one is separate — it
survives even after `@connection_parameters` exists, because it is about _where_
the errno arm lives. PR #7009 left a CONVERGEABLE `@missingRailsCall include?`
receipt on `newClient`; this story retires it.

Depends on the construction convergence: `new_client` cannot raise at open time
until it is what actually opens the driver handle (`sqlite3_adapter.rb:807`).

## Acceptance criteria

- [ ] `newClient` opens the driver and raises `NoDatabaseError` for the
      missing-file errno, re-raising anything else — the `include?("No such
file or directory")` discrimination at Rails' site.
- [ ] `translateException` no longer carries the missing-database mapping that
      only existed to cover for the open path, unless Rails has it there too.
- [ ] The `@missingRailsCall include?` receipt on `newClient` is deleted.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
