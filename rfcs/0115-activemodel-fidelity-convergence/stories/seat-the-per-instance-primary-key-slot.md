---
title: "seat-the-per-instance-primary-key-slot"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6934
claim: "2026-08-23T18:32:16Z"
assignee: "seat-the-per-instance-primary-key-slot"
blocked-by: null
closed-reason: null
---

# Seat the per-instance `@primary_key` slot from `Core#init_internals`

## Context

Split out of RFC 0115 `port-core-init-internals-missing-assignments` (trails PR
for that story ported the other three of `init_internals`' four missing
assignments). Rails
(`vendor/rails/activerecord/lib/active_record/core.rb:846`):

```ruby
@primary_key = klass.primary_key
```

and every instance-side primary-key reader then goes through that ivar —
`AttributeMethods::PrimaryKey#id` / `#id=` / `#id?` / `#id_before_type_cast`
(`vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb:18-56`)
and `CompositePrimaryKey#id=`'s `@primary_key.zip(value)`
(`vendor/rails/activerecord/lib/active_record/attribute_methods/composite_primary_key.rb:18-25`).

trails has no such slot. `primaryKeyOf`
(`packages/activerecord/src/attribute-methods/primary-key.ts:160-162`) reads
`record.constructor.primaryKey` at every call, and its own JSDoc already names
this as the deviation.

The blocker found while porting the sibling story: trails' schema reflection is
asynchronous. `getPrimaryKeyAttr`
(`attribute-methods/primary-key.ts:227-231`) falls through to `getPrimaryKey`,
which consults the schema cache; a record constructed before its model's
columns hash has arrived would latch whatever placeholder that returns and keep
it for the record's lifetime, where the current class-read picks up the real key
as soon as it lands. Seating the slot therefore needs a decision about what a
pre-schema construction stores (and whether a re-read on columns-hash arrival is
required), not just the one-line assignment.

## Acceptance criteria

- [ ] `Core#initInternals` assigns `_primaryKey` from `klass.primaryKey`, in
      core.rb:846's position.
- [ ] `primaryKeyOf` reads the per-instance slot, as Ruby's readers read
      `@primary_key`.
- [ ] A record constructed before its schema loads still resolves the real
      primary key (covered by a test that fails on the naive seat).
- [ ] `primary-keys.test.ts`, `base.test.ts` and `attribute-methods.test.ts`
      stay green on SQLite, PostgreSQL and MySQL/MariaDB.
