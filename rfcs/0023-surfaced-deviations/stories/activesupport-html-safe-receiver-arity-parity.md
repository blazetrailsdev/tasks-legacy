---
title: "isHtmlSafe takes the receiver as a parameter where Rails' html_safe? is zero-arg"
status: draft
updated: 2026-08-08
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6238: with each `Object` reopening credited to the TS file
mirroring it, `html_safe?` is compared against its TS port for the first time
and reports two arity mismatches (the method is expected from two Ruby
reopenings), `ruby()` vs `ts(value)`.

Rails
(`vendor/rails/activesupport/lib/active_support/core_ext/string/output_safety.rb:7-9`
and `:13-15`) defines it as a zero-arg predicate on the receiver:

```ruby
class Object
  def html_safe?
    false
  end
end

class Numeric
  def html_safe?
    true
  end
end
```

trails ports it as a free function taking the receiver as its only parameter
(`packages/activesupport/src/core-ext/string/output-safety.ts:166`):

```ts
export function isHtmlSafe(value: unknown): boolean;
```

This is the recurring core_ext receiver-as-first-argument shape — JS cannot
reopen `Object.prototype` the way Ruby reopens `Object`. It is a real divergence
in the arity register, not a false positive: the port is one function where Rails
has per-class arms, so the `Numeric` arm (`true`) and the `Object` arm (`false`)
are folded into one body's branching rather than two definitions.

## Converged shape

Decide the one shape for core_ext receiver predicates and apply it here, rather
than leaving `isHtmlSafe` as an unregistered arity row. Options, in fidelity
order:

1. Keep the per-class arms Rails has as separate declarations on the classes
   trails already models for `blank?`/`present?` — `core-ext/object/blank.ts`
   declares statics on an `Object` overload set, which is the settled trails
   idiom for exactly this Ruby shape. `html_safe?`'s `Object` and `Numeric` arms
   would follow it.
2. If (1) is not reachable, register the receiver parameter in
   `arity-exclude.json` with the language-shortcoming reason — but only after
   (1) has actually been tried, and note that this whole family (`blank?`,
   `present?`, `duplicable?`, …) would need the same treatment for consistency,
   so a per-method row is the wrong granularity.

Do not close this by broadening an existing baseline reason.

## Acceptance criteria

- [ ] `core-ext/string/output-safety.ts:isHtmlSafe` no longer reports an arity
      mismatch, and the fix is the same shape the sibling core_ext predicates use.
- [ ] Rails' per-class arms (`Object` -> false, `Numeric` -> true) are both
      preserved.
- [ ] `pnpm parity:api --arity` activesupport count drops by 2; delta
      non-negative.
