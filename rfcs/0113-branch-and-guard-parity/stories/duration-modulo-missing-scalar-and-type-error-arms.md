---
title: "Duration#% drops Rails' Scalar arm and raise_type_error tail"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
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

Rails' `Duration#%` has three arms
(`vendor/rails/activesupport/lib/active_support/duration.rb:310-320`):

```ruby
def %(other)
  if Duration === other || Scalar === other
    Duration.build(value % other.value)
  elsif Numeric === other
    Duration.build(value % other)
  else
    raise_type_error(other)
  end
end
```

`packages/activesupport/src/duration.ts`'s `modulo` (converged onto the value
seat and `Duration.build` by #6693) carries only the `Duration` and numeric
arms: its signature is `(other: Duration | number)`, so a `Scalar` receiver has
no arm and a non-numeric argument does not reach `raiseTypeError`. `Scalar` is
exported from the same file and `raiseTypeError` is already ported
(`duration.rb:520-522`), so both arms are a direct write.

The same `else raise_type_error(other)` tail is missing from `times` (`*`,
`:286-294`) and `dividedBy` (`/`, `:296-308`) — port those tails in the same
pass if they are still absent.

## Acceptance criteria

- [ ] `modulo` accepts `Duration | Scalar | number`, taking the
      `other.value` arm for both `Duration` and `Scalar`, the numeric arm for
      a number, and `raiseTypeError(other)` otherwise.
- [ ] `times` / `dividedBy` carry Rails' `raise_type_error` tail.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
