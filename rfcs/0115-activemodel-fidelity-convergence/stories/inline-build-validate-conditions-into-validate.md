---
title: "Inline _buildValidateConditions into validate — Rails has no such helper"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
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

## Context

Rails' `validate` has no condition-building helper. It merges `on:` and
`except_on:` into the `:if` / `:unless` arrays inline in its own body and hands
the result straight to `set_callback`
(`activemodel/lib/active_model/validations.rb:160-185`):

```ruby
if options.key?(:on)
  options = options.merge(if: [predicate_for_validation_context(options[:on]), *options[:if]])
end

if options.key?(:except_on)
  options = options.dup
  options[:except_on] = Array(options[:except_on])
  options[:unless] = [
    ->(o) { options[:except_on].intersect?(Array(o.validation_context)) },
    *options[:unless]
  ]
end

set_callback(:validate, *args, options, &block)
```

trails extracts that merge into `Model._buildValidateConditions`
(`packages/activemodel/src/model.ts`), a private static with no Rails
counterpart, called from three sites — `validate`, `validatesEach` and
`validatesWith`. PR #6793 converged what the helper _does_ (it no longer
resolves filters; `Callback#conditionsLambdas` does, via `CallTemplate`) but
left the decomposition in place, so this is the residue: one Rails method body
is still two TS methods.

CLAUDE.md is explicit that decomposition is part of the mirror — "If Rails
inlines something, inline it. One Rails method is one TS method."

Note the three call sites are not all `validate`: Rails' `validates_each`
(`validations.rb:190-192`) forwards to `validates_with BlockValidator`, and
`validates_with` (`validations/with.rb`) does its own option handling. Check
each against its own Rails body rather than assuming they all want `validate`'s
merge — the shared helper may itself be hiding a divergence at two of the three
sites.

## Converged shape

The `on:` / `except_on:` merge is inlined into `validate`'s body at the Rails
line positions, using Rails' locals (`options`), and each other call site does
what ITS Rails counterpart does. `_buildValidateConditions` disappears.

## Acceptance criteria

- `_buildValidateConditions` no longer exists and has no callers.
- `validate`'s body performs the `on:` / `except_on:` merge inline, matching
  `validations.rb:170-182` branch for branch.
- `validatesEach` / `validatesWith` are checked against `validations.rb:190-192`
  and `validations/with.rb` respectively; any divergence the shared helper was
  masking is either fixed or filed.
- `pnpm parity:api:extra --package activemodel` non-negative; existing
  validation tests unchanged and green.
