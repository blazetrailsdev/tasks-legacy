---
title: "arel-substitute-binds-and-string-join-invented-conversions"
status: closed
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Converged. Re-verified 2026-08-25: collectors/substitute-binds.ts:15-23 reassigns the `bind` local in place after the valueForDatabase unwrap, matching substitute_binds.rb:20-23 \u2014 the module-level extractValue helper and its BindParam recursion are gone. factory-methods.ts:74-75 is `createStringJoin(to) { return this.createJoin(to, null, StringJoin) }`, passing `to` through as factory_methods.rb:23-25 does; the SqlLiteral wrap is gone. Both named naming rows are retired."
---

## Context

Surfaced by RFC 0096 wave 2 (`naming-burndown-2-arel-activemodel`, PR pending).
Two `naming` call-argument rows in `packages/arel/src/collectors/substitute-binds.ts`
and `packages/arel/src/factory-methods.ts` are not identifier renames — they are
invented conversion layers Rails does not have (an a3 finding), so the burndown
story left them standing.

**1. `SubstituteBinds#addBind` — invented `extractValue` helper.**

`vendor/rails/activerecord/lib/arel/collectors/substitute_binds.rb:20-23`:

```ruby
def add_bind(bind, &)
  bind = bind.value_for_database if bind.respond_to?(:value_for_database)
  self << quoter.quote(bind)
end
```

Rails reassigns the `bind` local in place. trails
(`packages/arel/src/collectors/substitute-binds.ts:3-18,33-35`) extracts a
module-level `extractValue(bind)` that additionally recurses through
`BindParam.value` — extra behavior with no Rails counterpart. The call-arg row
reads `RB [ref:bind]` vs `TS [ref:extractValue]`.

**2. `FactoryMethods#createStringJoin` — invented `SqlLiteral` wrap.**

`vendor/rails/activerecord/lib/arel/factory_methods.rb:23-25`:

```ruby
def create_string_join(to)
  create_join to, nil, Nodes::StringJoin
end
```

trails (`packages/arel/src/factory-methods.ts:74-77`) inserts
`const node = typeof to === "string" ? new SqlLiteral(to) : to;` and passes
`node`. Row: `RB [ref:to, nil, const:StringJoin]` vs
`TS [ref:node, nil, const:StringJoin]`.

Note: the two matching rows in `packages/arel/src/visitors/to-sql.ts`
(`quoteTableName`/`quoteColumnName` wrapping `name` in `toS(...)`, vs Rails
passing `name` straight through at `to_sql.rb:872-880`) are NOT part of this
story — they are already owned by
`0023-surfaced-deviations/arel-visitor-to-s-belongs-in-adapter-quoting`, which
has the right fix: move the `to_s` down into each adapter's quoting module,
where Rails has it. Dropping the wrap in the visitor alone reds
`packages/arel/src/visitors/to-sql.trails.test.ts:80` (confirmed on PR #6421),
because `toS` also reproduces Ruby's inspect-style `Array#to_s`.

## Acceptance criteria

- [ ] `addBind` inlines the `value_for_database` unwrap onto a reassigned `bind`
      local, matching substitute_binds.rb:20-23. If the `BindParam` recursion is
      genuinely required by a trails caller, that caller is the thing to fix.
- [ ] `createStringJoin` passes `to` through to `createJoin` as Rails does, with
      the string→`SqlLiteral` promotion (if still needed) moved into
      `createJoin`/`StringJoin` where Rails puts it.
- [ ] Both `naming` rows disappear from
      `pnpm parity:api:calls:args:report`; no new `shape` rows.
- [ ] `pnpm lint` and `pnpm vitest run packages/arel/src` pass.
