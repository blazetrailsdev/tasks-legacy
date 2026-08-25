---
title: "__update_callbacks takes its target list as a parameter instead of computing descendants.prepend(self)"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `__update_callbacks` takes its target list as a parameter instead of computing `descendants.prepend(self)`

## Context

Surfaced converging the activesupport `callbacks.json` rows in PR #6735 (RFC
0106 wave 4f); left baselined as `__update_callbacks -> descendants` and
`-> prepend`.

Rails (`vendor/rails/activesupport/lib/active_support/callbacks.rb:686-691`):

```ruby
def __update_callbacks(name) # :nodoc:
  self.descendants.prepend(self).reverse_each do |target|
    chain = target.get_callbacks name
    yield target, chain.dup
  end
end
```

The receiver computes its own target list: every descendant, with `self`
prepended, walked in reverse. trails' `__updateCallbacks`
(`packages/activesupport/src/callbacks.ts:1255-1274`) takes that list as a
`targets` parameter and only does the reverse-walk and the chain dup, so both
Rails calls are absent and each caller re-derives the list — the exact
duplication `__update_callbacks` exists to prevent.

`setCallback` / `skipCallback` in the same file are the callers.

## Converged shape

Give the callbacks host a `descendants` reader (ActiveSupport's
`DescendantsTracker` is already ported), and have `__updateCallbacks(name, fn)`
compute `[this, ...this.descendants()].reverse()` itself, matching
`callbacks.rb:687`. The `targets` parameter goes away and the callers pass only
the name and the block, as Rails does.

## Acceptance criteria

- [ ] `__updateCallbacks` takes `(name, fn)` and computes its own target list
      via `descendants` + prepend.
- [ ] Every caller drops its hand-built target list.
- [ ] The two `activesupport/callbacks.json` `kind: "set"` rows are deleted,
      then `pnpm parity:api:calls:tighten activesupport/callbacks.json`.
- [ ] `CallbacksTest` and the ActiveRecord callback suites stay green — the
      reverse-walk order is load-bearing for inherited chains.
