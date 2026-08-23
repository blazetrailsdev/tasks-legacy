---
title: "DisableJoinsAssociationRelation overrides toArray on a clone where Rails overrides load in place"
status: claimed
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: "2026-08-23T12:57:31Z"
assignee: "converge-association-relation-inverse-wiring-onto-exec-queries"
blocked-by: null
closed-reason: null
---

## Context

Rails' `DisableJoinsAssociationRelation` does its id-order regrouping by
overriding **`load`**:

```ruby
# activerecord/lib/active_record/disable_joins_association_relation.rb:26-38
def load
  super
  records = @records
  records_by_id = records.group_by { |record| record[key] }
  records = ids.flat_map { |id| records_by_id[id] }
  records.compact!
  @records = records
end
```

Note what that does: it calls `super`, then rewrites **`@records` on
`self`**, so every later read — `records`, `to_ary`, `first`, `limit` —
sees the id-ordered array.

trails overrides **`toArray`** instead
(`packages/activerecord/src/disable-joins-association-relation.ts:552-626`)
and does the load on a **clone** (`Relation.prototype.toArray.call(loadClone)`),
returning the ordered array without ever assigning `this._records`. So on a
trails DJAR, `this.isLoaded` stays false and `records()` / `load()` do not
observe the reorder at all — only a direct `toArray()` call does.

Surfaced while converging the `Relation` loaded-arm readers onto the
`loaded?` / `records` seams (PR #6905), which could not invert
`to_ary` → `records` → `load` because it would route around this override.

## Converged shape

Override `load` with Rails' body: `await super.load()`, read `this._records`,
group by `key`, flat-map over `ids`, compact, and assign back to
`this._records`. Drop the `toArray` override and the clone; the limit/offset
handling that clone exists for belongs with Rails' own `limit` / `first`
overrides (disable_joins_association_relation.rb:13-24), which trails already
has.

## Acceptance criteria

- `DisableJoinsAssociationRelation` overrides `load`, not `toArray`.
- The id-order regroup is visible through `toArray()`, `records()` and
  `load()` alike, and `isLoaded` is true afterwards.
- `limit` / `first` keep their current loaded-chain semantics.
- Existing disable-joins tests stay green on all three lanes.
