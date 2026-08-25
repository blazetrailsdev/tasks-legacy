---
title: "MigrationProxy memoizes a rejected load promise; Rails memoizes only on success"
status: draft
updated: 2026-08-01
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`MigrationProxy.migration()`
(`packages/activerecord/src/deprecator.ts:82-85`) memoizes the _promise_:

```ts
this._migrationPromise ??= this.loadMigrationAsync().then((m) => (this._migration = m));
```

A rejected promise stays cached, so every later `migration()` call replays the
same rejection without re-running the load. Rails' `MigrationProxy#migration`
(`vendor/rails/activerecord/lib/active_record/migration.rb:1190-1192`) is
`@migration ||= load_migration` — memoized only on success, so a failed load is
retried on the next call. PR #5781 (`Migrator#_runMigration`) relies on this
re-resolution: its `catch` re-invokes `proxy.migration()` exactly as Rails'
`use_transaction?` does at `migration.rb:1540`, and in Rails that triggers a
genuine second `load_migration`.

The observable outcome matches today (both raise the load error out of the
rescue unwrapped), so this is latent, not a live bug. It diverges only when a
load failure is transient — e.g. a file made valid between attempts, or a load
that raises on first evaluation but succeeds on re-require.

## Acceptance criteria

- [ ] `MigrationProxy.migration()` memoizes only successful loads; a rejection
      clears the cached promise so the next call re-runs `loadMigrationAsync`,
      matching `@migration ||= load_migration`.
- [ ] A test asserts that a factory failing once and succeeding on the second
      call is loaded twice and ultimately succeeds.
- [ ] `Migrator#_runMigration`'s catch-path re-resolution
      (`packages/activerecord/src/migration.ts`) still raises the load error
      unwrapped after the change.
