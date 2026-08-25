---
title: "arel-activemodel-widened-params-and-clone-caches"
status: closed
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "arel"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Converged. Re-verified against origin/main 2026-08-25: all three parts are done. select-manager.ts:399,406 declare `intersect(other: SelectManager)` / `except(other: SelectManager)` with no union type and no hoisted otherAst local (select_manager.rb:209-216); attribute-set.ts:197-199 is `new AttributeSet(transformValues(this.attributes(), (attr) => attr.deepDup()))` with no clone cache (attribute_set.rb:73-75, converged by the RFC 0115 dup split, #7027); attribute-set/builder.ts:260-266 unions eachKey in Rails' types -> values -> delegate order (builder.rb:129-132). Nothing left to converge."
---

## Context

Surfaced by RFC 0096 wave 2 (`naming-burndown-2-arel-activemodel`). Four
`naming` call-argument rows are not identifier renames — each is a widened TS
parameter type or an invented clone-cache that forces a local Rails does not
have (a3).

**1. `SelectManager#intersect` / `#except` — union-typed `other`.** 2 rows.

`vendor/rails/activerecord/lib/arel/select_manager.rb:209-216`:

```ruby
def intersect(other)
  Nodes::Intersect.new ast, other.ast
end

def except(other)
  Nodes::Except.new ast, other.ast
end
```

trails (`packages/arel/src/select-manager.ts:381-392`) takes
`other: SelectManager | SelectStatement` and hoists
`const otherAst = other instanceof SelectManager ? other.ast : other;`.
Rows: `RB [ref:ast, ref:ast]` vs `TS [ref:ast, ref:otherAst]`. `#union`
(select-manager.ts:373-376) has the identical shape and is only unreported
because Ruby dispatches through a `node_class` local there.

**2. `AttributeSet#deepDup` — invented clone cache.** 1 row.

`vendor/rails/activemodel/lib/active_model/attribute_set.rb:73-75`:

```ruby
def deep_dup
  AttributeSet.new(attributes.transform_values(&:deep_dup))
end
```

trails (`packages/activemodel/src/attribute-set.ts:182-191`) builds a
`newAttrs` Map plus a `cache: Map<Attribute, Attribute>` threaded into
`this.cloneAttribute`. Row: `RB [ref:transformValues]` vs `TS [ref:newAttrs]`.
`AttributeSet#map` (same file) was converged in the burndown PR because Rails
does name a `new_attributes` local there; `deepDup` has no local at all.
Note `LazyAttributeHash#deepDup` in
`packages/activemodel/src/attribute-set/builder.ts:199-207` carries the same
cache.

**3. `LazyAttributeHash#keys` — receiver order.** 1 row (an a1 finding).

`vendor/rails/activemodel/lib/active_model/attribute_set/builder.rb:129-132`:

```ruby
def each_key(&block)
  keys = types.keys | values.keys | delegate_hash.keys
  keys.each(&block)
end
```

trails (`packages/activemodel/src/attribute-set/builder.ts:209-215,268-275`)
unions in the order `delegate` → `values` → `types` in both `eachKey` and
`keys`. Row: `RB [ref:attributes]` vs `TS [ref:values]`. Ruby's `|` preserves
first-seen order, so iteration order differs from Rails wherever a key exists
in more than one table. Also note Ruby has `each_key` and `key?` but no public
`keys` on `LazyAttributeHash` — check whether the TS `keys()` is extra surface.

## Acceptance criteria

- [ ] `intersect` / `except` (and `union`) take a `SelectManager` and read
      `other.ast`, matching select_manager.rb:198-216; any `SelectStatement`
      callers are fixed at the call site rather than by widening the signature.
- [ ] `deepDup` mirrors attribute_set.rb:73-75 with no cache local, or the cache
      is justified at the call site with the concrete cycle it prevents.
- [ ] `eachKey` / `keys` union in Rails' `types | values | delegate_hash` order.
- [ ] All four `naming` rows disappear from
      `pnpm parity:api:calls:args:report`; no new `shape` rows.
- [ ] `pnpm lint`, `pnpm vitest run packages/arel/src` and
      `pnpm vitest run packages/activemodel/src` pass.
