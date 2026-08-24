---
title: "website-frontiers-sandbox-still-scaffolds-db-migrations"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
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

PR #6977 converged trailties' migration directory to Rails' `db/migrate`
(`Migrator.migrations_paths`, `vendor/rails/activerecord/lib/active_record/migration.rb:1419`)
for both discovery (`packages/trailties/src/commands/db.ts`
`migrationsDirsForConfig`) and scaffolding (`generators/migration-generator.ts`,
`generators/rails/migration/migration-generator.ts`, `generators/app-generator.ts`,
`commands/destroy.ts`).

The website's Frontiers sandbox is a second, independent implementation of the
same CLI over a VFS — it does not import the trailties generators, so it was
untouched and still writes and scans `db/migrations/`:

- `packages/website/src/lib/frontiers/trail-cli.ts:61` (`startsWith("db/migrations/")`)
  and `:148` (the "No migrations found in db/migrations/." message).
- `packages/website/src/lib/frontiers/tutorials/generator-fixtures.ts` — ~12
  expected-file globs, `db/migrations/*_create_*.ts`.
- `packages/website/src/routes/dev/filetree/+page.svelte:27-28`.
- Tests: `frontiers/runtime.test.ts:39,65,91`,
  `frontiers/components/sandbox/FileTree.test.ts:26`,
  `frontiers/tutorials/generator-fixtures.test.ts:54`.

So the tutorials now teach a directory the real CLI no longer uses. It was left
out of #6977 to keep that PR scoped to the trailties story it claimed.

## Acceptance criteria

- [ ] The Frontiers sandbox CLI generates into and discovers `db/migrate`.
- [ ] `generator-fixtures.ts` globs and the `dev/filetree` seed match.
- [ ] The "No migrations found in ..." message names `db/migrate`.
- [ ] Website tests keep their names and pass; no `db/migrations` left under
      `packages/website/src`.
