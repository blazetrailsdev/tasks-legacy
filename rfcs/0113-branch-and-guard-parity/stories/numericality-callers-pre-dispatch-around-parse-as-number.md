---
title: "Numericality is_number?/option_as_number pre-coerce instead of letting parse_as_number dispatch"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`NumericalityValidator#is_number?` and `#option_as_number` both call
`parse_as_number` with the RAW value. trails pre-coerces at each call site and
passes the coerced result, so `parse_as_number`'s own type dispatch never sees
what Rails gives it.

Rails (`activemodel/lib/active_model/validations/numericality.rb:68-69, 94-99`):

```ruby
def option_as_number(record, option_value, precision, scale)
  parse_as_number(resolve_value(record, option_value), precision, scale)
end

def is_number?(raw_value, precision, scale)
  if options[:only_numeric] && !raw_value.is_a?(Numeric)
    return false
  end

  !parse_as_number(raw_value, precision, scale).nil?
rescue ArgumentError, TypeError
  false
end
```

`parse_as_number` (`numericality.rb:72-81`) is the single place the Float /
BigDecimal / Numeric / integer-string dispatch lives.

trails today (`packages/activemodel/src/validations/numericality.ts`) instead
passes a `coerced` value from `isNumber` and a `numeric` value from
`optionAsNumber`, having done a chunk of the dispatch inline at each site —
including a BigDecimal branch in `optionAsNumber` that deliberately bypasses
`parseAsNumber` entirely (documented in a comment there).

Surfaced by the RFC 0096 activemodel naming burndown (PR #6350) as two `naming`
rows; deliberately NOT renamed, because the locals are the tail of a duplicated
dispatch rather than misnamed variables.

The BigDecimal bypass is doing real work today (it avoids applying precision
where Rails' BigDecimal arm applies only scale). Converging means making
`parseAsNumber` itself faithful enough that the bypass is unnecessary — NOT
deleting the bypass and regressing the behaviour it protects.

## Converged shape

Make `parseAsNumber` carry Rails' full four-arm dispatch including the
scale-only BigDecimal arm (`round(raw_value, scale)`, `numericality.rb:75-76`),
then have both callers pass the raw value as Rails does and delete the inline
pre-dispatch at each site.

## Acceptance criteria

1. `isNumber` and `optionAsNumber` call `parseAsNumber` with the raw value, under
   Rails' parameter names (`rawValue`, and the `resolveValue(...)` result
   respectively).
2. `parseAsNumber` owns the whole dispatch; no arm is duplicated at a call site.
3. The BigDecimal precision-vs-scale behaviour the current bypass protects is
   pinned by a test BEFORE the bypass is removed.
4. The two `naming` rows for `validations/numericality.ts` in
   `pnpm parity:api:calls:args:report` are gone; report before/after.
5. `pnpm vitest run packages/activemodel` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
