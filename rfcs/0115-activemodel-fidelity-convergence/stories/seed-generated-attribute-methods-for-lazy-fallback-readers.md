---
title: "Seed GeneratedAttributeMethods for classes reached only through isInstanceMethodAlreadyImplemented"
status: done
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: 6962
claim: "2026-08-24T01:24:28Z"
assignee: "anchor-jsdoc-tag-recognition-to-line-start"
blocked-by: null
closed-reason: null
---

# Seed `GeneratedAttributeMethods` for classes reached only through `isInstanceMethodAlreadyImplemented`

## Context

Follow-on from `seed-generated-attribute-methods-before-class-body` (landed in
trails#6959), which retired `uninclude()` by seeding the
`GeneratedAttributeMethods` at the AR-owned entry points a class body passes
through — `Base.attribute` (`packages/activerecord/src/base.ts`),
`aliasAttribute` and `generateAliasAttributes`
(`packages/activerecord/src/attribute-methods.ts`).

Those cover every path a class body can take, but not every path into
ActiveModel's lazy `generated_attribute_methods`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:400-402`, ours
`packages/activemodel/src/attribute-methods.ts:524`). One reader still gets
there first for a class that declares nothing of its own:

    isInstanceMethodAlreadyImplemented
      (activemodel/lib/active_model/attribute_methods.rb:404-406,
       ours packages/activemodel/src/attribute-methods.ts:538)

Measured instance: an empty `class Leaf extends Middle {}` in
`attribute-methods.trails.test.ts` ("an inherited generated alias does not
suppress the subclass's own generation" and its dirty-accessor sibling) calls
`Leaf.isInstanceMethodAlreadyImplemented(...)` directly, which seats a bare
`Module` on `Leaf._generatedAttributeMethods`.

In Rails this cannot happen: `initialize_generated_modules` runs from
`inherited` (`activerecord/lib/active_record/attribute_methods.rb:265-272`), so
`@generated_attribute_methods` is a `GeneratedAttributeMethods`
(`:41-47`) for every class from the moment it exists. Since #6959 removed the
replacement branch, a class that reaches the lazy fallback first now KEEPS the
bare `Module` for its whole life rather than being upgraded.

Today the only observable difference is `inspect()` —
`GeneratedAttributeMethods#inspect` answers
`"#{ownerName}::GeneratedAttributeMethods"` (`attribute-methods.ts:183-185`,
the trails stand-in for Rails' `const_set(:GeneratedAttributeMethods, ...)`,
`:43`) and a bare `Module` does not — plus any future member added to the
subclass. It is still a latent divergence, and it is the last hole in the
seeding story.

## Converged shape

Seat the `GeneratedAttributeMethods` for an AR class before ANY reader of
`generatedAttributeMethods()` can seat a bare one, so
`_generatedAttributeMethods` is a `GeneratedAttributeMethods` for every AR
class unconditionally — the invariant Rails' `inherited` hook gives for free.

Also retire the now-misnamed trails test
`initializeGeneratedModules replaces a module ActiveModel built first`
(`packages/activerecord/src/attribute-methods.trails.test.ts:40`): the
replacement branch it is named for no longer exists, and what the body actually
asserts (a class-body `attribute` + `aliasAttribute` leaves one
`GeneratedAttributeMethods` answering the alias) is the seeding invariant.

## Acceptance criteria

- [ ] `_generatedAttributeMethods` is a `GeneratedAttributeMethods` for an AR
      class that declares nothing and is only ever reached through
      `isInstanceMethodAlreadyImplemented`.
- [ ] No AR class can hold a bare `Module` under `_generatedAttributeMethods` —
      demonstrated by a guard that fails on the pre-fix build.
- [ ] The `replaces a module ActiveModel built first` test is renamed to what it
      now asserts, or removed if a sibling already covers it.
- [ ] `attribute-methods.test.ts`, `attribute-methods.trails.test.ts` and
      `base.test.ts` green on SQLite, PostgreSQL and MySQL/MariaDB.
