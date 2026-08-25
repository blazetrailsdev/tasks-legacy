---
title: "Arel::Nodes::Node is instantiable in Rails but abstract in trails, forcing casts in mirrored tests"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
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

Surfaced while converging arel's node tests in PR #7057 (RFC 0122).

`Arel::Nodes::Node` is an ordinary instantiable class in Rails —
`vendor/rails/activerecord/lib/arel/nodes/node.rb:116`, `class Node` — and
Rails' tests instantiate it directly as the "some other node" foil:

```ruby
# vendor/rails/activerecord/test/cases/arel/nodes/distinct_test.rb
array = [Distinct.new, Node.new]
assert_equal 2, array.uniq.size
```

`false_test.rb`, `true_test.rb`, `window_test.rb` (CurrentRow) and `node_test.rb`
do the same.

`packages/arel/src/nodes/node.ts:26` declares it `export abstract class Node`.
Nothing in the class is actually abstract — there is no abstract member — so the
modifier is a port decision, not a language requirement, and it makes Rails'
`Node.new` untypable. The mirrored tests had to write:

```ts
const array = [new Nodes.Distinct(), new (Nodes.Node as unknown as new () => Node)()];
```

which is a double cast at four call sites purely to spell a constructor Rails
has. It also runs fine — `abstract` is erased at runtime — so the cast is
buying nothing but noise.

## Converged shape

Drop `abstract` from `packages/arel/src/nodes/node.ts:26` unless a concrete
abstract member is added, and replace the `as unknown as new () => Node` casts
in `packages/arel/src/nodes/{distinct,false,true,window}.test.ts` with plain
`new Nodes.Node()`.

Before flipping it, confirm no subclass relies on `abstract` to force an
override (`grep -n "abstract " packages/arel/src/nodes/node.ts`) — if one does,
that member is the thing to converge, not the class modifier.

## Acceptance criteria

- [ ] `Arel::Nodes::Node` is instantiable in TS, as it is in Ruby, or the
      `abstract` modifier is justified by a real abstract member with a Rails
      citation.
- [ ] The four `as unknown as new () => Node` casts are gone and the tests read
      `new Nodes.Node()`.
- [ ] `pnpm parity:api:extra:gate` green and `pnpm vitest run packages/arel` green.
