---
title: "AssociationRelation#first_or_initialize override has no Rails counterpart"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing #7053 (`converge-association-relation-through-scope-onto-scoping`).

Rails' `AssociationRelation`
(`vendor/rails/activerecord/lib/active_record/association_relation.rb`) defines
`==`, the six bulk-insert guards, `_new`, `_create`, `_create!` and
`exec_queries` — and nothing else. It does NOT override
`first_or_initialize`; that method lives once, on `Relation`
(`vendor/rails/activerecord/lib/active_record/relation.rb:186-188`):

```ruby
def first_or_initialize(attributes = nil, &block) # :nodoc:
  first || new(attributes, &block)
end
```

trails ports that faithfully at
`packages/activerecord/src/relation.ts:1604-1609`
(`(await this.first()) || this.new(attributes, block)`), but
`AssociationRelation` also carries a second, divergent copy
(`packages/activerecord/src/association-relation.ts:127-133`):

```ts
async firstOrInitialize(extra?: Record<string, unknown>): Promise<T> {
  const records = await this.limit(1);
  if (records.length > 0) return records[0];
  return this.build(extra ?? {});
}
```

It differs from Rails in three ways: `limit(1)` + index instead of `first`, no
`block` parameter at all, and a renamed `extra` parameter where Rails has
`attributes`.

The override existed because `build` had to be routed through the association
by hand. #7053 removed that reason: `AssociationRelation#_new` now delegates to
the association and `Relation#build` wraps it in `scoping`, so the inherited
`first_or_initialize` already builds through the association with the FK and
polymorphic type set.

## Converged shape

`AssociationRelation#firstOrInitialize` is deleted. The inherited
`Relation#firstOrInitialize` is the only copy, matching Rails' single
definition, and it reaches the association through `_new`.

## Acceptance criteria

- [ ] `firstOrInitialize` is gone from `association-relation.ts`.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new novel name for
      `association-relation.ts` (the override is redundant surface, not a gain).
- [ ] Association `first_or_initialize` behaviour is unchanged: the built record
      still carries the FK / polymorphic type and joins the loaded target.
- [ ] Association and relation suites stay green, test names unchanged.
