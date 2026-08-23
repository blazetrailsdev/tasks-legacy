---
title: "habtm-public-reflection-is-built-directly-not-via-the-has-many-macro"
status: done
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6900
claim: "2026-08-23T01:58:40Z"
assignee: "wave-5g-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

`has_and_belongs_to_many` (`vendor/rails/activerecord/lib/active_record/associations.rb:1904`)
finishes its macro body by re-entering the `has_many` macro:

```ruby
has_many name, scope, **hm_options, &extension
_reflections[name].parent_reflection = habtm_reflection
```

so the public reflection registered under `name` is an ordinary
`HasManyReflection` (wrapped in a `ThroughReflection` by `:through`), with
`parent_reflection` pointing at the `HasAndBelongsToManyReflection`.

trails' `Model.hasAndBelongsToMany`
(`packages/activerecord/src/associations.ts`, the macro body relocated there by
RFC 0112 `habtm-macro-body-lives-in-the-builder`) does not re-enter `hasMany`.
It builds the public reflection directly:

```ts
const habtmReflection = Reflection.create(
  "hasAndBelongsToMany",
  name,
  positionalScope,
  habtmOptions,
  self,
);
Reflection.addReflection(self, name, habtmReflection);
CollectionAssociationBuilder.defineAccessors(self, habtmReflection);
```

i.e. the macro on the reflection is `"hasAndBelongsToMany"` where Rails' is
`"has_many"`. That is why the body carries a `@missingRailsCall has_many`
receipt. The macro string is read across reflection walking, join planning and
`_resolveHabtmJoin`, so flipping it is not a local edit.

## Acceptance criteria

- [ ] `Model.hasAndBelongsToMany` calls `this.hasMany(name, scope, hmOptions)`
      for the public association, as `associations.rb:1904` does, instead of
      constructing the reflection itself.
- [ ] `_reflections[name].parentReflection = habtmReflection` still lands
      (`associations.rb:1905`).
- [ ] Every consumer that branches on `reflection.macro === "hasAndBelongsToMany"`
      is audited and moved to the `parentReflection` link Rails uses.
- [ ] The `@missingRailsCall has_many` receipt on `hasAndBelongsToMany`
      (`packages/activerecord/src/associations.ts`) is deleted, not reworded.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows; HABTM suites
      green on SQLite, PostgreSQL and MySQL/MariaDB.

- [ ] The macro constructs `HasAndBelongsToManyReflection` as its own object
      (`associations.rb:1871`) and points both
      `middleReflection.parentReflection` and `_reflections[name]
.parentReflection` at it, instead of reusing the public reflection. The
      `@missingRailsArgs new` receipt on `hasAndBelongsToMany` is deleted then.
