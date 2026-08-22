---
title: "XmlMini defines private to_i/to_f/to_d/to_s instead of shared core-ext ports; its to_s is not inspect-faithful"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6465's `ActiveSupport::XmlMini::PARSING` port
(`vendor/rails/activesupport/lib/active_support/xml_mini.rb:66-96`) needed four
Ruby core conversions that trails has no shared port of, so it implemented them
as module-private helpers inside
`packages/activesupport/src/xml-mini.ts`: `toI`, `toF`, `toD`, `toS`.

They stand in for real Ruby methods with real Rails call sites elsewhere:

- `toI` / `toF` — `String#to_i` / `String#to_f` (leading-numeric-prefix
  semantics: `"123,003".to_f == 123.0`, `"".to_i == 0`), used at
  `xml_mini.rb:72-73`.
- `toD` — `String#to_d` (`bigdecimal/util`), used at `xml_mini.rb:75`. Rails
  reaches it through `require "bigdecimal/util"` at `xml_mini.rb:5`.
- `toS` — `Object#to_s`, used at `xml_mini.rb:82`.

Two problems:

1. **They are invented surface duplicating Ruby core.** `parity:api:extra`
   cannot see them (they are not exported), but the next caller needing
   `String#to_d` will write a second copy. The natural homes already exist:
   `packages/activesupport/src/core-ext/big-decimal/conversions.ts` for `to_d`,
   and a `core-ext/string/conversions` for `to_i`/`to_f`.
2. **`toS` is not faithful for Hash.** Ruby's `Hash#to_s` IS `inspect`, which
   quotes string keys and values — `{"a" => "b"}.to_s == '{"a" => "b"}'` — while
   the local helper emits `{a => b}`. The Rails test only exercises `{}` and
   `[]` (`xml_mini_test.rb:328-336`), so the gap is untested and shipped green.
   `Array#to_s` has the same inspect-quoting rule.

## Converged shape

- `to_d` ported onto the existing BigDecimal core-ext with Ruby's lenient
  leading-prefix semantics (it never raises, unlike `BigDecimal(str)`).
- `to_i` / `to_f` ported to the String core-ext at their Rails-derived names.
- `to_s` for Array/Hash either routed through a real `inspect` port with Ruby's
  quoting, or narrowed and justified at the call site if a full `inspect` is out
  of scope — but not left silently wrong.
- `xml-mini.ts` imports all four rather than defining them.

Verify each against the `ruby` on PATH rather than deriving from docs; that is
how #6465 caught the `Date.new` arity and Temporal basic-format issues.

## Acceptance criteria

- `toI`, `toF`, `toD`, `toS` are gone from `xml-mini.ts`, replaced by imports of
  shared core-ext ports at their Rails-derived names.
- `Hash#to_s` / `Array#to_s` quote strings as Ruby's `inspect` does, with a
  cover asserting `{"a" => "b"}` and `["a"]` shapes against MRI output.
- `xml_mini_test.rb`'s `ParsingTest` (`integer`, `float`, `decimal`, `string`)
  still passes unchanged.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
