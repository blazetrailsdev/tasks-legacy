---
title: "AssociationScope _mergeReferencedJoins reimplements merger.rb instead of delegating to the join folders"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AssociationScope#_mergeReferencedJoins`
(`packages/activerecord/src/associations/association-scope.ts:1006-1078`)
hand-reimplements Rails `Relation::Merger#merge_joins` / `#merge_outer_joins`
(`vendor/rails/activerecord/lib/active_record/relation/merger.rb:117-152`)
inline: it recomputes the `other.model == relation.model` branch, partitions
association specs from an already-built `JoinDependency`, and builds the
cross-klass JD itself.

Those exact branches now live — matching merger.rb line for line — in
`foldMergeJoins` / `foldMergeOuterJoins`
(`packages/activerecord/src/relation/merge-joins.ts:36-100`), which PR #5737
restructured onto merger.rb's own shape (early `return` on empty source, the
same-klass union branch, the else-branch `partition` into
`associations`/`others`, and the `joins!(join_dependency, *others)`-style union
via `joinsUnionEq`).

Two copies of the same Rails method now exist. They already differ in detail:
`_mergeReferencedJoins` dedups same-klass inner specs with
`target._namedInnerJoins.includes(v)` (JS reference identity) where the folder
uses `structuralUnionEq` (Ruby `eql?`/`hash` semantics), and it has no
`joinsUnionEq` union on the cross-klass push. Any future merger.rb convergence
has to be applied twice or the copies drift.

Rails has no `_mergeReferencedJoins`: `AssociationScope` reaches the same
behaviour by routing `merge! item.only(:joins, :left_outer_joins)` through the
one `Merger`.

## Acceptance criteria

- `_mergeReferencedJoins` delegates its joins / left_outer_joins folding to
  `foldMergeJoins` / `foldMergeOuterJoins` rather than reimplementing the
  branches, so merger.rb is ported exactly once.
- The same-klass inner dedup goes through `structuralUnionEq`, matching the
  folder (and Ruby `|=`), not reference identity.
- The trails-only eager/includes `associations` push at the tail
  (`association-scope.ts:1067-1077`) is either kept with a justification at the
  call site or converged; it has no merger.rb counterpart.
- Ported association / through-scope tests pass unchanged, no test renames.

## Update from the 2026-08-18 RFC 0023 triage pass — the convergence TARGET moved

`packages/activerecord/src/relation/merge-joins.ts` no longer exists, and with
it `foldMergeJoins`, `foldMergeOuterJoins` and `joinsUnionEq`. Do **not** go
looking for that file.

The Rails-faithful bodies now live on `Merger` itself, in
`packages/activerecord/src/relation/merger.ts`:

- `Merger#mergeJoins` (`merger.ts:145-178`) — the same-model branch is now one
  `structuralUnionEq` union over the whole `joinsValues` store
  (`merger.rb:121`), and the cross-model branch partitions into
  `associations` / `others` and calls
  `QueryMethodBangs.joinsBang(rel, joinDependency, ...others)`
  (`merger.rb:123-137`).
- `Merger#mergeOuterJoins` (`merger.ts:181-`) — same shape for
  `leftOuterJoinsValues` (`merger.rb:140-152`).

`AssociationScope#_mergeReferencedJoins` (`association-scope.ts:971-1050`,
called from `:914`) still hand-reimplements both, including its own
`item._model === target._model` same-klass test and its own
`namedInner`/`others` partition. **Converge it onto the `Merger` methods above**
rather than onto the deleted folders.

Note the one genuinely trails-only piece it also carries — copying
`item._joinClauses` across — which is the separate
`fold-join-clauses-into-joins-values` story; leave that side-channel alone here.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
