---
title: "over_test equality bodies assert nothing; port Rails' Over value equality"
status: claimed
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-25T15:39:01Z"
assignee: "arel-case-reader-readonly-vs-attr-accessor"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while relocating `Attribute#over` to `WindowPredications` in #5027
(story `arel-attribute-over-not-in-window-predications`).

`packages/arel/src/nodes/over.test.ts`'s `describe("equality")` carries two
tests whose names match Rails but whose bodies are invented — they assert
nothing about `Over` equality:

- `"is equal with equal ivars"` compares two `Attribute` names
  (`(a.left as Nodes.Attribute).name`), never constructing an `Over`.
- `"is not equal with different ivars"` builds two `And` nodes and asserts
  `expect(a).not.toBe(b)` — a JS reference check that passes for any two
  distinct objects, so it cannot fail.

Rails (`vendor/rails/activerecord/test/cases/arel/nodes/over_test.rb:52-68`)
tests value equality via array-uniq on `Over` nodes:

```ruby
array = [Arel::Nodes::Over.new("foo", "bar"), Arel::Nodes::Over.new("foo", "bar")]
assert_equal 1, array.uniq.size
```

`Over < Binary`, and `binary.rb:19-27` defines `hash`, `eql?` and
`alias :== :eql?` — that is what makes `uniq` collapse the pair. The port
needs an equivalent value-equality surface before these tests can say
anything; `toBe`/`Set` dedup on object identity will not reproduce it.

Left out of #5027 deliberately: it depends on node equality/hash semantics,
which is a separate concern from `over`'s placement, and the fix is not
local to `over.test.ts` (it is a `Binary`-level capability question).

## Acceptance criteria

- [ ] Determine whether trails `Binary` (or `Node`) exposes a value-equality
      surface equivalent to Rails' `hash`/`eql?`/`==`; if not, decide and
      record whether to port one or to express the Rails assertion another
      way.
- [ ] `over.test.ts`'s two `describe("equality")` bodies assert `Over` value
      equality per `over_test.rb:52-68`, not attribute names or reference
      inequality. Test names unchanged.
- [ ] Each rewritten test fails on a baseline where the equality surface is
      absent or wrong (a `not.toBe` style assertion that cannot fail is the
      defect being removed).
- [ ] Audit sibling arel `describe("equality")` blocks for the same
      cannot-fail `expect(a).not.toBe(b)` pattern and note the count.
- [ ] parity:api / parity:test delta non-negative.
