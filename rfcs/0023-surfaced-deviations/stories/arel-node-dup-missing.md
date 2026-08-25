---
title: "Arel nodes have no dup(), so Rails' mutation tests are ported as a weaker twin comparison"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

Surfaced while converging arel's node tests in PR #7057 (RFC 0122).

Ruby gives every `Arel::Nodes::Node` a shallow `Object#dup`, and Rails' arel
tests lean on it to snapshot a node before a mutating call.
`vendor/rails/activerecord/test/cases/arel/nodes/extract_test.rb:21-26`:

```ruby
it "should not mutate the extract" do
  table = Arel::Table.new :users
  extract = table[:timestamp].extract("date")
  before = extract.dup
  extract.as("foo")
  assert_equal extract, before
end
```

`packages/arel/src/nodes/node.ts:26` declares `export abstract class Node` with
no `dup`, and no arel node carries one (`grep -rn "dup(" packages/arel/src` over
non-test sources finds nothing). The ported test therefore substitutes an
untouched twin built the same way:

```ts
const extract = users.get("timestamp").extract("date");
const before = users.get("timestamp").extract("date");
extract.as("foo");
expect(extract.eql(before)).toBe(true);
```

That passes, but it does not test what Rails tests: a twin proves the two
constructions agree, whereas `dup` proves the ORIGINAL object was not mutated in
place. A node whose `as()` mutated a shared sub-node would still satisfy the
twin form.

## Converged shape

Give `Arel::Nodes::Node` a `dup()` mirroring Ruby's shallow copy — same class,
same own slots, no deep copy of children — then restore
`packages/arel/src/nodes/extract.test.ts`'s "should not mutate the extract" to
Rails' body verbatim (`const before = extract.dup();`).

`node.ts` already carries the slot machinery (`node-slots.ts`), so the copy
should go through it rather than a bespoke property walk. Check Ruby's
`initialize_copy` semantics for the nodes that own mutable arrays
(`Nodes::Window#orders` / `#partitions`, `SelectStatement#cores`) before
deciding how far the shallowness reaches — Rails' own clone tests
(`select_statement_test.rb`, `insert_statement_test.rb`,
`update_statement_test.rb`, `delete_statement_test.rb`) pin that behaviour and
are still on the RFC 0122 residue list, so this story and
`arel-assertion-residue-to-zero` should agree on it.

## Acceptance criteria

- [ ] `Node#dup` exists with Ruby's shallow-copy semantics and is exercised by
      the Rails clone tests, not only by extract's.
- [ ] `packages/arel/src/nodes/extract.test.ts`'s "should not mutate the extract"
      is Rails' body, with the substituted-twin workaround and its call-site
      comment removed.
- [ ] `pnpm parity:api:extra:gate` green — `dup` is Rails-backed, so arel's
      novel count must not move.
- [ ] `pnpm vitest run packages/arel` green.
