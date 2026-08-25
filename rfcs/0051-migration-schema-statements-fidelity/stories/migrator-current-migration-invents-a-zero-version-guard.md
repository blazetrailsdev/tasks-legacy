---
title: "Migrator#currentMigration invents a version-0 guard and reads the private list"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 19
pr: 7020
claim: "2026-08-25T00:30:08Z"
assignee: "relocate-model-name-to-naming-module"
blocked-by: null
closed-reason: null
---

## Context

`Migrator#currentMigration`
(`packages/activerecord/src/migration.ts:2796-2800`) adds a guard Rails does not
have, and reads a different collection:

```ts
async currentMigration(): Promise<MigrationProxy | null> {
  const version = await this.currentVersion();
  if (version === 0) return null;
  return this._migrations.find((m) => m.version === version) ?? null;
}
```

Rails is a bare `detect` over the _public_ `migrations` reader
(`vendor/rails/activerecord/lib/active_record/migration.rb:1439-1441`):

```ruby
def current_migration
  migrations.detect { |m| m.version == current_version }
end
alias :current :current_migration
```

Two divergences:

1. **The `version === 0` short-circuit.** Rails has no such branch. A migration
   whose version genuinely is `0` is findable in Ruby and unreachable here.
   Version 0 is a real value in this codebase — `migrator.test.ts`'s
   `trackedSensor(null, 0)` builds one, and `MigrationContext#move`
   (`migration.rb:1386-1401`) passes `0` as a target.
2. **`this._migrations` vs `this.migrations`.** The getter is
   `down? ? @migrations.reverse : @migrations.sort_by(&:version)`
   (`migration.rb:1470-1472`); the private field is neither reversed nor
   sorted. `detect` returns the _first_ match in that order, so on a `:down`
   migrator the two can differ.

Surfaced while converging `Migrator` onto its `migration.rb:1404-1620` surface
in PR #6982, which deliberately did not touch this method — it is pre-existing
and `MigrationContext#move` currently depends on the null:

```ts
const currentMigration = await migrator.currentMigration();
if (currentVersion !== 0 && !currentMigration) {
  throw new UnknownMigrationVersionError(currentVersion);
}
```

so converging `currentMigration` means re-reading `move` (`migration.rb:1386-1401`)
in the same pass.

## Converged shape

```ts
async currentMigration(): Promise<MigrationProxy | null> {
  const currentVersion = await this.currentVersion();
  return this.migrations.find((m) => m.version === currentVersion) ?? null;
}
```

and `#move`'s guard re-derived from `migration.rb:1386-1401` rather than from
the removed short-circuit.

## Acceptance criteria

- [ ] `currentMigration` has no `version === 0` branch and reads the public
      `migrations` getter, matching `migration.rb:1439-1441`.
- [ ] `MigrationContext#move` still raises `UnknownMigrationVersionError` where
      `migration.rb:1386-1401` does, with the guard traced to the Ruby rather
      than to `currentMigration`'s old null.
- [ ] `migrator.test.ts` (`migrator forward`, `migrator rollback`) and
      `multi-db-migrator.test.ts` (`migrator forward`) keep their names and pass.
