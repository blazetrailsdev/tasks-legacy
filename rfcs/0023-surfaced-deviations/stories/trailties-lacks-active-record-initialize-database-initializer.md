---
title: "trailties has no active_record.initialize_database initializer"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails boots Active Record's configuration through the Railtie initializer
`active_record.initialize_database`
(`vendor/rails/activerecord/lib/active_record/railtie.rb:256-262`):

```ruby
initializer "active_record.initialize_database" do
  ActiveSupport.on_load(:active_record) do
    self.configurations = Rails.application.config.database_configuration

    establish_connection
  end
end
```

That is the _only_ place `config/database.yml` reaches `Base.configurations` in
a booted app, and it is why `establish_connection`'s no-arg arm
(`connection_handling.rb:50-54`) can resolve purely from the registry without
ever touching the filesystem.

trails has no equivalent initializer. PR #6777 converged the AR half — it
deleted `autoConnect`'s `loadConfigFile` disk fallback and `Base._configPath`,
so no-arg `establishConnection` now takes the Rails funnel and raises
`AdapterNotSpecified` on an empty registry — and it landed the config-reading
half as `databaseConfiguration()` in `packages/trailties/src/database.ts`, the
seat for `Rails::Application::Configuration#database_configuration`
(`railties/lib/rails/application/configuration.rb:434-468`).

What is missing is the wire between them. Nothing in the trailties boot path
(`packages/trailties/src/application/bootstrap.ts`,
`packages/trailties/src/application/finisher.ts`,
`packages/trailties/src/trailtie.ts`) calls `databaseConfiguration()` and
assigns `Base.configurations`. Today the only callers are the CLI
(`packages/trailties/src/commands/db.ts`, via `loadDatabaseConfig`) and
`packages/trailties/src/database.test.ts`.

Consequence flagged in the #6777 review: a real trails app that calls a bare
`Base.establishConnection()` gets `AdapterNotSpecified` unless something
explicitly primes `Base.configurations` first, and no such wiring exists in the
repo. This matches how plain Active Record behaves _outside_ a Rails app, so it
is not a regression — but inside a booted trails app it is a gap, and the
initializer is the Rails-shaped fix.

## Acceptance criteria

- A trailtie initializer named `active_record.initialize_database` exists and
  mirrors railtie.rb:256-262: it assigns
  `Base.configurations(await databaseConfiguration())` and then calls
  `establishConnection()`, in that order.
- It runs from the trailties boot path, registered the way the other
  initializers in `packages/trailties/src/` are, not called ad hoc.
- Booting an app with a `config/database.*` and then calling a bare
  `Base.establishConnection()` connects, rather than raising
  `AdapterNotSpecified`.
- An app with no config file and no `DATABASE_URL` still surfaces Rails' error
  from `databaseConfiguration()` (`Could not load database configuration`).
- The CLI's `loadDatabaseConfig` path is either routed through the same
  initializer or left alone deliberately, with the reason stated — it must not
  end up as a second, divergent copy of the boot wiring.

## Notes

Related, and worth landing near this one:
`0112-one-rails-thing-n-trails-things/database-configuration-shared-key-reverse-merge`
(the `shared`-key reverse-merge branch of configuration.rb:439-458 that
`databaseConfiguration()` does not yet port).
