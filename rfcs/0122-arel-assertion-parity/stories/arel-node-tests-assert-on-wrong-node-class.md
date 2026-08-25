---
title: "arel-node-tests-assert-on-wrong-node-class"
status: ready
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while scoping `arel-assertion-mark-to-zero` (RFC 0122). A family of
arel node tests carries a body copy-pasted from a _different_ node's test file,
so the test never exercises the node its file is named for. The test names are
correct Rails mirrors — only the bodies are wrong — which is why
`parity:test` scores them 100% on test presence while the assertion
dimensions stay red.

Confirmed instances (`grep -A4` for the shared equality test names, then the
`Nodes.X` each body actually constructs):

| trails test file                           | body actually constructs               |
| ------------------------------------------ | -------------------------------------- |
| `packages/arel/src/nodes/count.test.ts`    | `Nodes.TableAlias`                     |
| `packages/arel/src/nodes/sum.test.ts`      | `Nodes.NamedFunction`                  |
| `packages/arel/src/nodes/extract.test.ts`  | `Nodes.Not`                            |
| `packages/arel/src/nodes/grouping.test.ts` | `Nodes.Window`                         |
| `packages/arel/src/nodes/comment.test.ts`  | `Nodes.SqlLiteral`                     |
| `packages/arel/src/attributes.test.ts`     | `Nodes.NamedFunction`                  |
| `packages/arel/src/nodes/distinct.test.ts` | `Nodes.False` / `Nodes.True`           |
| `packages/arel/src/nodes/true.test.ts`     | `Nodes.CurrentRow` / `Nodes.Preceding` |

Worked example — `count.test.ts:32-36` is

```ts
it("is not equal with different ivars", () => {
  const a = new Nodes.TableAlias(users, "u");
  const b = new Nodes.TableAlias(users, "v");
  expect(a.name).not.toBe(b.name);
});
```

against `vendor/rails/activerecord/test/cases/arel/nodes/count_test.rb`:

```ruby
it "is not equal with different ivars" do
  array = [Arel::Nodes::Count.new("foo"), Arel::Nodes::Count.new("foo!")]
  assert_equal 2, array.uniq.size
end
```

Two divergences per body: the wrong node class, and an `a.name != b.name`
field probe standing in for Rails' `uniq.size` assertion, which is what
actually exercises the `hash` / `eql?` contract the test exists to pin. The
same substitution is why these tests show as
`equal rails 1 vs trails 0, notEqual rails 0 vs trails 1` in
`pnpm parity:test -- --assertions --package arel --missing`.

This is a prerequisite for `arel-assertion-mark-to-zero`: those bodies cannot
reach assertion parity without being rewritten against the right node, and the
rewrite is a correctness fix independent of the mark.

## Acceptance criteria

- [ ] Every listed file's shared equality tests (`is equal with equal ivars`,
      `is not equal with different ivars`, `is not equal with other nodes`,
      `is not equal with different contents`) construct the node the file is
      named for.
- [ ] Each body mirrors its Rails counterpart's assertion shape — Rails'
      `assert_equal 2, array.uniq.size` ports as a `uniq`-equivalent length
      assertion, not an `a.name !== b.name` field probe.
- [ ] **No test name is renamed or reworded.**
- [ ] The list above is verified exhaustive: re-run the `grep -A4` sweep over
      `packages/arel/src/**/*.test.ts` and fix any further instance found.
- [ ] `pnpm parity:test` deltas non-negative; arel's three assertion
      dimensions in `scripts/test-compare/assertion-mismatch-mark.json` move
      DOWN, never up.
