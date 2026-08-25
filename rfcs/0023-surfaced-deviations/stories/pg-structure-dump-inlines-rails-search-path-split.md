---
title: "structureDump routes search_path through an invented normalizeSchemaSearchPath helper"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQLDatabaseTasks#structure_dump` turns the search path into `--schema=`
flags. Rails does it inline, with no helper
(`vendor/rails/activerecord/lib/active_record/tasks/postgresql_database_tasks.rb:62-64`):

```ruby
unless search_path.blank?
  args += search_path.split(",").map do |part|
    "--schema=#{part.strip}"
  end
end
```

trails routes this through an exported, trails-invented helper,
`normalizeSchemaSearchPath` (`packages/activerecord/src/tasks/postgresql-database-tasks.ts:270-300`),
which does more than Rails: it strips surrounding single/double quotes,
unescapes doubled quotes inside quoted identifiers, and drops `$user` entries.

Two problems:

1. **It is invented surface.** It is the one `novel` entry `pnpm parity:api:extra
--package activerecord` reports for this file, and it carries no
   `@noRailsEquivalent` tag — only a prose doc comment.
2. **It changes behaviour, not just spelling.** Rails passes `'$user'` through to
   pg_dump as `--schema='$user'`; trails silently drops it. Rails passes a quoted
   identifier through verbatim; trails unquotes it. Whatever pg_dump does with
   those, Rails' behaviour is the reference and trails does not reproduce it.

The helper is exported solely so `postgresql-database-tasks.trails.test.ts:137-149`
can test it directly, which is a test convenience, not a Rails shape.

## Converged shape

Inline the split/strip into `structure_dump` exactly as Rails has it, delete the
exported helper, and retire the two trails-only tests that exercise it (their
subject stops existing). If the quote/`$user` handling turns out to be load-bearing
for a real trails lane, that is a separate finding to raise on its own evidence —
do not preserve it here by default.

Removing the export also removes the file's only `parity:api:extra` novel row.

## Acceptance criteria

- [ ] `normalizeSchemaSearchPath` is gone; `structure_dump` splits inline per
      `postgresql_database_tasks.rb:62-64`.
- [ ] `pnpm parity:api:extra --package activerecord` shows 0 novel for
      `tasks/postgresql-database-tasks.ts`.
- [ ] `adapters/postgresql/postgresql-rake.test.ts` "structure dump with schema
      search path" / "…dump schemas string" stay green (they pin the `foo,bar`
      argv through the public path).
- [ ] Green on the PG lane.
