---
title: "Discriminate local_level='s Symbol arm from its String arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
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

# Discriminate `local_level=`'s Symbol arm from its String arm

## Context

`vendor/rails/activesupport/lib/active_support/logger_thread_safe_level.rb:14-22`:

```ruby
case level
when Integer
when Symbol
  level = Logger::Severity.const_get(level.to_s.upcase)
when nil
else
  raise ArgumentError, "Invalid log level: #{level.inspect}"
end
```

Four arms. A **Symbol** goes through `const_get` (NameError on a miss); a
**String** falls to `else` and raises ArgumentError — `:debug` and `"debug"`
behave differently in Ruby.

PR #6535 (`converge-local-level-symbol-arm-name-error`) split the NameError
arm out of the ArgumentError arm, so an unknown Symbol name now raises what
Ruby raises. What is still collapsed is the Symbol/String discrimination:
`packages/activesupport/src/logger.ts`'s setter treats EVERY JS string as the
Symbol arm, so Ruby's String arm is unreachable for any string value. The
setter's JSDoc states this and why it is currently harmless — `LogLevel` is the
closed set of level names, it is the same spelling `level=` takes, and no
caller passes a string here meaning anything but a level (`logAt`,
`broadcast-logger.ts:60-66`, the tests). That is a justification, not a
convergence.

## Converged shape

CLAUDE.md's rule for a method whose control flow turns on `Symbol === x` is to
keep the Symbol's leading colon in the string: `":debug"` for the Symbol arm,
`"debug"` for the String arm, `.slice(1)` for the name. Applying it here means
deciding it for the whole logger severity surface at once, not just this one
writer — `level=`, `logAt`, `silence` and `broadcast-logger.ts`'s delegating
setter all take the same `LogLevel` spelling, and a colon on one of them only
would be worse than the current collapse. So the story is: either

- spell severity Symbols with the colon across the logger surface and let the
  bare-string arm raise ArgumentError as Ruby does; or
- establish (and write down once, not per-file) that trails' severity Symbols
  are bare strings because the String arm is dead everywhere, and retire the
  per-call-site justification.

The first is the convergence; the second is only correct if the first is shown
to be unbuildable.

## Acceptance criteria

- [ ] `logger.localLevel = "nope"` and `logger.localLevel = ":nope"` raise the
      errors Ruby raises for `"nope"` and `:nope` respectively, OR the decision
      not to discriminate is recorded once at the type, with every logger entry
      point agreeing.
- [ ] `level=`, `logAt`, `silence` and `broadcast-logger.ts:60-66` all take the
      same spelling as this setter.
- [ ] Covers for both arms.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
