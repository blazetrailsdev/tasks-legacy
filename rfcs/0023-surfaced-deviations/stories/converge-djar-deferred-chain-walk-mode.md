---
title: "DisableJoinsAssociationRelation carries a trails-only deferred chain-walk mode Rails has no second arm for"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 450
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `DisableJoinsAssociationRelation` has ONE mode. It is constructed with
`(klass, key, ids)` after `DisableJoinsAssociationScope#scope` has already
finished the chain walk synchronously
(`activerecord/lib/active_record/associations/disable_joins_association_scope.rb:7-19`),
so the relation only ever loads a single SELECT and regroups it in `load`
(`activerecord/lib/active_record/disable_joins_association_relation.rb:26-38`).

trails' `DisableJoinsAssociationRelation`
(`packages/activerecord/src/disable-joins-association-relation.ts`) carries a
SECOND, trails-only mode: a `_chainWalker` callback plus `_walkOnce()`,
`_composeChainedState()` and a `TRUSTED_CLONE` fast-clone payload, because
trails' chain walk is async and `DJAS.scope()` must return a Relation
synchronously. Every Rails-named method on the class is now a two-armed
`if (this._chainWalker) ... else ...`: `execQueries`, `load`, `limit`, `first`,
`ids`, `count`, `pluck`.

PR #6912 (converge-disable-joins-association-relation-onto-load) moved the
loaded-chain arm onto Rails' `load` and the deferred walk into `execQueries`,
which is as far as the split can converge while the second mode exists. The
deferred arm of `execQueries` additionally has to strip `limitValue` /
`offsetValue` off a walked scope that turns out to be a loaded-chain DJAR and
slice in memory, because trails' `.limit()` landed on the deferred DJAR (which
routes to `Relation.prototype.limit`) where Rails' would have landed on the
final relation and hit the in-memory `limit` override
(disable_joins_association_relation.rb:13-24). That whole compensation exists
only because of the deferred mode.

## Converged shape

Delete the deferred-chain mode. `DisableJoinsAssociationScope#scope` performs
its walk before handing back a relation, so the DJAR it constructs is Rails'
`(klass, key, ids)` shape and nothing else; `execQueries` loses its override
entirely, and `load`, `limit`, `first`, `ids`, `count`, `pluck` each collapse to
their single Rails arm. `_chainWalker`, `_walkOnce`, `_composeChainedState`,
`TRUSTED_CLONE`/`TrustedClonePayload` and the `clone()` override built for them
go with it.

The blocker to be solved is the sync/async seam: trails' intermediate plucks are
async, so either `DJAS.scope()` becomes async (and its callers with it), or the
walk is deferred one level UP — inside the association scope machinery — rather
than inside the relation Rails names.

## Acceptance criteria

- [ ] `DisableJoinsAssociationRelation` has no `_chainWalker` arm; every method
      body is Rails' single arm.
- [ ] No `execQueries` override on the class, and no limit/offset stripping.
- [ ] `pnpm parity:api:extra --package activerecord` loses the DJAR-only novel
      names (`_walkOnce`, `_composeChainedState`, `ids`-shape helpers).
- [ ] Existing disable-joins tests stay green on all three lanes.
