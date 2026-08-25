---
title: "Browser CLI's MigrationContext override re-introduces the retired hyphen filename alias"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 50
priority: 20
pr: 7032
claim: "2026-08-25T12:58:54Z"
assignee: "split-model-mixin-surface-to-active-model-model"
blocked-by: null
closed-reason: null
---

## Context

RFC 0051's `retire-the-hyphen-migration-filename-alias` is **done**: trails
migration filenames use `_` as the version/name separator, matching Rails'

```ruby
/\A([0-9]+)_([_a-z0-9]*)\.?([_a-z0-9]*)?\.rb\z/
```

(`vendor/rails/activerecord/lib/active_record/migration.rb:1374-1376`), which
`MigrationContext#parseMigrationFilename`
(`packages/activerecord/src/migration.ts:2364-2369`) mirrors with `ts|js` for
`rb` — underscore only, no hyphen arm.

The browser CLI still carries the retired alias. `VfsMigrationContext` in
`packages/website/src/lib/frontiers/trails-cli.ts` overrides
`parseMigrationFilename` purely to widen the separator back out:

```ts
const MIGRATION_FILE_PATTERN = /^(\d+)[-_](.+)\.(?:ts|js)$/;

protected override parseMigrationFilename(filename: string): [string, string, string] | null {
  const basename = filename.split("/").pop() ?? "";
  const match = basename.match(MIGRATION_FILE_PATTERN);
  if (!match) return null;
  return [match[1], match[2].replace(/-/g, "_"), ""];
}
```

Three divergences from the base method it overrides:

1. `[-_]` where Rails is `_`, then `replace(/-/g, "_")` to undo it — the exact
   alias that was retired.
2. `(.+)` where Rails is `([_a-z0-9]*)`, so it accepts characters Rails rejects.
3. The scope capture is hardcoded `""`, dropping Rails' third group
   (`migration.rb:1375`) — so a scoped virtual migration silently loses its
   scope.

The website's own generator writes `db/migrate/<timestamp>_<name>.ts` through
`MigrationGenerator` (`packages/trailties/src/generators/migration-generator.ts:227`),
i.e. underscores, so the hyphen arm is dead weight for anything the sandbox
generates; only hand-authored tutorial fixtures could rely on it. Those live in
`packages/website/src/lib/frontiers/tutorials/generator-fixtures.ts` and already
use `*_create_users.ts` spellings.

Surfaced in PR #6982, which introduced `VfsMigrationContext` to route the
browser CLI's run surface through `MigrationContext`; the override was carried
over verbatim from the `discoverMigrations` function it replaced rather than
re-derived, so the retired alias came with it.

## Converged shape

`parseMigrationFilename` is not overridden at all — the inherited
`migration.rb:1374-1376` regex handles `db/migrate/<version>_<name>.ts`, scope
group included. `migrationFiles()` stays overridden (that is the real virtual-FS
seam), and `MIGRATION_FILE_PATTERN` and its `replace(/-/g, "_")` are deleted.

Confirm no tutorial fixture or checked-in sandbox file uses a hyphen separator
before deleting; if one does, rename the fixture rather than keeping the arm.

## Acceptance criteria

- [ ] `VfsMigrationContext` no longer overrides `parseMigrationFilename`.
- [ ] `MIGRATION_FILE_PATTERN` is gone from `trails-cli.ts`.
- [ ] Scoped virtual migration filenames keep their scope, per `migration.rb:1375`.
- [ ] `packages/website/src/lib/frontiers/sandbox-sw.test.ts` and the tutorial
      replay tests keep their names and pass.
