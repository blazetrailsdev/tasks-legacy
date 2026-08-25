---
title: "_insert_record's prefetch guard adds a composite-PK exclusion Rails does not have"
status: draft
updated: 2026-08-23
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

`_insert_record`'s prefetch arm
(`vendor/rails/activerecord/lib/active_record/persistence.rb:242-247`):

```ruby
if prefetch_primary_key? && primary_key
  values[primary_key] ||= begin
    primary_key_value = next_sequence_value
    _default_attributes[primary_key].with_cast_value(primary_key_value)
  end
end
```

trails (`packages/activerecord/src/persistence.ts:249-259`, after PR #6900
converged the `with_cast_value` half):

```ts
if (ctor.isPrefetchPrimaryKey?.() && primaryKey && !Array.isArray(primaryKey)) {
  if (values[primaryKey] == null) {
    primaryKeyValue = ctor.nextSequenceValue?.();
    values[primaryKey] = ctor
      ._defaultAttributes()
      .getAttribute(primaryKey)
      .withCastValue(primaryKeyValue);
  }
}
```

Two deviations remain in the guard, both left in place by #6900 because they
were out of that story's scope:

1. **`!Array.isArray(primaryKey)`** has no Rails counterpart. Rails' guard is
   `prefetch_primary_key? && primary_key`; a composite primary key is an Array
   there too and Rails does not exclude it. Whether the arm is right for a
   composite PK is a real question — `_default_attributes[primary_key]` with an
   Array key would miss — but the answer belongs in the ported body, not in an
   invented guard that silently skips.

2. **`values[primaryKey] == null`** is narrower than Ruby's `||=`, which also
   fires on a stored `false`. This is the `fetch`-vs-`??` class from
   CLAUDE.md's Ruby-idiom list. A `false` primary-key value is nonsense in
   practice, so this is the lower-value half.

Unobservable today: `prefetchPrimaryKey()` is false for every adapter trails
ships (SQLite, PostgreSQL, MySQL); Rails' arm exists for Oracle and similar.
The `PostWithPrefetchedPk` test model and the trails-only cast guard added in
PR #6900 added are the only things that drive it.

## Acceptance criteria

- [ ] The guard is `isPrefetchPrimaryKey() && primaryKey`, matching
      `persistence.rb:242` — the `!Array.isArray` arm is gone.
- [ ] Composite-PK behaviour under a prefetching adapter is decided by porting
      what Rails does, with the decision cited at the call site; if Rails would
      raise, trails raises the same error from the same place.
- [ ] The `||=` arm matches Ruby truthiness (`!= null && !== false`) or carries
      a call-site note explaining why `null` alone is sufficient here.
- [ ] The existing prefetch guards stay green:
      `create prefetched pk` (persistence.test.ts) and
      `prefetched pk is re-cast through the primary key's default attribute`
      (persistence.trails.test.ts).
