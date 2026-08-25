---
title: "Converge schema invalidation onto Rails' push-only DescendantsTracker model (eager subclass registration, delete the per-read pull fallback)"
status: blocked
updated: 2026-08-25
rfc: "0123-blocked-convergence-holding"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: "2026-08-18T18:14:58Z"
assignee: "port-test-date-strftime-different-format"
blocked-by: "Re-verified 2026-08-24 against origin/main. The OLD blocker reason is now factually wrong and was replaced: git grep for schemaStaleAgainstAncestors, _staleCheck, _schemaRevision, schemaEpoch and blazetrails/schema-memo-read-through-guard returns ZERO hits on origin/main — #6809 deleted all of them, and the story body already records that (ACs 2 and 3 are checked off). What is still genuinely blocked is the ORIGINAL open problem, unchanged: eager subclass registration. 'class X extends Y {}' offers no receiver-bearing hook — [[SetPrototypeOf]] fires on X before Y can see it, and a proxied-Base 'prototype' get trap fires before the class object exists — so registerSubclass (inheritance.ts:397) stays lazy, and Inheritance#subclasses (inheritance.ts:90) still unions two hand-filled registries where Rails delegates to the VM-maintained Class#subclasses (descendants_tracker.rb:97-100). AC1, AC4 and AC5 all hinge on that. Still needs an RFC 0078 owner decision on the second-best shape (proxied-Base get trap, codegen/lint-enforced registration at every extends, or ratifying the two-registry union). Worth a cheap check when unblocking: AC4 says the explicit registerSubclass(Circle) in model-schema-sync-load.test.ts may already be redundant after #6809's union."
closed-reason: null
---

## Context

Residual left by [[sti-schema-stale-invariant-unenforced]] (PR #6705), which
took that story's **"Enforce the invariant"** branch: a new eslint rule,
`blazetrails/schema-memo-read-through-guard` (`eslint/`), now flags any raw read
of `_schemaLoaded` / `_columnsHash` / `_columns` / `_attributesBuilder` /
`_virtualAttributesReconciled` in `packages/activerecord/src/*.ts` that does not
route through `ownSchemaMemo` / `isSchemaLoaded`. That makes the pull fallback
**safe**, but it does not make it **Rails**.

Rails invalidates schema state by pushing DOWN through `DescendantsTracker`,
which Ruby's `inherited` hook populates the moment a subclass is defined
(`vendor/rails/activerecord/lib/active_record/model_schema.rb:553-568`;
`descendants` via `ActiveSupport::DescendantsTracker`). There is no per-read
staleness check anywhere in Rails — invalidation is push-only.

trails carries an extra apparatus Rails has no counterpart for:

- `schemaStaleAgainstAncestors` (`packages/activerecord/src/model-schema.ts:74`)
  — a prototype-chain walk run on EVERY schema-memo read, including the
  `new Model()` hot path.
- its `_staleCheck` epoch memo, and the global `_schemaRevision` epoch it
  compares against.
- `ownSchemaMemo` (`model-schema.ts:99`) exists only to apply that walk.

It is there because `registerSubclass` (`packages/activerecord/src/inheritance.ts`)
is LAZY — triggered by `attribute()` / `decorate_attributes()` /
`_defaultAttributes()` / association declarations, not by `class X extends Y {}`
— so `reloadSchemaFromCache`'s recursive push (`model-schema.ts:920-922`)
reaches only subclasses that happened to register.

## Converged shape

Make STI subclass registration EAGER so the recursive push reaches every
descendant, as Rails' `inherited` hook does, and then delete the pull apparatus
outright: `schemaStaleAgainstAncestors`, `_staleCheck`, the `_schemaRevision`
epoch, and `ownSchemaMemo`'s staleness arm (the own-property check stays — Ruby
class ivars are not inherited and JS statics are).

The eslint rule added by #6705 becomes unnecessary at that point and should be
deleted with it; it is scaffolding for an invariant that would no longer exist.

The open question is the eager-registration hook. JS has no `inherited`, and
PR #6705 did not find a mechanism that runs per `class X extends Y {}` without a
decorator or an explicit call — that is the thing to solve here. If it genuinely
cannot be solved in the JS object model, `pnpm tasks block` this with the
specific blocker at a trails `file:line`; do NOT close it by re-justifying the
pull fallback.

## Superseded in part by PR #6809 (2026-08-21)

The blocker below concluded the pull apparatus "cannot be deleted" without an
eager-registration hook. That is now falsified: #6809 deleted
`schemaStaleAgainstAncestors`, `_staleCheck`, the `_schemaRevision` epoch, `ownProp`
and the eslint rule, via a route the analysis did not consider — trails has a
**second** subclass registry, the `ActiveSupport::DescendantsTracker` that
`_defaultAttributes` registers into (`packages/activemodel/src/attribute-registration.ts`),
which covers classes `Inheritance`'s `_subclasses` misses. `Inheritance#subclasses`
(`packages/activerecord/src/inheritance.ts:90`) now reads both, so
`reloadSchemaFromCache`'s recursive push reaches them and the per-read prototype
walk is gone from the `new Model()` hot path.

What remains is the deviation #6809 shipped in its place, and it is what this
story should now converge:

**Rails has exactly ONE registry.** Ruby's `Class#subclasses` is VM-maintained,
so `DescendantsTracker.subclasses(klass)` is a plain delegation to it
(`vendor/rails/activesupport/lib/active_support/descendants_tracker.rb:97-100`),
and `inherited` fills it the moment `class X < Y` is evaluated. trails fills two
by hand and unions them at read time. The union is a bridge, not a mirror: it is
still lazy (both registries are filled by later calls, not by `extends`), so a
subclass that never triggers either one is still invisible to the push.

The converged shape is unchanged — one registry, eagerly filled — and the
eager-registration hook is still the open problem. The blocker analysis below
remains accurate about `class X extends Y {}` offering no receiver-bearing hook;
it is only its conclusion about the pull apparatus that #6809 overtook.

## Acceptance criteria

- [ ] STI subclass registration is eager, so `reloadSchemaFromCache`'s recursive
      push reaches every descendant without an explicit `registerSubclass` call.
- [x] `schemaStaleAgainstAncestors`, `_staleCheck` and the `_schemaRevision`
      epoch are deleted; per-read prototype walking is gone from the
      `new Model()` hot path. (#6809)
- [x] `blazetrails/schema-memo-read-through-guard` and its test are deleted, and
      the `eslint.config.mjs` block with them. (#6809)
- [ ] `model-schema-sync-load.test.ts`'s "resetting the STI base propagates to
      subclasses" no longer needs its explicit `registerSubclass(Circle)` call —
      that call is the visible symptom of the push side being partial.
      (Unverified after #6809: the two-registry union may already have made it
      redundant. Check before assuming work is needed.)
- [ ] `Inheritance#subclasses` reads ONE registry, not a union of two
      (`inheritance.ts:90`), and `descendants` recurses through it.
- [ ] parity:api and parity:test deltas non-negative.
