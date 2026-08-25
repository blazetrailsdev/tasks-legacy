---
title: "ForeignKeyChangeColumnTest cases drive rows with raw SQL instead of Rails' Rocket/Astronaut models"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Migration::ForeignKeyChangeColumnTest`
(vendor/rails/activerecord/test/cases/migration/foreign_key_test.rb:23-143)
declares its own `Rocket` (`has_many :astronauts`) and `Astronaut`
(`belongs_to :rocket`) models and drives every case through them:
`Rocket.create!(name: "myrocket")`, `rocket.astronauts << Astronaut.create!`,
and `assert_equal "myrocket", Rocket.first.name`. The `setup`/`teardown` pair
calls `Rocket.reset_table_name` / `reset_column_information` on both, which is
what makes the `WithPrefix` / `WithSuffix` subclasses (:145-163) re-resolve the
model table names under `ActiveRecord::Base.table_name_prefix` / `_suffix`.

The trails port in `packages/activerecord/src/migration/foreign-key.test.ts`
(the `foreignKeyChangeColumnTest` factory, landed in #5543) substitutes raw
`INSERT` / `SELECT` statements through `conn.executeMutation` /
`conn.execute` for all of it. So the port never exercises the association
write path (`<<`), the model-level table-name re-resolution, or
`resetColumnInformation` after the DDL under test — which is a large part of
what the Rails cases are checking (the FK survives a change/rename _and_ the
model still reads back through it).

The blocker to a straight conversion is
[[project_bespoke_registermodel_shadows_canonical_file_wide]]: declaring
`Rocket`/`Astronaut` in this test file registers them file-wide and would
shadow the canonical models used by the other ~70 cases in the same file.
Sizing assumes that has to be solved (scoped registration, or moving the
change-column describes to their own file) before the models can land.

## Acceptance criteria

- [ ] The `ForeignKeyChangeColumnTest` cases drive their rows through `Rocket`
      / `Astronaut` models declared as Rails declares them, not raw SQL.
- [ ] `resetTableName` / `resetColumnInformation` are called in the
      setup/teardown positions Rails calls them, so the prefix/suffix variants
      re-resolve the model table names rather than relying on literals.
- [ ] The canonical models used by the rest of `foreign-key.test.ts` are not
      shadowed.
- [ ] Green on all three adapters; `parity:test` delta non-negative and
      `--gates --check` stays at exit 0.
