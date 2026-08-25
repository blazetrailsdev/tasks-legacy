---
title: "merger-merge-clauses-drops-where-merge"
status: draft
updated: 2026-08-12
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

Surfaced while burning down RFC 0096 wave-2 naming rows.

`Merger#merge_clauses`
(`vendor/rails/activerecord/lib/active_record/relation/merger.rb:176-184`) is:

```ruby
def merge_clauses
  relation.from_clause = other.from_clause if replace_from_clause?

  where_clause = relation.where_clause.merge(other.where_clause)
  relation.where_clause = where_clause unless where_clause.empty?

  having_clause = relation.having_clause.merge(other.having_clause)
  relation.having_clause = having_clause unless having_clause.empty?
end
```

trails (`packages/activerecord/src/relation/merger.ts:271-278`) does the
having-clause merge **first**, the from-clause replacement **second**, and
**omits the where-clause merge entirely** — the where values are merged
elsewhere in the port. This is a control-flow divergence (branch order plus a
dropped statement), not a naming one, so RFC 0096 leaves its
`merge → ref:_havingClause` vs `ref:whereClause` row standing.

## Acceptance criteria

- [ ] `mergeClauses` mirrors `merge_clauses` statement for statement: the
      `replaceFromClause?` guard first, then the where-clause merge, then the
      having-clause merge, each with Rails' `unless …empty?` guard.
- [ ] Whatever path currently merges where values is either the one
      `mergeClauses` calls or is removed, so the merge happens exactly once.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green and
      the `merger.ts` naming row is gone.
- [ ] Relation merge tests pass; a regression test covers the ordering if the
      current order is observable.
