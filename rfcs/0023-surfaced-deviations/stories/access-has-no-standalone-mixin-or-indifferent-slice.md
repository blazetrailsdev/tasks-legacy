---
title: "ActiveModel::Access has no standalone mixin and slice lacks indifferent access"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `access_test.rb`'s assertions in PR #6519 (RFC 0105). The
assertion port is blocked on this deviation.

Rails `ActiveModel::Access` (`vendor/rails/activemodel/lib/active_model/access.rb`)
is a module a plain class includes, and `#slice` answers a
`HashWithIndifferentAccess`. Its test
(`vendor/rails/activemodel/test/cases/access_test.rb:6-24`) defines a bespoke
`Point` that does exactly that:

```ruby
class Point
  include ActiveModel::Access
  def initialize(*vector); @vector = vector; end
  def x; @vector[0]; end
  ...
end
```

and `test "slice"` (`access_test.rb:30-39`) leans on indifferent access:

```ruby
expected = { z: @point.z, x: @point.x }.with_indifferent_access
actual = @point.slice(:z, :x)
assert_equal expected.keys, actual.keys
expected.each do |key, value|
  assert_equal value, actual[key.to_s]
  assert_equal value, actual[key.to_sym]
end
```

trails has two gaps:

1. **No standalone mixin.** `slice` / `valuesAt` live directly on `Model`
   (`packages/activemodel/src/model.ts:2584-2605`); `packages/activemodel/src/access.ts`
   is an interface declaration only, with no implementation to include. A class
   that is not a `Model` cannot get them, so the Rails `Point` has no
   counterpart and `packages/activemodel/src/access.test.ts` uses a bespoke
   `SliceModel` instead.
2. **No indifferent access.** `slice` answers a plain object, so the Rails
   String-key / Symbol-key assertion pair collapses to one lookup, and
   `test "slice"`'s 3 `assert_equal`s cannot be mirrored without writing one
   vacuously.

Per the repo mixin convention this is the `this`-typed-functions shape (see
CLAUDE.md "Module mixins"): the implementations move to `access.ts` as
`this`-typed functions over an `Access` host interface, and `Model` assigns
them — which is also what makes `parity:api` find them at the Rails path.

## Converged shape

Move `slice` / `valuesAt` into `packages/activemodel/src/access.ts` as
`this`-typed functions, assigned onto `Model`; decide whether `slice` returns a
`HashWithIndifferentAccess` counterpart or whether the Symbol arm is a
documented language shortcoming at the call site. Then port `access_test.rb`'s
four tests against a `Point` that includes the mixin, replacing the bespoke
`SliceModel`.

## Acceptance criteria

- `access_test.rb` reports 0 assertion-count / -kind / -value mismatches in
  `pnpm parity:test -- --assertions --package activemodel`, and the mark is
  lowered by that amount.
- `slice` / `valuesAt` live at the Rails file path and `pnpm parity:api` for
  activemodel does not drop.
- No bespoke test model where the Rails test uses a plain class.
