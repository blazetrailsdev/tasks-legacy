---
title: "Store accessors must read through the store's HashWithIndifferentAccess"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Store accessors must read through the store's HashWithIndifferentAccess

## Context

Surfaced in PR #6568, which made HWIA convert nested Hashes on write
(`convert_value`, `hash_with_indifferent_access.rb:392-403`).

`vendor/rails/activerecord/test/cases/store_test.rb:214-221`:

    user.update(settings: { "color" => { "jenny" => "blue" }, homepage: "rails" })

    assert_equal true, user.settings.instance_of?(ActiveSupport::HashWithIndifferentAccess)
    assert_equal "blue", user.settings[:color][:jenny]
    assert_equal "blue", user.color[:jenny]

In Rails the last two lines read the SAME object: `store_accessor` reads the
key out of the store hash (`store.rb`'s `read_store_attribute` →
`store_accessor_for(store_attribute).read(self, ...)`), so a nested value comes
back as the HWIA the store holds. In trails, `user.settings.get("color")` is a
`HashWithIndifferentAccess` while `user.color` is the raw deserialized plain
object, so `user.color.get("jenny")` raises `TypeError: not a function`. The
trails test at `packages/activerecord/src/store.test.ts` ("serialize stored
nested attributes") is currently pinned to the plain-object shape with
`toMatchObject`, which is what hides it.

## Converged shape

The store accessor reads through the same HWIA the store attribute holds, so
`user.color` and `user.settings.get("color")` are the same object, and the test
asserts Rails' `user.color[:jenny]` directly.

## Acceptance criteria

- [ ] `store.test.ts`'s "serialize stored nested attributes" asserts
      `(user.color as HashWithIndifferentAccess).get("jenny")` — Rails'
      `user.color[:jenny]` — with no `toMatchObject` fallback.
- [ ] Cited Rails read path (`store.rb` `read_store_attribute`) is mirrored,
      not special-cased for nested values.
