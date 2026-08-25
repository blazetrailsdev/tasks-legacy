---
title: "consolidate the ported Kernel#Integer / Kernel#Float conversions into one home"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
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

Shipped in PR #6445 (`port-cache-retrieve-pool-options`).

`Cache::Store.retrieve_pool_options` coerces its pool options through Kernel:

```ruby
# activesupport/lib/active_support/cache.rb:213-214
pool_options[:size] = Integer(pool_options[:size]) if pool_options.key?(:size)
pool_options[:timeout] = Float(pool_options[:timeout]) if pool_options.key?(:timeout)
```

trails has no ported `Kernel#Integer` / `Kernel#Float`, so
`packages/activesupport/src/cache/store.ts` carries file-local `Integer()` and
`Float()` functions (plus `inspect()` and `rubyClassName()` for the error
messages). They are differential-tested against MRI 3.3.11 over 34 inputs and
correct, but they are private to one file, and the same conversion keeps getting
re-derived elsewhere: `packages/activemodel/src/attribute-assignment.ts:257`
declares its own Ruby `TypeError` for `Kernel.Float` in numericality coercion,
and `duration.ts` / `range-ext.ts` hand-roll adjacent numeric guards.

## Converged shape

One ported `Kernel#Integer` / `Kernel#Float` pair (with the Ruby literal
grammar: whitespace trim, underscore separators, `0x`/`0b`/`0o`/`0` radix
prefixes, leading-dot and hexadecimal float mantissas, `ArgumentError` on a bad
String and `TypeError` on a non-String/non-Numeric) in a shared activesupport
home, with `cache/store.ts` and `activemodel/attribute-assignment.ts` reading
it rather than each declaring their own. Keep the MRI differential coverage.

## Acceptance criteria

- [ ] `Integer()` / `Float()` live in one place at their Ruby names.
- [ ] `cache/store.ts` drops its file-local copies and calls the shared pair.
- [ ] The MRI-verified grammar cases stay covered by tests.
- [ ] `pnpm parity:api:extra --package activesupport` shows no new novel surface.
