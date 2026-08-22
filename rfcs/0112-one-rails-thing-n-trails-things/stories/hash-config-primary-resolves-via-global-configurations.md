---
title: "HashConfig#primary? should read the global configurations registry, not a _primaryChecker global"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 90
pr: 6811
claim: "2026-08-21T11:40:36Z"
assignee: "hash-config-primary-resolves-via-global-configurations"
blocked-by: null
closed-reason: null
---

## Context

Rails' `HashConfig#primary?` is one line — it asks the global registry:

```ruby
def primary? # :nodoc:
  Base.configurations.primary?(name)
end
```

(`vendor/rails/activerecord/lib/active_record/database_configurations/hash_config.rb:129-130`,
delegating to `DatabaseConfigurations#primary?` at
`database_configurations.rb:142-147`, which is `name == "primary" ||
name == find_db_config(default_env)&.name`.)

trails cannot reach `Base` from `hash-config.ts` (cycle), so it reimplemented
the lookup as a module-global indirection: `_primaryChecker`, installed at
`database-configurations.ts:496` against a second module global,
`_currentConfigurations` (`database-configurations.ts:13`, mutated as a side
effect of the `DatabaseConfigurations` constructor at `:119` and `:135`).
`HashConfig.isPrimary()` (`hash-config.ts:33-36`) consults that instead of the
registry.

This is now removable. PR #5386 (story
`converge-configurations-storage-onto-a-single-global-registry`) moved
`configurations` onto a single process-global registry in `core.ts` with a
receiver-less accessor, which is exactly the `Base.configurations` that Rails'
`primary?` reads — and importing it from `core.js` does not close a cycle the
way importing `base.js` would (`connection-handling.ts` already does this).

The `_currentConfigurations` global is also test-visible debt: suites save and
restore `(DatabaseConfigurations as any).current` around anything that builds
configs (`connection-handling.test.ts:486`, `:519`, `:722`,
`connection-handlers-sharding-db.test.ts`, `connection-swapping-nested.test.ts`)
purely because the constructor mutates it. Rails has no `current` and no such
dance.

PR #5512 moved the registry's backing store out of `core.ts` into
`database-configurations.ts` (`configurationsStore()` / `setConfigurationsStore()`,
both `@internal`) so `ConnectionHandler` could read it without an import edge to
`core.ts`. That makes this story strictly smaller: `hash-config.ts`' checker can
now read the store from the module it already sits next to, with no `core.js`
import at all.

That PR also made `setConfigurationsStore` assign `_currentConfigurations`, so
`Base.configurations=` re-registers the primary checker. The pair is still
asymmetric in the other direction: the `current` setter
(`database-configurations.ts:156-158`) and the constructor / `fromRaw`
registrations (`:173`, `:189`) write only `_currentConfigurations`, so any
`new DatabaseConfigurations(...)` silently repoints `primary?` without changing
`Base.configurations()`. Collapsing the shim into the store removes both
directions at once.

## Acceptance criteria

- `HashConfig#isPrimary` resolves through the global `configurations` registry,
  matching `Base.configurations.primary?(name)`.
- `_primaryChecker` / `_setPrimaryChecker` are deleted.
- `DatabaseConfigurations.current` / `_currentConfigurations` is deleted, or —
  if some other consumer still needs it — the story is reduced to the
  `primary?` half and the remaining consumers are named in a follow-up.
  Rails has no counterpart for either.
- The `(DatabaseConfigurations as any).current` save/restore blocks in the
  suites above are removed along with it.
- No `base.js` or `core.js` import from `database-configurations*` — read
  `configurationsStore()` in `database-configurations.ts` directly.
- Suites pass: database-configurations, connection-handling, shard-keys,
  test-databases, connection-swapping-nested, connection-handlers-\*,
  schema-dumper (schema_dump / seeds read `primary?`).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
