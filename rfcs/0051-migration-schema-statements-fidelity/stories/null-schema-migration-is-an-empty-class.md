---
title: "NullSchemaMigration carries five invented no-op members; Rails' is an empty class"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 33
pr: 7022
claim: "2026-08-25T00:54:07Z"
assignee: "split-model-mixin-surface-to-active-model-model"
blocked-by: null
closed-reason: null
---

## Context

Exact sibling of `null-internal-metadata-is-an-empty-class` (RFC 0051, done
in PR #6264 / #6263), surfaced while closing it — the two null objects are declared
side by side in Rails and diverge in trails the same way.

Rails' `NullSchemaMigration` is an **empty class**
(`activerecord/lib/active_record/schema_migration.rb:9-10`):

```ruby
class SchemaMigration # :nodoc:
  class NullSchemaMigration # :nodoc:
  end
```

trails' (`packages/activerecord/src/schema-migration.ts:30-45`) instead carries
five short-circuiting members with no Ruby counterpart:

```ts
export class NullSchemaMigration {
  async createTable(): Promise<void> {}
  async dropTable(): Promise<void> {}
  async allVersions(): Promise<string[]> {
    return [];
  }
  async count(): Promise<number> {
    return 0;
  }
  async tableExists(): Promise<boolean> {
    return false;
  }
}
```

Same defect the InternalMetadata story fixed: methods that answer a comfortable
lie (`tableExists() === false` against a table that may physically exist,
`count() === 0` against a populated `schema_migrations`) rather than the truth.
Because every method succeeds, a caller cannot tell the null object apart by
behavior, so the branch Rails expects callers to make never has to be written.

Rails hands it out from `Migration.copy` (`migration.rb:1065`) and for a
`NullPool`; callers branch on which class they got.

## Converged shape

Empty the class to match `schema_migration.rb:9-10`, then fix whatever call
sites relied on the silent no-ops — they should branch on the class
(`instanceof NullSchemaMigration`) or hold a real `SchemaMigration`, as Rails'
callers do.

Note the InternalMetadata sibling turned out to have **zero** consumers in the
repo, so emptying it needed no call-site changes at all. Check the same here
before assuming fallout: grep for `NullSchemaMigration` across `packages/`
first. It is currently exported from `activerecord/src/index.ts`.

## Acceptance criteria

- [ ] `NullSchemaMigration` declares no members (`schema_migration.rb:9-10`).
- [ ] Call sites that relied on the no-op methods branch on the class instead,
      as Rails' do.
- [ ] `pnpm parity:api:extra --package activerecord` reports no novel names on
      `schema-migration.ts` for this class.
