---
title: "Duration::Scalar's arithmetic drops Rails' Duration arms and the calculate dispatch"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `Scalar#<=>` and `Scalar#==` in PR #6802.
`packages/activesupport/src/duration.ts`'s `Scalar` arithmetic does not mirror
`vendor/rails/activesupport/lib/active_support/duration.rb:41-105`:

```ruby
def +(other)
  if Duration === other
    seconds   = value + other._parts.fetch(:seconds, 0)
    new_parts = other._parts.merge(seconds: seconds)
    new_value = value + other.value
    Duration.new(new_value, new_parts, other.variable?)
  else
    calculate(:+, other)
  end
end
```

with `-`, `*`, `/`, `%` each carrying their own Duration arm, and the private
`calculate(op, other)` (duration.rb:96-104) dispatching Scalar / Numeric /
`raise_type_error`.

trails has, instead:

- `plus` / `minus` (`duration.ts`): a single `other instanceof Scalar ? other.value : other`
  unwrap returning a `Scalar` — the Duration arm is missing entirely, so
  `Scalar.new(1) + 1.day` is a `Scalar`, not the `Duration` Ruby returns.
- `times` / `div`: typed `(other: number)`, so both the Duration arm and the
  Scalar arm are gone.
- No `modulo` at all, though `Scalar#%` exists (duration.rb:82-88).
- `raiseTypeError` is ported (duration.rb:108-110) but nothing calls it: no
  `calculate`, so a non-numeric operand silently produces `NaN` where Ruby
  raises `TypeError`.

## Converged shape

One TS method per Ruby method, same branch order: each operator keeps its
`Duration === other` arm building a `Duration` from the merged `_parts`, and the
else arm goes through a private `calculate` that dispatches Scalar / Numeric /
`raiseTypeError`. Add `modulo` for `%`. Names and parameter names come from the
Ruby (`other`, `seconds`, `newParts`, `newValue`).

## Acceptance criteria

- [ ] `plus`, `minus`, `times`, `div`, `modulo` each mirror duration.rb:41-88,
      Duration arm first.
- [ ] A private `calculate` mirrors duration.rb:96-104 and is the only caller of
      `raiseTypeError`.
- [ ] A non-numeric, non-Duration operand raises `TypeError` with Rails' message
      instead of yielding `NaN`.
- [ ] Regression tests fail on the pre-change baseline; Rails-named tests where
      `duration_test.rb` has a counterpart, `duration.trails.test.ts` otherwise.
