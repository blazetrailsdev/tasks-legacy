---
title: "Make the adapter-specific boot path idempotent so add_index can drop ifNotExists"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`loadPostgresqlSpecificSchema`
(`packages/activerecord/src/support/load-schema-helper.ts`) passes
`ifNotExists: true` to the `company_include_index` `addIndex` call. Rails'
`vendor/rails/activerecord/test/schema/postgresql_specific_schema.rb:201` is a
bare `add_index(:companies, [:firm_id, :type], name: "company_include_index",
include: [:name, :account_id])` with no `if_not_exists:`.

The flag was added in PR #5550 because Rails loads its schema once per suite,
while trails re-runs the adapter-specific loader on every worker boot against a
database that may already carry the index. `force: true` makes the surrounding
`create_table` calls idempotent; `add_index` has no such flag, so without
`ifNotExists` the second boot dies on
`relation "company_include_index" already exists`.

The deviation is therefore a symptom of trails' per-worker re-boot model, not of
the index itself. The convergent fix is to make the boot path idempotent at the
boot-path level — e.g. skip the whole adapter-specific arm when the database
already carries it, or drop-and-relay it — so every `add_index` in these loaders
can be Rails-verbatim. Any future `add_index` added to these loaders will hit the
same wall and be tempted into the same deviation.

## Acceptance criteria

- The boot path is idempotent without per-call `ifNotExists` flags.
- `company_include_index`'s `addIndex` call is Rails-verbatim (no
  `ifNotExists`), and re-running `loadAdapterSpecificSchema` against an
  already-loaded database still succeeds.
