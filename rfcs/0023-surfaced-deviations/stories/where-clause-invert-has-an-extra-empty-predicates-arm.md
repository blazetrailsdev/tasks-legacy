---
title: "WhereClause#invert adds an empty-predicates branch Rails does not have"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while burning down the relation-family call-set rows (PR #6721,
`wave-4a-relation-family-residue`), reading `WhereClause#invert` against Rails.

Rails (`vendor/rails/activerecord/lib/active_record/relation/where_clause.rb:83-90`):

```ruby
def invert
  if predicates.size == 1
    inverted_predicates = [ invert_predicate(predicates.first) ]
  else
    inverted_predicates = [ Arel::Nodes::Not.new(ast) ]
  end

  WhereClause.new(inverted_predicates)
end
```

Two branches. trails (`packages/activerecord/src/relation/where-clause.ts`,
`invert`) has **three** — it adds an empty-predicates early return Rails does
not have:

```ts
invert(): WhereClause {
  if (this.predicates.length === 0) return this.clone();   // <-- no Rails counterpart
  if (this.predicates.length === 1) { ... }
  return new WhereClause([new Nodes.Not(this.ast)]);
}
```

With zero predicates Rails takes the `else` arm: `ast` is
`Arel::Nodes::And.new([])` (`where_clause.rb:69-72`, `predicates.one?` false →
`And.new(predicates)`), wrapped in `Not`. trails returns a COPY of the empty
clause instead, so `where.not` over an empty clause produces a different AST
than Rails.

Left out of scope in #6721: it is a control-flow deviation, not one of that
story's baseline rows, and flipping it changes emitted SQL — it wants its own
change with its own test coverage.

## Converged shape

Delete the `length === 0` early return so the body is Rails' two branches
verbatim. Check what `Nodes.And` with an empty child list renders as in the
trails Arel visitor first — if it does not match MRI's `Arel` here, that
rendering is the actual bug and the guard is masking it.

## Acceptance criteria

- [ ] `WhereClause#invert` has exactly Rails' two branches (`where_clause.rb:83-90`).
- [ ] A test covers inverting an empty where-clause and asserts the Rails AST /
      SQL (verify against MRI — `ruby` is on PATH).
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no new
      baseline row.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
