---
title: "Type Attribute#name for the nil Table#[] accepts, dropping the Table#get cast"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

`Arel::Table#[]` takes any name, nil included:

```ruby
def [](name, table = self)
  name = name.to_s if name.is_a?(Symbol)
  Attribute.new(table, name)
end
```

(`vendor/rails/arel/lib/arel/table.rb:82`)

`ActiveRecord::Relation#delete_all` / `#update_all` depend on that: a model with
no primary key reaches `compile_delete` / `compile_update` as `table[nil]`,
since the key expression branches only on `composite_primary_key?`:

```ruby
key = if model.composite_primary_key?
  primary_key.map { |pk| table[pk] }
else
  table[primary_key]
end
```

(`vendor/rails/activerecord/lib/active_record/relation.rb:1027-1031`, and
`:610-614` for `update_all`)

PR #6602 converged both methods onto that single path, so trails now calls
`table.get(null)` too. `Table#get` accepts `string | null`, but
`Attribute.name` is typed `string`
(`packages/arel/src/attributes/attribute.ts`), so the null is smuggled through
a cast:

```ts
return new Attribute(table ?? this, resolved as string);
```

The cast is the deviation: the runtime value can be null, and the type says it
cannot. Widening `Attribute.name` to `string | SqlLiteral | null` was out of
scope for #6602 — `name` is read in many visitor and predicate-builder paths,
each of which needs its own null handling decided — so the narrow cast shipped
with a call-site comment instead.

Note the null name only ever renders when the statement takes the subquery
shape (`WHERE (pk) IN (SELECT ...)`), which is ill-defined for a pkless model
in Rails too — MRI emits an empty identifier there. Converging the type is
about removing the lie, not about changing that behaviour.

Surfaced while landing
`converge-update-delete-all-pkless-fallback-to-single-arel-path` (PR #6602).

## Acceptance criteria

- `Attribute.name` is typed to admit the null `Table#[]` can produce, and
  `Table#get` no longer casts (`packages/arel/src/table.ts`).
- Every reader of `Attribute.name` handles the null arm explicitly rather than
  inheriting it via `any` — walk the visitors (`to-sql.ts`
  `visitArelAttributesAttribute`, the composite-PK ordering path) and the
  activerecord predicate-builder call sites.
- Emitted SQL is unchanged for every non-null name; the pkless subquery arm
  keeps whatever MRI produces (verify with `ruby` against
  `vendor/rails/arel`, which is on PATH).
- `pnpm parity:api:calls` / `:args` clean; `parity:api` / `parity:test` deltas
  non-negative; green on SQLite, PostgreSQL and MySQL/MariaDB.
