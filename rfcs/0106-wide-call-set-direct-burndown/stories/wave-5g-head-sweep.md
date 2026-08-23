---
title: "wave-5g-head-sweep"
status: ready
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
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

`wave-5f-head-sweep` (PR #6897) cleared 13 of the ~47 `kind: "set"` rows in its
band, emptying and deleting two shards (`result.json`,
`connection-adapters/schema-cache.json`). It stopped at the 700 LOC ceiling —
a migrated row costs ~7 JSON deletions plus a JSDoc line — so the rest of the
band is this story.

Remaining shards and row counts (measured on #6897's branch):

    activerecord/database-configurations/connection-url-resolver.json   5
    activerecord/relation/merger.json                                   5
    activerecord/relation/batches/batch-enumerator.json                 5
    activesupport/callbacks.json                                        5
    activesupport/encrypted-file.json                                   7
    activerecord/connection-adapters/postgresql/database-statements.json 4
    activesupport/cache.json                                            4
    activerecord/attribute-methods.json                                 3  (all seed placeholder)
    activerecord/relation/batches.json                                  1  (`any?`)
    activesupport/duration.json                                         3  (`iso8601` x2, `parse`)
    activesupport/values/time-zone.json                                 4  (`utc` x4, added on main after #6897 was cut)

Four of those are **not** straight migrations — #6897 left them baselined on
purpose because their reviewed reason names a real divergence, and a
`@missingRailsCall` receipt there would ratify it:

- `batches.rb:346` `batch_on_unloaded_relation | any?` — Rails'
  `values.flatten.any?(nil)` raises `ArgumentError` when a custom select clause
  omits a cursor column. The guard is unported. **Converge it.**
- `duration.rb:474` `iso8601 | new` / `iso8601 | serialize`, and
  `duration.rb:144-147` `parse` — Rails extracts `ISO8601Serializer` and
  `ISO8601Parser`; trails inlines both. **Extract them, with the Rails names.**
- `postgresql/database_statements.rb` `execute_batch | raw_execute` — Rails
  calls `raw_execute`; trails routes through `internalExecute` so the
  `materializeTransactions` / `allowRetry` kwargs travel with it. Decide
  whether that is convergeable and either fix it or give the tag a
  `CONVERGEABLE (story …)` claim naming a follow-up.
- `activesupport/cache.json` (4 rows) — `pnpm parity:api:build` reports
  `no body-bearing declaration` for `constructor`, `keyMatcher`,
  `mergedOptions`, `normalizeVersion` in `cache.ts`, so the migrator cannot
  place their tags. Find out why the declarations don't match and fix the
  matching, or write the tags by hand.
- `activerecord/attribute-methods.json` (3 rows) — still the RFC 0047/0084 seed
  placeholder. The migrator refuses those: converge, or write a real per-site
  reason first.

## Acceptance criteria

- [ ] Every row above is converged, or leaves as a `@missingRailsCall` tag
      carrying its reviewed per-site reason at the call site, opened with a
      `PERMANENT` / `CONVERGEABLE (story …)` permanence claim.
- [ ] Emptied shards deleted, not committed as `[]`;
      `pnpm parity:api:calls:tighten <shard>` for any stale mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] Anything that cannot converge is filed as its own story with the Rails
      `file:line`, not ratified in place.
