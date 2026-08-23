---
title: "attributes-for-create-reads-the-composite-pk-id-reader"
status: done
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6907
claim: "2026-08-23T11:42:26Z"
assignee: "attributes-for-create-reads-the-composite-pk-id-reader"
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/call-mismatches-exclude/activerecord/attribute-methods.json`
carries one row, `attributes_for_create | id`, whose reason says trails checks
the per-column attribute value where Rails reads the composite-PK `id` reader:

`attribute_methods.rb`'s `attributes_for_create` rejects a column when
`pk_attribute?(name) && id.nil?`. For a composite key, Rails' `id` returns an
Array, which is never `nil` even when one member is — so the Rails behaviour and
the trails behaviour differ on a composite PK with one nil member.

RFC 0106 wave 5g could not migrate it: the migrator refuses a seeded reason, and
the reason here names a real behavioural divergence rather than a language
shortcoming, so a `@missingRailsCall` receipt would ratify it.

## Acceptance criteria

- [ ] `attributesForCreate` reads the `id` reader exactly as
      `attribute_methods.rb` does, so a composite PK with a nil member behaves
      as Rails does.
- [ ] A regression test covering a composite-PK insert with one nil member that
      fails on the baseline (see project_cpk_pk_null_passes_sqlite_fails_pg_mysql).
- [ ] The baseline row is deleted, not reworded, and the shard removed if empty.
- [ ] `pnpm parity:api:calls` green; SQLite, PostgreSQL and MySQL lanes green.
