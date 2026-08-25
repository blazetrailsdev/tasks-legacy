---
title: "NumericalityValidator#validate_each walks one slice(*RESERVED_OPTIONS) loop"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6377, which ported `COMPARE_CHECKS` as Rails' Hash and moved
NumericalityValidator's six hand-rolled compare blocks onto the Rails
dispatch. The REST of `validate_each` is still not Rails' shape.

`activemodel/lib/active_model/validations/numericality.rb:48-64`:

```ruby
options.slice(*RESERVED_OPTIONS).each do |option, option_value|
  if NUMBER_CHECKS.include?(option)
    unless value.to_i.public_send(NUMBER_CHECKS[option])
      record.errors.add(attr_name, option, **filtered_options(value))
    end
  elsif RANGE_CHECKS.include?(option)
    unless value.public_send(RANGE_CHECKS[option], option_value)
      record.errors.add(attr_name, option, **filtered_options(value).merge!(count: option_value))
    end
  elsif COMPARE_CHECKS.include?(option)
    option_value = option_as_number(record, option_value, precision, scale)
    unless value.public_send(COMPARE_CHECKS[option], option_value)
      record.errors.add(attr_name, option, **filtered_options(value).merge!(count: option_value))
    end
  end
end
```

`packages/activemodel/src/validations/numericality.ts` instead runs the
compare loop, then two standalone `if` blocks for `in` and for `odd`/`even`.
Two consequences: the branches are not the one `slice(*RESERVED_OPTIONS)` loop
Rails walks, and errors come out in a FIXED order rather than the caller's
option order, which is what Rails' `Hash#slice` preserves.
`RANGE_CHECKS`/`NUMBER_CHECKS` are declared in the file (`:13-14` ported) but
only `RESERVED_OPTIONS` reads them.

`options[:in]` is also modelled as a `[min, max]` tuple rather than a Range, so
`value.public_send(:in?, option_value)` (`:55`) has no receiver to dispatch on.

## Acceptance criteria

1. `validateEach` walks ONE loop over the reserved options in the caller's
   option order, with Rails' `NUMBER_CHECKS` / `RANGE_CHECKS` / `COMPARE_CHECKS`
   branch order and guards (`numericality.rb:48-64`).
2. `NUMBER_CHECKS` and `RANGE_CHECKS` are the dispatch tables their Rails
   counterparts are, read at the branch rather than only through
   `RESERVED_OPTIONS`.
3. The `:in` option's covers-check reaches the same shape (or the deviation is
   cited at the call site if Range has no trails port yet).
4. Existing Rails numericality tests stay green; `pnpm parity:api:calls` and
   `pnpm parity:api:calls:args` green.
