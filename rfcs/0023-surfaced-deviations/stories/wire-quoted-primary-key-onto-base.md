---
title: "quoted_primary_key is not wired onto Base and diverges from adapter_class.quote_column_name"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6842 (`wave-4c-ar-core-residue-attributes-remainder-part-5`)
while working `attribute-methods/primary-key.ts`.

Rails,
`vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb:88-90`:

```ruby
# Returns a quoted version of the primary key name.
def quoted_primary_key
  adapter_class.quote_column_name(primary_key)
end
```

trails has `quotedPrimaryKey` in
`packages/activerecord/src/attribute-methods/primary-key.ts`, but it is never
wired onto `Base` — there is no `static quotedPrimaryKey` assignment in
`base.ts`, so `Model.quotedPrimaryKey` does not exist. The gap is already
recorded in a test comment in
`packages/activerecord/src/primary-keys.test.ts` under
`it("quoted primary key after set primary key")`, which tests the
`primary_key=` setter instead of the method Rails' test exercises
(`vendor/rails/activerecord/test/cases/primary_keys_test.rb`).

The body also diverges from the Rails one-liner in three ways:

- it reads `this.connection` (a lease) rather than `adapter_class`;
- it carries a bespoke `fallback` quoter (`"${k.replace(/"/g, '""')}"`) for
  the no-connection case, which Rails has no counterpart for and which hard-codes
  the SQLite/PG identifier quote — wrong on MySQL/MariaDB, where it is a backtick;
- it joins a composite key with `", "`, which Rails' single
  `quote_column_name(primary_key)` call does not do.

## Acceptance criteria

- [ ] `quotedPrimaryKey` is wired onto `Base` so `Model.quotedPrimaryKey`
      resolves, matching primary_key.rb:88-90.
- [ ] The body is `adapterClass.quoteColumnName(primaryKey)`; the bespoke
      `fallback` quoter is gone, or its retention is justified at the call
      site as a language shortcoming.
- [ ] `PrimaryKeysTest > quoted primary key after set primary key` converges to
      the Rails test body and drops its "not wired to the Base class surface"
      note. Do NOT rename the test.
- [ ] `pnpm parity:api` delta non-negative; no new `parity:api:extra` surface.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green — the fallback quoter's
      identifier character is adapter-sensitive.
