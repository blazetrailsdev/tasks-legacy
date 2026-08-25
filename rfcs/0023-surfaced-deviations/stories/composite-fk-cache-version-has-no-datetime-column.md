---
title: "composite-fk-cache-version-has-no-datetime-column"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`cpk-eager-pluck-cache-version-composite-fk-collection.trails.test.ts` carried
two `cache_version` tests over `CpkBook`:

- `cache_version over eagerLoad('chapters') joins the composite-FK collection`
- `cache_version over eagerLoad('chapters') with a limit materializes the
composite primary keys`

Both passed `"revision"` as the timestamp column. `cpk_books.revision` is an
`integer` (`vendor/rails/activerecord/test/schema/schema.rb:243-250`), and
Rails' `compute_cache_version` calls `timestamp.utc` unconditionally
(`relation.rb:510-514`). Verified against MRI:

    ruby -e '5.utc'
    # NoMethodError: undefined method 'utc' for an instance of Integer

There is no `Integer#utc`/`Numeric#utc` extension anywhere in
`vendor/rails/activesupport`, and no Rails test calls
`cache_version`/`compute_cache_version` with anything but a datetime column.
So Rails cannot produce the `"#{size}-#{integer}"` those two tests asserted —
it raises first. They were removed in PR #6663 rather than kept alive by a
`String(timestamp)` fallback with no Rails counterpart.

Removing them lost no JoinDependency coverage: the same two behaviours are
already asserted through `pluck` in the same file
(`pluck over eagerLoad('chapters') joins the composite-FK collection` and
`pluck over eagerLoad('chapters') with a limit materializes the composite
primary keys`), which is why only the observable, not the subject, was dropped.

The reason the coverage cannot simply be moved to a datetime column: **no
`cpk_*` table in `schema.rb` has one.** `cpk_books`, `cpk_chapters`,
`cpk_authors`, `cpk_posts`, `cpk_comments`, `cpk_reviews`, `cpk_orders`,
`cpk_order_tags` and `cpk_tags` (schema.rb:243-300) are all integer/string
only. Adding one would be a bespoke-schema invention, which CLAUDE.md forbids.

## Acceptance criteria

- [ ] If Rails ever adds a datetime/timestamp column to a `cpk_*` table,
      restore composite-FK `cache_version` coverage against that column and
      mirror the canonical schema addition in `test-helpers/test-schema.ts`.
- [ ] Do NOT re-add an integer-timestamp arm to `Relation#computeCacheVersion`
      to make such a test possible — Rails raises there, and that is the
      behaviour to mirror.
- [ ] If the composite-FK `cache_version` path is judged worth covering before
      then, do it against a single-PK model that already has `updated_at`, and
      say plainly in the test name that it is not the composite case.
