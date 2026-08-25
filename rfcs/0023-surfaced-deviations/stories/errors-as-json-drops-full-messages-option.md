---
title: "errors-as-json-drops-full-messages-option"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `Errors#as_json` drops the `:full_messages` option Rails forwards to `to_hash`

## Context

Spotted while converging the neighbouring `messages -> to_hash()` argument row
(RFC 0099, PR for `converge-ar-and-model-non-constructor-argument-rows`).

`vendor/rails/activemodel/lib/active_model/errors.rb`:

```ruby
def as_json(options = nil)
  to_hash(options && options[:full_messages])
end
```

`packages/activemodel/src/errors.ts:178-183` ignores its `_options` parameter
entirely and hardcodes `this.toHash(false)`, so
`errors.asJson({ fullMessages: true })` silently returns short messages where
Rails returns full ones. The parameter is even spelled `_options` to mark it
unused.

Rails test: `vendor/rails/activemodel/test/cases/errors_test.rb`
(`test_as_json_with_full_messages`).

## Acceptance criteria

- [ ] `asJson(options)` forwards `options?.fullMessages` to `toHash`, matching
      Rails' `options && options[:full_messages]` (note Ruby's `&&` returns the
      LHS when options is nil, so a nil/absent options object yields a falsy
      argument, not `false` substituted for a stored value).
- [ ] The Rails `as_json` full-messages test is ported under its Rails name and
      fails on the current baseline.
- [ ] `pnpm parity:test` delta non-negative.
