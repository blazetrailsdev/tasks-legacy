---
title: 'Give type/time.test.ts the verbatim "user input in time zone" Rails test name'
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
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

`parity:test` reports activemodel at 962/963, and the single remaining gap is a
**Miss** on `type/time_test.rb`'s `test_user_input_in_time_zone`
(`vendor/rails/activemodel/test/cases/type/time_test.rb`), not a skip:

````text
type/time_test.rb   type/time.test.ts   2  0  0  0  1  23  3
    - user input in time zone
```text

The behaviour is ported — `port-user-input-in-time-zone-and-close-the-activemodel-test-gap`
landed it — but it was split into four differently-named cases in
`packages/activemodel/src/type/time.test.ts:88-110`:

- `it("user input in time zone wraps plain time in Time.zone")`
- `it("user input in time zone answers a zoneless value when Time.zone is unset")`
- `it("user input in time zone returns null for null")`
- `it("user input in time zone passthrough for ZonedDateTime")`

`parity:test` matches on the exact normalized name, so none of the four credits
the Rails test: it counts 1 Miss plus 4 Extra, and the package cannot reach
963/963 while the name is absent.

Surfaced while closing `converge-numericality-bigint-exponent-skip` (#6989),
which took `numericality_validation_test.rb` to 41/41 and left this as the only
activemodel gap.

## Converged shape

One `it("user input in time zone")` carrying the assertions of Rails'
`test_user_input_in_time_zone`, verbatim and in Rails' order. Any TS-only arm
that Rails' body does not assert stays as a separate case in the `.trails`
sibling, per CLAUDE.md's "TS-only extras go in the trails test file" — not
folded into the Rails-named test and not left occupying a near-miss name.

## Acceptance criteria

- [ ] `packages/activemodel/src/type/time.test.ts` carries
      `it("user input in time zone")` matching Rails' `test_user_input_in_time_zone`.
- [ ] `parity:test` shows `type/time_test.rb` at Miss=0, and activemodel at
      **963/963**.
- [ ] No behaviour change; the arms not asserted by Rails keep their coverage.
````
