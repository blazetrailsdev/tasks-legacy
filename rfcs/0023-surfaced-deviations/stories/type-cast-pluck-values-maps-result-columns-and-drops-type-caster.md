---
title: "type_cast_pluck_values maps result.columns and never consults column.type_caster"
status: draft
updated: 2026-08-18
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

Found while writing the per-site reason for the
`type_cast_pluck_values -> fetch` / `-> size` call-set rows (PR #6721,
`wave-4a-relation-family-residue`).

Rails (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:610-624`):

```ruby
def type_cast_pluck_values(result, columns)
  cast_types = if result.columns.size != columns.size
    model.attribute_types
  else
    join_dependencies = nil
    columns.map.with_index do |column, i|
      column.try(:type_caster) ||
        model.attribute_types.fetch(name = result.columns[i]) do
          join_dependencies ||= build_join_dependencies
          lookup_cast_type_from_join_dependencies(name, join_dependencies) ||
            result.column_types[i] || Type.default_value
        end
    end
  end
  result.cast_values(cast_types)
end
```

trails (`packages/activerecord/src/relation/calculations.ts`,
`typeCastPluckValues`) diverges twice in the else arm:

1. It maps over **`result.columns`**, not `columns`. Rails maps the CALLER's
   column list and uses `result.columns[i]` only to name the attribute lookup.
2. It drops Rails' first alternative, **`column.try(:type_caster)`** — an Arel
   node passed to `pluck` that carries its own type caster should win over the
   model's attribute type, and here it never gets consulted.

Because the else arm is only reached when the two lists are the same length,
(1) is currently invisible; (2) is a live behavioral gap for
`pluck(Arel.sql(...))`-style calls whose node answers `type_caster`.

Out of scope for #6721, which converged call-SET rows only.

## Converged shape

Port the else arm line by line: `columns.map((column, i) => ...)`, with
`column.try(:type_caster)` as the first alternative, then the
`attribute_types.fetch(result.columns[i]) { ... }` miss block that is already
correct.

## Acceptance criteria

- [ ] The else arm maps `columns` (the parameter), not `result.columns`.
- [ ] `column.try(:type_caster)` is consulted first, per calculations.rb:616.
- [ ] A test plucks an Arel node carrying a `typeCaster` and asserts that caster
      wins over the model attribute type; canonical schema/models only.
- [ ] `pnpm parity:api:calls` / `:args` green; the two existing rows either
      retire or keep an accurate reason.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
