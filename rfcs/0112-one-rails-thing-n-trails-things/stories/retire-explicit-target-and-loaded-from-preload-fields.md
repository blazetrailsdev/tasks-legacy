---
title: "retire-explicit-target-and-loaded-from-preload-fields"
status: ready
updated: 2026-08-24
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

`Association#_explicitTarget` and `Association#_loadedFromPreload`
(`packages/activerecord/src/associations/association.ts:105`, `:115`) are
trails-only fields with no counterpart on Rails' `Association`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb`),
which distinguishes the same cases with `@loaded` alone (`loaded?` / `loaded!`)
plus `CollectionAssociation#@target`.

They are read by `_loadedSingularTarget` and `_preloadedHolderTarget`
(`packages/activerecord/src/associations.ts`), and written from
`Association#setInverseInstance` / `#setInverseInstanceFromQueries`
(`association.ts:452-490`), `join-dependency.ts:1053-1093`,
`preloader/association.ts:213-250`, `preloader/batch.ts:98-99` and
`support/seed-association-cache.ts:25-26`.

Split out of `cache-singular-target-inverse-write-bypasses-inversed-from`
(PR TBD), which converged the one item its acceptance criteria named — the
bespoke `_setTargetFromLoader` + `_explicitTarget` write inside
`_cacheSingularTarget`, now `existing?.inversedFrom(target)` — and left the
fields themselves standing. Retiring them touches ~8 files and both the
preload and eager-load readers, so it does not fit alongside that change.

## Acceptance criteria

- [ ] `_explicitTarget` and `_loadedFromPreload` are retired in favour of Rails'
      `loaded?` state, with `_loadedSingularTarget` / `_preloadedHolderTarget`
      rewritten onto it — or, if a genuine TypeScript/async shortcoming forces
      one to survive, it carries a `@noRailsEquivalent` whose reason names that
      shortcoming.
- [ ] The inverse-of, preload, eager-load, strict-loading and autosave suites
      stay green on SQLite, PostgreSQL and MySQL/MariaDB.
