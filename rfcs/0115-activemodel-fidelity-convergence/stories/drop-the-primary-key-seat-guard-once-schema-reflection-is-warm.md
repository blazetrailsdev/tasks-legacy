---
title: "Seat @primary_key unconditionally and delete isPrimaryKeySettled"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: 6956
claim: "2026-08-23T22:12:31Z"
assignee: "date-suite-is-not-run-by-any-ci-job"
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Core#init_internals`
(`vendor/rails/activerecord/lib/active_record/core.rb:846`) seats the ivar
unconditionally:

```ruby
@primary_key = klass.primary_key
```

PR #6934 (story `seat-the-per-instance-primary-key-slot`) ported that, but had
to guard it: trails reflects the schema asynchronously, so
`getPrimaryKey` (`packages/activerecord/src/attribute-methods/primary-key.ts`)
answers the `"id"` convention until the schema cache is warm, and seating that
would latch it for the record's lifetime. The guard is `isPrimaryKeySettled`
(same file) — true only for an explicitly configured key or one the schema
cache has reflected — and `primaryKeyOf` (primary-key.ts, and its twin in
`attribute-methods/composite-primary-key.ts`) reads through to the class while
the slot is unseated.

So trails has three shapes Rails does not: a guard, a fallback branch in each
`primaryKeyOf`, and a record whose `@primary_key` may be absent.

## Converged shape

`initInternals` assigns `this._primaryKey = klass.primaryKey` with no guard, and
both `primaryKeyOf` bodies are a bare read of the slot — Ruby's
`primary_key.rb:18-56` and `composite_primary_key.rb:18-25`. `isPrimaryKeySettled`
is deleted.

That needs the class to answer `primary_key` truthfully at construction time,
i.e. it is gated on the schema-reflection warm-up work: a record must not be
constructible before its model's columns hash has landed, or `getPrimaryKey`
must not fall back to `"id"` on a cold cache. File the enabling half against the
schema-cache-warming RFC if this is picked up before it lands.

## Acceptance criteria

- [ ] `initInternals` seats `_primaryKey` unconditionally, as core.rb:846 does.
- [ ] Both `primaryKeyOf` helpers read only the slot; no class fallback.
- [ ] `isPrimaryKeySettled` is deleted from primary-key.ts.
- [ ] `primary-key-slot.trails.test.ts`'s cold-construction case is retired or
      rewritten against the new invariant; `primary-keys.test.ts`,
      `base.test.ts`, `attribute-methods.test.ts` green on SQLite, PG and
      MySQL/MariaDB.
