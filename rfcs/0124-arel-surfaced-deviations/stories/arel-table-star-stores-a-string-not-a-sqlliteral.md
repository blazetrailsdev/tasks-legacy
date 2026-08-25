---
title: "Arel::Table#star stores a plain string, forcing a sentinel branch in visit_Arel_Attributes_Attribute"
status: done
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 7054
claim: "2026-08-25T16:56:36Z"
assignee: "arel-star-is-a-shared-const-not-a-per-call-method"
blocked-by: null
closed-reason: null
---

## Context

Surfaced converging `visit_Arel_Attributes_Attribute` in PR #6378.

Rails (`vendor/rails/activerecord/lib/arel/visitors/to_sql.rb:746-749`) has no
special case for the star projection:

```ruby
def visit_Arel_Attributes_Attribute(o, collector)
  join_name = o.relation.table_alias || o.relation.name
  collector << quote_table_name(join_name) << "." << quote_column_name(o.name)
end
```

It does not need one, because `Arel::Table#[Arel.star]` puts a
`SqlLiteral("*")` on the Attribute and `quote_column_name` passes any
`SqlLiteral` through untouched (`to_sql.rb:877-880`).

trails puts the plain JS string `"*"` on the Attribute instead
(`packages/arel/src/table.ts:233-235`):

```ts
get star(): Attribute {
  return new Attribute(this, "*");
}
```

so `quoteColumnName` (`packages/arel/src/visitors/to-sql.ts:1591-1594`) would
quote it as an identifier, and the visitor carries a sentinel short-circuit
Rails does not have (`to-sql.ts:1327-1336`):

```ts
collector.append(o.name === "*" ? "*" : this.quoteColumnName(o.name));
```

This is a modeling divergence, not a language shortcoming: trails already has
`Nodes.SqlLiteral` and `quoteColumnName` already passes it through.

## Acceptance criteria

- `Table#star` returns `new Attribute(this, sql("*", { retryable: true }))` —
  the `SqlLiteral`, as `Arel::Table#[Arel.star]` does. Coordinate with
  [[arel-star-is-a-shared-const-not-a-per-call-method]], which owns whether
  `Arel.star` is a const or a per-call method.
- `Attribute#name` accepts `string | SqlLiteral` (Rails' `o.name` is whatever
  was stored).
- `visitArelAttributesAttribute` drops the `o.name === "*"` branch and is the
  three bare appends Rails has, with no comment explaining a sentinel.
- Existing arel to_sql tests still pass on all three dialects — MySQL must
  still emit `` `table`.* `` and not `` `table`.`*` ``.
