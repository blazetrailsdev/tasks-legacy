---
title: "extract_options! returns a tuple instead of mutating the receiver"
status: draft
updated: 2026-08-17
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

`extractOptionsBang` (`packages/activesupport/src/hash-utils.ts:179`) has a
different signature from the Ruby it ports.

Ruby's `Array#extract_options!`
(`activesupport/lib/active_support/core_ext/array/extract_options.rb:22-28`)
MUTATES the receiver — it `pop`s the trailing hash off `self` — and returns
only the options:

```ruby
def extract_options!
  if last.is_a?(Hash) && last.extractable_options?
    pop
  else
    {}
  end
end
```

The port instead leaves the argument array untouched and returns a
`[args, options]` tuple, so every call site destructures a pair Rails has no
counterpart for:

```ts
export function extractOptionsBang<T>(args: T[]): [T[], AnyObject];
```

The existing JSDoc justifies this with "a TS caller has no `pop`-in-place idiom
for a rest parameter" — but that is not a language shortcoming. A rest
parameter IS a real, mutable array in JS, and `args.pop()` does exactly what
Ruby's `pop` does. The tuple shape is a preference, and per CLAUDE.md a
documented deviation is debt, not permission.

This was surfaced while converging `core_ext/array/extract_options_test.rb`
assertion parity in PR #6620: the Rails test asserts on the mutated receiver
(`assert_equal([], array)` after `array.extract_options!`,
`extract_options_test.rb:38`), which the port can only express by reading the
tuple's first element.

Note `isExtractableOptions` dispatch itself was converged in #6620 — the guard
now admits any Hash subclass the way Ruby's `last.is_a?(Hash)` does. Only the
signature is left.

## Converged shape

```ts
export function extractOptionsBang<T>(args: T[]): AnyObject {
  const last = args[args.length - 1];
  if (args.length > 0 && isHash(last) && isExtractableOptions(last)) {
    return args.pop() as unknown as AnyObject;
  }
  return {};
}
```

Two non-test call sites and the ported `extract_options_test.rb` update with
it. Related but distinct: [[extract-options-bang-baseline-rows]] converges the
two call-mismatch baseline rows for the same method.

## Acceptance criteria

- `extractOptionsBang` returns only the options and pops the extracted hash off
  the array it is given, matching extract_options.rb:22-28.
- Both non-test call sites and `core-ext/array/extract-options.test.ts` are
  updated; the ported test asserts on the mutated array as Rails does.
- `pnpm parity:test -- --assertions --package activesupport` does not regress
  for `core_ext/array/extract_options_test.rb` (it is at 0 mismatches today).
