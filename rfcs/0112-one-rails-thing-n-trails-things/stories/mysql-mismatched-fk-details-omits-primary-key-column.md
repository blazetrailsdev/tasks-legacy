---
title: "mismatched_foreign_key_details omits primary_key_column; the lookup is deferred to a trails-only _enrichMismatchedForeignKey rebuild"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Baselined in PR #6577 (RFC 0106 wave 3b): row
`mismatched_foreign_key_details | column_for` in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`.

`abstract_mysql_adapter.rb:978-999` ends with:

    if match
      options[:table] = match[:table]
      options[:foreign_key] = match[:foreign_key]
      options[:target_table] = match[:target_table]
      options[:primary_key] = match[:primary_key]
      options[:primary_key_column] = column_for(match[:target_table], match[:primary_key])
    end
    options

trails (`abstract-mysql-adapter.ts:1440`) returns the four parsed names and
omits `primary_key_column`. The referenced column is instead looked up in
`_enrichMismatchedForeignKey` (`:1451`), a trails-only method with no Rails
counterpart, which rebuilds the whole `MismatchedForeignKey` afterwards.

**Why it was baselined:** `column_for` is an async schema read in trails, and
`mismatchedForeignKeyDetails` is reached synchronously from
`_translateException`. Rails' `column_for` is a plain blocking call.

Note Rails also uses this method as a lazy `query_parser` proc when `sql` is
nil (rb:1012), so whatever shape this converges to must still work when the SQL
arrives later.

## Converged shape

`mismatchedForeignKeyDetails` returns `primaryKeyColumn` alongside the four
names, and `_enrichMismatchedForeignKey` — the trails-only rebuild step — is
retired. This depends on the exception-translation path being able to await,
which is the same constraint several other rows on this file hit; sequence it
after (or with) whatever RFC 0076 settles for the async translate path.

If the sync constraint proves genuinely immovable, the fallback is to keep the
enrichment but justify it with a `@noRailsEquivalent` tag at the call site
rather than a baseline row — but that is the fallback, not the goal.

## Acceptance criteria

- [ ] `mismatchedForeignKeyDetails` mirrors rb:978-999 including
      `primary_key_column`.
- [ ] `_enrichMismatchedForeignKey` is gone, or carries a reviewed
      `@noRailsEquivalent` with maintainer sign-off.
- [ ] The lazy `query_parser` arm (rb:1009-1013) still resolves details when
      `sql` is nil at raise time.
- [ ] `mysql2-adapter.mismatched-fk-enrichment.trails.test.ts` still passes or
      is re-pointed at the converged shape.
- [ ] The `column_for` row deleted by hand from the shard (no reseed).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
