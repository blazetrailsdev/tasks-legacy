---
title: "retire-version-env-legacy-fallback-in-target-version"
status: closed
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
closed-reason: 'Obsolete: PR #6980 converged targetVersion() to read ENV["VERSION"] only (maintainer decision, Rails database_tasks.rb:317-325 over BC-2''s TRAILS_ prefix for this variable). There is no TRAILS_MIGRATION_VERSION read left to retire the fallback from.'
---

## Context

`DatabaseTasks.targetVersion` reads
`getEnv("TRAILS_MIGRATION_VERSION") ?? getEnv("VERSION")`
(`packages/activerecord/src/tasks/database-tasks.ts`). The `VERSION` arm is
labelled a "legacy fallback (one-release window)" — the compatibility tail of
BC-2's env rename (`VERSION` → `TRAILS_MIGRATION_VERSION`,
`docs/infrastructure/browser-compat-plan.md`, shipped in #1251). The window has
never been closed and the comment has outlived it.

Rails reads only `ENV["VERSION"]` (`database_tasks.rb:317-325`), so trails is
carrying two keys where Rails and the post-BC-2 trails convention each carry
one. PR #6980 converged `checkTargetVersion` to zero-arity over
`targetVersion()` so the pair is now single-source over that one expression;
this story retires the second key from the expression itself.

Producers to retire alongside it:

- `packages/activerecord-cli/src/db-tasks.ts:113-122` (sets
  `TRAILS_MIGRATION_VERSION` — already the new name, no change expected)
- `packages/trailties/src/commands/db.ts` `withTargetVersionEnv` (same)
- `packages/activerecord/src/tasks/database-tasks.test.ts` —
  `DatabaseTaskTargetVersionTest` / `DatabaseTaskCheckTargetVersionTest` set
  `process.env.VERSION`; they would move to the canonical key. These are Rails
  test names (`database_tasks_test.rb`) — the NAMES must not change, only the
  env key the bodies set.

## Acceptance criteria

- [ ] `targetVersion()` reads `TRAILS_MIGRATION_VERSION` only; the `?? getEnv("VERSION")`
      arm and its "legacy fallback" note are gone.
- [ ] The `Invalid format of target version: \`VERSION=...\`` message still
      matches Rails verbatim (the message text is Rails', independent of the key
      trails reads).
- [ ] No test name changes; only the env key the bodies set.
- [ ] `docs/infrastructure/browser-compat-plan.md` needs no edit — it already
      specifies the rename as a direct replacement, not an aliasing.
