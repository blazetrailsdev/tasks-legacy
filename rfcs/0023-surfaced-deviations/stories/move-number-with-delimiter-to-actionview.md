---
title: "number_with_delimiter is ActionView's helper living in ActiveSupport's number_helper.ts"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionview"
  - "activesupport"
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

`numberWithDelimiter` lives in `packages/activesupport/src/number-helper.ts`,
but it is not an ActiveSupport method. ActiveSupport's `number_helper.rb` has
`number_to_delimited` (`activesupport/lib/active_support/number_helper.rb:264-266`)
— PR #6556 ported it under that name. `number_with_delimiter` is **ActionView's**
helper (`actionview/lib/action_view/helpers/number_helper.rb:75-77`):

```ruby
def number_with_delimiter(number, options = {})
  delegate_number_helper_method(:number_to_delimited, number, options)
end
```

PR #6556 left it in place (now delegating to `numberToDelimited`, with the
ActionView cite in its JSDoc) rather than widen the blast radius, but it is a
member of the wrong package's file: `packages/actionview/src/helpers/index.ts`
re-exports it FROM activesupport, which inverts Rails' dependency direction —
ActionView delegates down to ActiveSupport, never the reverse.

`parity:api:extra` scores it "moved" rather than "novel" (the name exists in
Rails, just in another .rb), so it does not currently show as invented surface —
which is exactly why it has survived.

## Converged shape

Move `numberWithDelimiter` to ActionView's number-helper module, next to the
other `number_*` helpers ActionView owns, and have it delegate to
`@blazetrails/activesupport`'s `numberToDelimited` — the direction
`delegate_number_helper_method` goes. Drop it from
`packages/activesupport/src/number-helper.ts`, from the `_helpers` object, from
the `NumberHelper` namespace and from the activesupport barrel.

Check the other six `_helpers` members while you are there: if any of them is
likewise an ActionView name rather than an ActiveSupport one, it belongs in the
same move.

## Acceptance criteria

- [ ] `numberWithDelimiter` is declared in actionview and delegates to
      activesupport's `numberToDelimited`.
- [ ] activesupport's `number-helper.ts` exports only ActiveSupport's own
      `number_helper.rb` members.
- [ ] `packages/actionview/src/helpers/index.ts` no longer re-exports it from
      activesupport; actionview's number-helper tests still pass.
- [ ] `pnpm parity:api:extra` deltas non-negative for both packages.
