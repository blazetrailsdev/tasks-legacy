---
title: "has_and_belongs_to_many's whole macro body lives in Builder::HasAndBelongsToMany._build, a method Rails' builder does not have"
status: claimed
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 240
priority: null
pr: null
claim: "2026-08-23T00:57:31Z"
assignee: "wave-5g-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

Rails' `has_and_belongs_to_many` macro
(`vendor/rails/activerecord/lib/active_record/associations.rb:1870-1905`) builds
the HABTM **in the macro body**: it constructs the `HasAndBelongsToManyReflection`
and the `Builder::HasAndBelongsToMany`, then calls `builder.through_model`,
`const_set` / `private_constant`, `builder.middle_reflection`,
`Builder::HasMany.define_callbacks`, `Reflection.add_reflection`,
`include Module.new { def destroy_associations … end }` and finally
`has_many name, scope, **hm_options, &extension`.

trails' `Model.hasAndBelongsToMany`
(`packages/activerecord/src/associations.ts:789-830`) does none of that: it
normalizes the options and hands everything to `HabtmBuilder.build`, which runs
the whole Rails macro body inside
`HasAndBelongsToMany._build`
(`packages/activerecord/src/associations/builder/has-and-belongs-to-many.ts:255-414`).
Rails' builder has no `_build` — `Builder::HasAndBelongsToMany` only supplies
`through_model` / `middle_reflection` / `middle_options`, and the macro is the
frame that sequences them.

That relocation is the whole reason five `@missingRailsCall` receipts sit on
`hasAndBelongsToMany` today (`add_reflection`, `define_callbacks`, `has_many`,
`include`, `new`), migrated out of the RFC 0047 seed baseline by the RFC 0106
wave-5e head sweep. Each says the same thing: the call is made one frame down.

## Acceptance criteria

- [ ] `Model.hasAndBelongsToMany` sequences the Rails macro body itself:
      reflection construction, `throughModel()`, the registry `const_set` +
      `privateConstant`, `middleReflection()`, the define-callbacks call,
      `Reflection.addReflection`, the `destroyAssociations` override, the
      `hmOptions` allowlist, and the through-`hasMany` registration.
- [ ] `HasAndBelongsToMany` keeps only the members Rails' builder has —
      `throughModel`, `middleReflection`, `middleOptions` and the derivation
      helpers; `_build` (and the `build` static that only forwards to it) is
      gone.
- [ ] The five `@missingRailsCall` receipts on `hasAndBelongsToMany`
      (`packages/activerecord/src/associations.ts`) are deleted, not reworded.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows; HABTM suites
      green on SQLite, PostgreSQL and MySQL/MariaDB.
