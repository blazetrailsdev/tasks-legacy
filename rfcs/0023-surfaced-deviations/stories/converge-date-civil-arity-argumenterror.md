---
title: "Converge Date.civil to raise ArgumentError on an over-long argument list"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "date"
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

Surfaced by PR #6652 (assertion parity for `xml_mini_test.rb`). Five of that
file's `ParsingTest` tests each close with the same Rails assertion:

```ruby
# vendor/rails/activesupport/test/xml_mini_test.rb:258, :292, :303, :314, :335
assert_raises(ArgumentError) { parser.call(Date.new(2013, 11, 12, 02, 11)) }
```

MRI raises from `Date.new`'s own arity — `wrong number of arguments (given 5,
expected 0..4)` — before `parser.call` ever runs (verified with the `ruby` on
PATH). trails spells `Date.new` as `Date.civil`
(`packages/date/src/date.ts`, `civil`), which takes four parameters and
silently ignores a fifth argument at runtime: calling
`(RubyDate.civil as any)(2013, 11, 12, 2, 11)` returns `2013-11-12` rather
than raising. Verified in this worktree.

So the assertion has no faithful counterpart today and PR #6652 left it
dropped, with the reasoning recorded at
`packages/activesupport/src/xml-mini.test.ts:22-31`. That is the last thing
holding `xml_mini_test.rb` at 5 assertion-count / 5 assertion-kind mismatches
in `pnpm parity:test -- --assertions --package activesupport`; everything else
in the file is at 0/0/0.

## Converged shape

`Date.civil` (and the other `Date` constructors that take a bounded parameter
list) raise `ArgumentError` with MRI's message when handed more arguments than
they accept:

```text
ArgumentError: wrong number of arguments (given 5, expected 0..4)
```

JS functions cannot express Ruby arity checking declaratively, but the check is
writable — `arguments.length` / a rest parameter guarded at the top of the body
is the shape other trails ports use for Ruby arity raises (see the
`converge-notifications-subscribe-callback-arity` and
`converge-delete-restriction-error-arity` stories for prior art).

Once `Date.civil` raises, restore the five dropped assertions in
`packages/activesupport/src/xml-mini.test.ts` (`symbol`, `integer`, `float`,
`decimal`, `string`) as
`expect(() => (RubyDate.civil as any)(2013, 11, 12, 2, 11)).toThrow(ArgumentError)`
and delete the deviation note at xml-mini.test.ts:22-31.

## Acceptance criteria

- `Date.civil` raises `ArgumentError` with MRI's message on an over-long
  argument list.
- The five `ParsingTest` tests in
  `packages/activesupport/src/xml-mini.test.ts` carry the Rails
  `assert_raises(ArgumentError)` arm again.
- `xml_mini_test.rb` reports 0 assertion-count and 0 assertion-kind mismatches
  in `pnpm parity:test -- --assertions --package activesupport`, and the
  activesupport marks in `scripts/test-compare/assertion-mismatch-mark.json`
  are lowered accordingly (currently 943 / 1293 / 114).
- The deviation note at xml-mini.test.ts:22-31 is deleted, not reworded.
