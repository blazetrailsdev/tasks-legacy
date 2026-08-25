---
title: "Converge ThroughAssociation#stale_state onto Rails' array shape"
status: draft
updated: 2026-08-20
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

Surfaced in PR #6766, which moved `target_scope` / `stale_state` /
`foreign_key_present?` into the `ThroughAssociation` mixin.

Rails' `ThroughAssociation#stale_state`
(`vendor/rails/activerecord/lib/active_record/associations/through_association.rb:82-88`):

```ruby
def stale_state
  if through_reflection.belongs_to?
    Array(through_reflection.foreign_key).filter_map do |foreign_key_column|
      owner[foreign_key_column]
    end.presence
  end
end
```

returns a Ruby **Array** (or nil), compared by value with `!=` in
`Association#stale_target?` (association.rb:97-99).

trails
(`packages/activerecord/src/associations/through-association.ts`,
`ThroughAssociation.staleState`) returns the single element when there is one
and `JSON.stringify(state)` when the key is composite — an invented string
shape shimming JS's identity-only `!==` in
`Association#isStaleTarget` (`associations/association.ts`). A raw array return
would compare stale on every read, and a BigInt component would throw in
`JSON.stringify` — the exact bug fixed for the belongs_to base class in PRs
4620 and 5090.

Same deviation class as
[[belongs-to-polymorphic-stale-state-tuple-shape]]; converging both wants one
value-comparing `isStaleTarget`.

## Acceptance criteria

- [ ] `ThroughAssociation#staleState` returns Rails' array shape (or nil),
      not a scalar/`JSON.stringify` fold.
- [ ] `Association#isStaleTarget` compares stale states by value, so a
      composite through key does not read as stale on every access.
- [ ] No throw when a through FK component is a BigInt (PG/MariaDB hydrate
      PKs as BigInt).
- [ ] has_many/has_one `:through` and CPK through suites stay green on all
      three lanes.
