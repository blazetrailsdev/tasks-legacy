---
title: "find_nth_from_last diverges from Rails on index base and the order/limit guard"
status: draft
updated: 2026-08-16
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

Surfaced while converging the `finder-methods.ts` call-set rows in PR #6584
(RFC 0106 wave 2). Two divergences in
`packages/activerecord/src/relation/finder-methods.ts` `findNthFromLast`, both
left alone there because they are behavioral rather than call-set rows.

Rails (`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:621-633`):

```ruby
def find_nth_from_last(index)
  if loaded?
    records[-index]
  else
    relation = ordered_relation

    if relation.order_values.empty? || relation.has_limit_or_offset?
      relation.records[-index]
    else
      relation.reverse_order.offset(index - 1).first
    end
  end
end
```

**1. Index base.** Ruby indexes from the end 1-based (`records[-index]`, and
`offset(index - 1)`); trails indexes `records[records.length - 1 - index]` and
`offset(index)`, i.e. 0-based. The callers were ported to match the trails
convention, so the pair is self-consistent and the tests pass — but every
identifier is off by one against the Ruby, which is exactly the kind of silent
skew a later port of a caller will get wrong.

**2. The order guard.** Rails' condition is
`relation.order_values.empty? || relation.has_limit_or_offset?`. trails spells
it `!hasOrder(relation) || relation._limitValue != null || relation._offsetValue != null`,
with a comment saying `hasOrder()` is used so `_rawOrderClauses` (e.g.
`inOrderOf`) also count as "has an order". `has_limit_or_offset?` exists on the
relation in Rails (`relation.rb`) and trails has a `hasLimitOrOffset` sited by
PR #6578, so the second half can converge directly; the `order_values.empty?`
half needs the `_rawOrderClauses` question answered rather than assumed.

## Acceptance criteria

- [ ] `findNthFromLast` uses Ruby's 1-based-from-the-end indexing, or the
      deviation is justified at the call site with the caller convention it
      pairs with spelled out.
- [ ] The limit/offset half of the guard calls `hasLimitOrOffset()` rather than
      re-deriving it from `_limitValue`/`_offsetValue`.
- [ ] The `order_values.empty?` half is either converged or the
      `_rawOrderClauses` divergence is cited against the Rails behaviour it
      changes, with a test.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
