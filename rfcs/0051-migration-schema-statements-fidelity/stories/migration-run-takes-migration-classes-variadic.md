---
title: "Migration#run takes one instance where Rails takes migration classes"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 52
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `Migration#run` takes one instance where Rails takes migration classes

## Context

Rails' `Migration#run(*migration_classes)`
(`vendor/rails/activerecord/lib/active_record/migration.rb:937-949`) takes a
splat of migration **classes**, pops the options hash off the end with
`extract_options!`, and calls `migration_class.new.exec_migration(connection, dir)`
on each.

The port (`packages/activerecord/src/migration.ts:1210-1240`) takes a single
already-constructed `Migration` instance plus an explicit `opts` object, and
calls `migration.execMigration(...)` on it. `Migration#revert` (migration.rb:853)
passes `*migration_classes.reverse`; the port's `revert` correspondingly forwards
one instance, so the reverse-order semantics of a multi-class `revert` are not
ported either.

Surfaced by the review of #6698, which renamed `_run` → `run` (converging the
callee name for the call-set gate) and deleted the invented
`run(adapter, direction)` shim that had been squatting on the name, but left the
signature alone as out of scope for that story.

## Acceptance criteria

- `Migration#run` accepts a variadic list of migration classes with a trailing
  options object, constructs each (`migration_class.new`), and calls
  `execMigration(connection, dir)` per Rails' loop.
- `Migration#revert(*migrationClasses)` forwards them reversed, as
  migration.rb:853 does.
- Existing callers and tests updated; `pnpm parity:api --arity` shows no new
  mismatch.
