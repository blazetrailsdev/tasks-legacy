---
title: "converge-relation-equals-to-rails-to-sql-comparison"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced converging `with_values` in #6607. `Relation#equals`
(`packages/activerecord/src/relation.ts:4602`) is:

```ts
async equals(other: Relation<T>): Promise<boolean> {
  const a = await this.toArray();
  const b = await other;
  ...
}
```

Rails' `Relation#==` (activerecord/lib/active_record/relation.rb:1253-1262) is:

```ruby
def ==(other)
  case other
  when Associations::CollectionProxy, AssociationRelation
    self == other.records
  when Relation
    other.to_sql == to_sql
  when Array
    records == other
  end
end
```

Two divergences:

- **The `Relation` arm never runs.** Rails compares `to_sql` — no query. trails
  materializes both sides with `toArray()` for every receiver, so comparing two
  unloaded relations issues two SELECTs where Rails issues none, and two
  relations with identical SQL but a changed table compare unequal.
- **It is `async`.** That makes it unusable from any synchronous Ruby-equality
  path. #6607 had to add an `isAsyncFunction` guard in `query-methods.ts`'s
  `deepEqual` because an unawaited `Promise` is truthy, which silently reported
  every pair of relations equal — that guard is a workaround for this shape, and
  converging the `Relation` arm to the sync `to_sql` comparison retires it.

The `CollectionProxy`/`AssociationRelation` and `Array` arms are missing
entirely; the port answers the `to_a` comparison for every input.

## Acceptance criteria

- [ ] `equals` mirrors relation.rb:1253-1262 arm for arm, including the
      `CollectionProxy`/`AssociationRelation` and `Array` cases.
- [ ] The `Relation` arm compares `toSql()`, issuing no query.
- [ ] The `isAsyncFunction` guard on the `equals` hook in `deepEqual`
      (`relation/query-methods.ts`) is removed once `equals` is synchronous for
      the Relation arm, and `with.trails.test.ts`'s "keeps both entries" case
      still passes.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
