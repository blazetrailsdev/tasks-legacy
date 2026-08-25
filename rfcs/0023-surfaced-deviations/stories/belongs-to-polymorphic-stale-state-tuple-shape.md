---
title: "belongs_to polymorphic staleState: converge JSON.stringify shim to Rails [fk, type] tuple"
status: ready
updated: 2026-07-27
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

Surfaced while converging composite-FK `staleState` (PR #5090). Rails'
`BelongsToPolymorphicAssociation#stale_state`
(`vendor/rails/activerecord/lib/active_record/associations/belongs_to_polymorphic_association.rb:43-47`):

```ruby
def stale_state
  if foreign_key = super
    [foreign_key, owner[reflection.foreign_type]]
  end
end
```

returns a Ruby **Array** `[fk, type]` (or nil), compared by value via `!=` in
`Association#stale_target?` (association.rb:97-99).

trails (`packages/activerecord/src/associations/belongs-to-polymorphic-association.ts:60-66`)
returns `JSON.stringify([fkState, this.readForeignType()])` — an invented
string shape shimming JS's identity-only `!==` on arrays (a raw array return
would compare stale on every read). A BigInt FK component would also throw in
`JSON.stringify` here, the exact bug fixed for the base class in #4620/#5090.

## Acceptance criteria

- [ ] Converge the polymorphic `staleState` shape to Rails' `[fk, type]`
      tuple (or justify the stringify shim at the call site per the
      deviations rule), making `isStaleTarget()` compare by value.
- [ ] No throw when the polymorphic FK is a BigInt.
- [ ] `undefined` vs Rails nil return normalized to `null` for consistency
      with the base class.
