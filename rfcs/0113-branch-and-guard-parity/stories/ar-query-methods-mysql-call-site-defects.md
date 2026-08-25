---
title: "ar-query-methods-mysql-call-site-defects"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the `naming` call-argument rows for RFC 0096 in
`naming-burndown-activerecord-rest-3`. Three rows in that story's file list are
NOT naming defects — renaming the local would paper over a real divergence — so
they were left alone per that story's acceptance criterion 4 ("argument-ORDER
defects and invented call-site conversions are filed, not renamed away").

### 1. `build_with_value_from_hash` builds the CTE with swapped arguments

`relation/query_methods.rb:1921-1927`:

```ruby
def build_with_value_from_hash(hash)
  hash.map do |name, value|
    expression = build_with_expression_from_value(value)
    Arel::Nodes::TableAlias.new(expression, name)
  end
end
```

Rails passes `(expression, name)`. `packages/activerecord/src/relation/query-methods.ts`
(`buildWithValueFromHash`, ~:2393) constructs with `(name, expr)` — the
comparator reports Rails `(ref:buildWithExpressionFromValue, ref:name)` vs
trails `(ref:name, ref:expr)`. Verify against `vendor/rails` which node class
trails constructs and whether the trails node's own parameter order is the
inverted one (in which case the node, not the call site, is what diverges).

### 2. `case_sensitive_comparison` invents a quoting conversion

`connection_adapters/abstract_mysql_adapter.rb:600-608`:

```ruby
attribute.eq(Arel::Nodes::Bin.new(value))
```

`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1042`
passes `new Nodes.Bin(attribute.quotedNode(value))` — an extra
`attribute.quotedNode(...)` conversion Rails does not make. Establish whether
trails' `Nodes::Bin` visitor already quotes (in which case the call site drops
the conversion) or whether the quoting belongs inside `Bin`.

### 3. `flattened_args` drops Rails' Hash arm

`relation/query_methods.rb:2077-2079`:

```ruby
def flattened_args(args)
  args.flat_map { |e| (e.is_a?(Hash) || e.is_a?(Array)) ? flattened_args(e.to_a) : e }
end
```

`packages/activerecord/src/relation/query-methods.ts:1580-1584` recurses only
on `Array.isArray(e)`, never on a Hash, and never applies `.to_a`. A Hash
argument therefore passes through un-flattened where Rails flattens it into
`[key, value]` pairs.

## Acceptance criteria

1. Each of the three sites either matches the Rails body cited above, or
   carries a reviewed one-line justification at the call site naming the Rails
   `file:line` and the blocker.
2. For (3), a regression test that fails on baseline: a `flattened_args`-reached
   caller passing a hash, asserting Rails' flattened result.
3. `pnpm parity:api:calls:args` green; any baseline row that goes stale is
   deleted by hand (only-shrink, no `--write`).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
