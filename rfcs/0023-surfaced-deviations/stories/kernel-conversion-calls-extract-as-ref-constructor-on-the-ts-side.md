---
title: "call-args normalizes TS new Rational(...) to ref:constructor where Ruby's Kernel Rational() is ref:Rational"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

`scripts/api-compare/call-args.ts:182` normalizes a call named `new` to the
argument token `ref:constructor` (`:828` does the same for the Ruby side), which
is right for Ruby `Foo.new` vs TS `new Foo()` — both are constructions of a
named class and both lose the name the same way.

It is wrong for Ruby's **Kernel conversion functions**, which are ordinary
method calls carrying the class name as the callee. `Rational(999999999, 1000)`
extracts as `ref:Rational`; its only TS spelling is `new Rational(999999999,
1000)`, which extracts as `ref:constructor`, so the two never match:

```text
rubyArgs: kwargs{hour=num:23,min=num:59,sec=num:59,usec=ref:Rational}
tsArgs:   kwargs{hour=num:23,min=num:59,sec=num:59,usec=ref:constructor}
```

PR #6937 hit this converging `Time#change` onto Rails' single `new_sec`
Rational: `end_of_day` / `end_of_hour` / `end_of_minute`
(`activesupport/lib/active_support/core_ext/time/calculations.rb:261`, `:277`,
`:292`) pass `usec: Rational(999999999, 1000)`, and passing the _same_ Rational
in TS turned `parity:api:calls:args` red where the previous, less faithful
float `999999999 / 1000` was green. Three `@missingRailsArgs change — PERMANENT`
tags in `packages/activesupport/src/time-ext.ts` (endOfDay, endOfHour,
endOfMinute) exist only to hold that spelling difference, and the gate will
flag the same shape at every future `Rational()` / `Integer()` / `Float()` /
`String()` / `Array()` port.

## Converged shape

Where a TS argument is `new X(...)` and the paired Ruby argument is a call to a
same-named Kernel conversion function (`Rational`, `Complex`, `Integer`,
`Float`, `String`, `Array`, `Hash`), normalize the TS side to `ref:X` rather
than `ref:constructor` so the two tokens agree. `Foo.new` vs `new Foo()` keeps
today's `ref:constructor` behaviour — the change is scoped to the callee being a
capitalized bare-function call on the Ruby side, which is what distinguishes a
Kernel conversion from a constructor.

Cover it in `scripts/api-compare/call-args.test.ts` next to the existing
`normalizeArg("call:new")` cases (`:25-26`).

## Acceptance criteria

- [ ] A Ruby `Rational(a, b)` argument and a TS `new Rational(a, b)` argument
      compare equal; `Foo.new` vs `new Foo()` is unchanged.
- [ ] The three `@missingRailsArgs change` tags in
      `packages/activesupport/src/time-ext.ts` are deleted, and
      `pnpm parity:api:calls:args` stays clean without them.
- [ ] No baseline row is added; if the fix reveals converged rows, tighten with
      `parity:api:calls:tighten` rather than reseeding.
