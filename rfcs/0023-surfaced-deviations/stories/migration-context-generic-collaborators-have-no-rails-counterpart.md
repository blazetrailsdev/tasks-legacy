---
title: "MigrationContext's collaborator type parameters and this-annotations have no Rails counterpart"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Shipped by PR #6281 (`migration-context-collaborator-readers-cast-away-the-null-object`)
and tracked here so it stays on the ledger.

`MigrationContext#initialize` (`vendor/rails/activerecord/lib/active_record/migration.rb:1214-1217`) is:

```ruby
def initialize(migrations_paths, schema_migration = nil, internal_metadata = nil)
  @migrations_paths = migrations_paths
  @schema_migration = schema_migration || SchemaMigration.new(connection_pool)
  @internal_metadata = internal_metadata || InternalMetadata.new(connection_pool)
end
```

Three plain positional parameters, two `nil` defaults, and an `attr_reader`
(`migration.rb:1212`) that hands back whatever was seated — including the null
objects `Migration.copy` seats (`migration.rb:1065-1066`,
`schema_migration.rb:9`, `internal_metadata.rb:13`).

trails now spells this as a class generic over its two collaborators plus a
conditional rest tuple (`packages/activerecord/src/migration.ts`,
`type SeatedCollaborators<S, I>` and
`constructor(migrationsPaths: string[], ...seated: SeatedCollaborators<S, I>)`),
with `this: MigrationContext` annotations on the fifteen connected methods
(`migrate` / `up` / `down` / `run` / `open` / `migrationsStatus` /
`protectedEnvironment` / `lastStoredEnvironment` / `getAllVersions` /
`currentVersion` / `needsMigration` / `pendingMigrationVersions` / `rollback` /
`forward` / `move`). Rails has none of it: no type parameters, no receiver
annotations, no arity-shaping tuple.

The machinery is standing in for Ruby duck typing — Rails needs no static
distinction between a discovery-only context and a connected one because a null
object simply never receives a `SchemaMigration` message. It was reviewed and
accepted on #6281 as the only shape that keeps the readers honest (the previous
`as SchemaMigration` cast at the seat could hand a real collaborator back
through a null-object reader), but it is deviation surface, not Rails.

## Converged shape

Whatever removes the type machinery without reintroducing a reader that can
claim a type the slot does not hold. Candidates, none costed:

- A narrower runtime shape that makes the null objects unnecessary in the first
  place — e.g. if the two call sites that build a discovery-only context
  (`Migration.copy`, activerecord-cli's `loadMigrations`) could read migrations
  off something that is not a `MigrationContext`, the null objects and the whole
  generic go away. Note this diverges from Rails in the other direction, since
  Rails routes both through `MigrationContext`.
- A TS feature that expresses "when this argument is omitted, its type parameter
  is at its default", which is exactly what `SeatedCollaborators` works around.
  If a future TS release lands it, the tuple collapses back to two optional
  parameters and the arity matches Ruby's three verbatim.

Do not close this by widening a reader back to a union or by restoring the cast.

## Acceptance criteria

- [ ] `MigrationContext` carries no type parameters, no `SeatedCollaborators`,
      and no `this: MigrationContext` annotations, OR the story is blocked with
      the specific TS limitation that keeps them.
- [ ] The readers still hand back exactly what `initialize` seated, with no cast.
- [ ] `Migration.copy` still seats the null objects verbatim as
      `migration.rb:1065-1066` does.
- [ ] The four `@ts-expect-error` pins in `migration-context.trails.test.ts`
      are removed or replaced, not left alongside.
