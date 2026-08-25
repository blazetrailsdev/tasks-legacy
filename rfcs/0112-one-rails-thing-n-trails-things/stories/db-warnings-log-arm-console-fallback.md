---
title: "db-warnings-log-arm-console-fallback"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
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

`ActiveRecord.dbWarningsAction`'s `"log"` arm
(`packages/activerecord/src/ar-config.ts`) mirrors
`active_record.rb:243-246`:

```ruby
->(warning) do
  warning_message = "[#{warning.class}] #{warning.message}"
  warning_message += " (#{warning.code})" if warning.code
  ActiveRecord::Base.logger.warn(warning_message)
end
```

Two deviations survive there, both inherited from the per-adapter handlers this
arm replaced (PR #7055, RFC 0112):

1. **`console.warn` fallback.** When `ActiveRecord::Base.logger` is nil Rails
   raises `NoMethodError`; trails falls back to `console.warn`. The fallback is
   load-bearing for the PG case `logs warnings when behaviour log`
   (`adapters/postgresql/postgresql-adapter.test.ts`), which spies on
   `console.warn` rather than setting `Base.logger`.
2. **`"[ActiveRecord::SQLWarning]"` is a literal**, where Rails interpolates
   `warning.class` — so a `SQLWarning` subclass renders as its parent.

The PG warning cases also set the global directly
(`ActiveRecord.dbWarningsAction = "log"` + a `beforeEach`/`afterEach`
save/restore) where Rails wraps each body in
`with_db_warnings_action(:log) do … end` (test_case.rb:164); trails already has
`support/with-db-warnings-action.ts`. Rails' log case asserts through
`assert_called_with(ActiveRecord::Base.logger, :warn, [sql_warning])`
(postgresql_adapter_test.rb:631-639), which is what makes the fallback
unnecessary there.

## Acceptance criteria

- [ ] The `"log"` arm calls `getBase().logger.warn(...)` with no `console.warn`
      fallback, and interpolates the warning's own class name.
- [ ] The five PG warning cases wrap their bodies in `withDbWarningsAction`,
      dropping the bespoke save/restore, and the log case asserts on a
      `Base.logger` double as Rails does.
- [ ] mysql2's `warnings_test.rb` counterparts stay green on the MySQL lane.
