---
title: "Converge the through chain onto one JoinAssociation per reflection"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Converge the `through` chain onto one JoinAssociation per reflection

## Context

Surfaced converging `JoinDependency#build` in PR #6578, which ported
`build(associations, base_klass)` onto Rails' recursive node tree but left the
through-chain node model as it was.

`vendor/rails/activerecord/lib/active_record/associations/join_dependency.rb:228-240`
builds exactly ONE `JoinAssociation` per reflection, including a
`through` reflection. The chain's intermediate joins are emitted at
constraint time by
`vendor/rails/activerecord/lib/active_record/associations/join_dependency/join_association.rb`
(`join_constraints` walks `reflection.chain.reverse_each` and emits one join
per link), so the TREE never contains a node for a through link.

trails' `_addThroughViaJoinAssociation`
(`packages/activerecord/src/associations/join-dependency.ts`) instead
materializes one tree node per chain link — the target plus a synthetic
`JoinLeaf` per link, named `_through_<reflection name>` — pushed as SIBLINGS of
the target under the same parent, with a shared `ThroughJoinGroup` resolving
their aliases at emit. That is why `build` has to `flatMap` and pick
`nodes[nodes.length - 1]` as the attach point for nested children, where Rails
maps 1:1 and returns the node it just constructed.

The synthetic leaves also leak: `relation.ts` skips nodes whose
`immediateAssocName` starts with `_through_` when wiring preloaded proxies, and
`aliases()` has to reason about which tree nodes are real reflected nodes.

## Converged shape

A `through` reflection produces ONE `JoinAssociation` node, as
join*dependency.rb:239 does; the chain's intermediate joins are emitted from
`JoinAssociation#joinConstraints` at emit time without any tree node standing
for them. `build`'s body then maps 1:1 —
`new JoinAssociation(reflection, this.build(right, reflection.klass))` — with no
flatMap and no last-element attach point, and the `\_through*`name prefix and
the`ThroughJoinGroup`sibling bookkeeping disappear along with the`JoinLeaf` class.

## Acceptance criteria

- [ ] `_addThroughViaJoinAssociation` returns a single node (or is inlined into
      the node construction `build` already does).
- [ ] No `_through_`-prefixed tree nodes; `relation.ts`'s `_through_` skip and
      the `JoinLeaf` class are deleted.
- [ ] `build` maps 1:1 over the associations hash, mirroring
      join_dependency.rb:228-240 with no flatMap.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Absorbed: `converge-through-chain-into-one-join-association-node`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Fold a through chain into one JoinAssociation node so reflections needs no filter"

### Context

Rails represents a `has_many :through` as ONE `JoinAssociation` node whose
`reflection.chain` carries the hops
(`activerecord/lib/active_record/associations/join_dependency.rb:81-83` reads
`join_root.drop(1).map!(&:reflection)` over exactly those nodes; every non-root
node has a reflection).

trails materializes each chain hop as its own tree node: the non-target hops are
`JoinLeaf`s named `_through_<name>`
(`packages/activerecord/src/associations/join-dependency.ts`,
`_addThroughViaJoinAssociation`), and they carry no `reflection`. That forces
`JoinDependency#reflections` (same file) to append a
`.filter((reflection) => reflection != null)` that Rails does not have — an
otherwise faithful `drop(1).map(&:reflection)` cannot be written without it.

### Converged shape

Fold a through chain back into a single `JoinAssociation` tree node whose
`reflection` is the through reflection, with the hops living in the reflection's
chain the way `JoinAssociation#join_constraints`
(`join_dependency/join_association.rb`) already walks them. Then drop the
`.filter` from `reflections` so the body is Rails' one line.

Note the emit-time machinery (`ThroughJoinGroup`, `_resolveThroughGroup`,
`tableIndex = -1` for reused chain tails) is keyed on the per-hop nodes, so this
is a real restructure, not a rename.

### Acceptance criteria

- [ ] A through association is one tree node carrying the through reflection.
- [ ] `reflections` is `joinRoot.drop(1).map((node) => node.reflection)`, no filter.
- [ ] Join SQL and aliasing for through/nested-through eager loads unchanged.
