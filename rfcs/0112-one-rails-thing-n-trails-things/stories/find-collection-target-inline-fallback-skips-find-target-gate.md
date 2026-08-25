---
title: "find-collection-target-inline-fallback-skips-find-target-gate"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
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

Surfaced by review of #7053 (`converge-find-target-reachable-onto-find-target-gate`).

trails lets a test build an association holder from a bare options hash for a
name the owner never declared — the "inline fallback" shape, e.g.
`findHasManyTarget(author, "destroy_all_posts", { className: "...", foreignKey:
"..." })` (`packages/activerecord/src/associations/has-many-associations.test.ts:951`,
and ~40 more call sites in that file). Rails has no such thing:
`Association#initialize` takes a reflection
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:41-45`),
and every predicate downstream reads it.

Since #7039 made `Association#klass` the reflection delegate, such a holder
answers `klass === undefined`. `find_target?` ends in `&& klass`
(`association.rb:320-321`), so `findTargetNeeded()` is unconditionally false
for the inline shape — the gate cannot be evaluated for it at all.

PR #7053 therefore reads the `find_target?` gate off the owner's _real_ holder
when the name is declared, and skips the gate for the undeclared inline shape
(`packages/activerecord/src/test-helpers/find-collection-target.ts:40-56`).
That is correct today — no test drives an inline association with a new-record
owner and no foreign key — but it is an asymmetry: a future inline test in that
position would get no `find_target?` suppression, where the declared path would.

## Converged shape

The inline fallback goes away: every `findCollectionTarget` call site names an
association the owner actually declares, so the helper always has a reflection
and the gate is read off it unconditionally — no `declared` branch. The ~40
bespoke inline call sites in `has-many-associations.test.ts` are the burndown.

Alternatively, if some call sites must keep an options set that is not the
declared one, the ad hoc holder needs to resolve `klass` from
`options.className` the way the loaders already do via `resolveAssocClass`, so
`find_target?` is answerable for it.

## Acceptance criteria

- [ ] `findCollectionTarget` consults `find_target?` on every path — the
      `declared` branch in `find-collection-target.ts` is gone.
- [ ] No call site builds a holder for a name the owner has not declared, or
      the reflection-less holder resolves `klass`.
- [ ] `has-many-associations.test.ts`, `associations.test.ts` and
      `strict-loading.trails.test.ts` stay green, test names unchanged.
