---
title: "check-target-version-takes-a-version-argument-rails-takes-none"
status: claimed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-24T12:51:22Z"
assignee: "converge-schema-cache-install-onto-cache-replacement"
blocked-by: null
closed-reason: null
---

## Context

Rails' `DatabaseTasks.check_target_version` takes **no arguments** and reads
`ENV["VERSION"]` itself:

```ruby
def check_target_version
  if target_version && !Migration.valid_version_format?(ENV["VERSION"])
    raise "Invalid format of target version: `VERSION=#{ENV['VERSION']}`"
  end
end
```

(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:317-321`,
with `target_version` at `:323-325` reading the same env var.)

trails' `DatabaseTasks.checkTargetVersion` takes an optional `version` parameter
(`packages/activerecord/src/tasks/database-tasks.ts:508`). PR #6977 inlined
`db:migrate:up` / `db:migrate:down` into `packages/trailties/src/commands/db.ts`
per `railties/databases.rake:172-177` and `:203-208`, and those call sites now
pass `opts.version` explicitly — surfacing the arity deviation at two more
places. `DatabaseTasks.migrate` (`database-tasks.ts:339`) calls it with no
argument, so both arms are live.

The rake tasks read `ENV["VERSION"]` for exactly this reason: `check_target_version`
and `target_version` are one pair over one source, and a caller that supplies
its own version bypasses `target_version`'s `blank?` normalization.

## Converged shape

`checkTargetVersion()` drops the parameter and reads the version through
`targetVersion()` / the `VERSION` env var, as Rails does. The trailties
`migrate:up` / `migrate:down` commands set `VERSION` (which is what the rake
task's caller does) rather than threading a value through the predicate.

## Acceptance criteria

- [ ] `DatabaseTasks.checkTargetVersion` is zero-arity and reads `ENV["VERSION"]`,
      matching `database_tasks.rb:317-321`.
- [ ] No caller passes a version to it.
- [ ] The invalid-format error message still matches Rails verbatim.
- [ ] activerecord-cli and trailties `migrate:up` / `migrate:down` tests keep
      their names and pass.
