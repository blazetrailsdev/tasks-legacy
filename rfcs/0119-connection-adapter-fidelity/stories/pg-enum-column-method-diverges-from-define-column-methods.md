---
title: "PG TableDefinition#enum is unimplemented and diverges from Rails' generated shape"
status: draft
updated: 2026-07-29
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Surfaced while porting the variadic `*names` shape in PR #5575, which
deliberately left `enum` out of scope.

`enum` IS one of the 31 types in PostgreSQL's `define_column_methods` call
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:185-189`),
so in Rails it has exactly the same generated body as its 30 siblings:

```ruby
def enum(*names, **options)
  raise ArgumentError, "Missing column name(s) for enum" if names.empty?
  names.each { |name| column(name, :enum, **options) }
end
```

trails diverges three ways:

1. `enum` is **declared but never implemented** on
   `PostgreSQL::TableDefinition`. It appears only in that file's
   `ColumnMethods` interface
   (`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:97`)
   with no matching class member. The live implementation is the abstract
   `TableDefinition#enum`
   (`abstract/schema-definitions.ts:1396-1398`), which is not where Rails puts it.
2. It takes a **required** `enum_type` option and forwards that as the column
   type, rather than passing `:enum` as the type with `enum_type` riding along
   in options.
3. The option key is snake_case `enum_type`, against the repo's camelCase
   convention, and the abstract implementation destructures that literal key.

That signature is why PR #5575 could not fold `enum` into the variadic
conversion: a trailing-options split would read the required option hash as a
column name.

The PG-specific `enumType(name, enumName, options)` helper is a separate
trails-only surface in the same file and should be reviewed alongside this.

Note `abstract/schema-dumper.ts:393` emits `t.enum(...)` lines, so the dumper
output is the compatibility constraint on any signature change.

## Acceptance criteria

- [ ] `enum` is implemented on `PostgreSQL::TableDefinition` with Rails'
      generated `*names, **options` shape, raising
      `Missing column name(s) for enum` on an empty name list.
- [ ] `enum_type` is passed through options as a camelCase key rather than being
      consumed as the column type; the abstract override is removed or
      justified at the call site.
- [ ] The PG `ColumnMethods` interface entry matches the implementation — no
      declared-but-unimplemented member.
- [ ] `abstract/schema-dumper.ts`'s `t.enum(...)` emission still round-trips;
      PG schema-dumper suite green.
- [ ] `enumType` reviewed: either justified as a needed extra or removed.
- [ ] Green on all three lanes; parity:api delta non-negative.
