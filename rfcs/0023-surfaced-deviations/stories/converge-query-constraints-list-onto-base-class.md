---
title: "query_constraints_list walks the prototype chain instead of base_class?, and its composite-PK compare is always true"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `query_constraints_list` walks the prototype chain instead of asking `base_class?` — and the composite-PK compare is wrong

## Context

Surfaced converging the `persistence.json` call-set rows in PR #6735 (RFC 0106
wave 4c); left baselined as `query_constraints_list -> base_class?`.

Rails (`vendor/rails/activerecord/lib/active_record/persistence.rb:222-229`):

```ruby
def query_constraints_list # :nodoc:
  @query_constraints_list ||= if base_class? || primary_key != base_class.primary_key
    primary_key if primary_key.is_a?(Array)
  else
    base_class.query_constraints_list
  end
end
```

trails (`packages/activerecord/src/persistence.ts:199-216`) re-derives the same
question by walking `Object.getPrototypeOf` and testing `parent.name === "Base"`
/ `_isBaseClass`, then compares `this.primaryKey !== parent.primaryKey`.

Two problems:

1. **`base_class?` is already ported** and is the reader Rails asks. The
   prototype walk duplicates the hierarchy logic `baseClass` itself owns, so the
   two can disagree for an abstract class or a namespaced STI root.

2. **The primary-key comparison is wrong for a composite PK.** Ruby's `!=` on
   two Arrays is VALUE equality, so `["shop_id", "id"] != ["shop_id", "id"]` is
   false. JS `!==` on two arrays is reference inequality and is ALWAYS true, so
   a composite-PK subclass whose PK matches its base class takes the wrong
   branch and answers its own `primaryKey` instead of delegating to
   `base_class.query_constraints_list`.

## Converged shape

```ts
export function queryConstraintsList(this: PersistenceHost): string[] | null {
  this._queryConstraintsList ??= (() => {
    const base = baseClass.call(this);
    if (isBaseClass.call(this) || !primaryKeysEqual(this.primaryKey, base.primaryKey)) {
      const pk = this.primaryKey;
      return Array.isArray(pk) ? pk : null;
    }
    return queryConstraintsList.call(base);
  })();
  return this._queryConstraintsList;
}
```

where the equality is element-wise for the Array arm — Ruby's `!=`, not `!==`.

Related but distinct: `composite-query-constraints-list-drops-memo`
(0023-surfaced-deviations) covers the `@query_constraints_list ||=` memo; fold
the two if they land together.

## Acceptance criteria

- [ ] `queryConstraintsList` calls `isBaseClass` / `baseClass` rather than
      walking the prototype chain itself.
- [ ] The primary-key comparison is value equality for the composite-PK arm.
- [ ] A regression test covering a composite-PK subclass whose PK equals its
      base class's — it must delegate, and must FAIL on baseline.
- [ ] The `persistence.json` `query_constraints_list -> base_class?` row is
      deleted, then `pnpm parity:api:calls:tighten activerecord/persistence.json`.
- [ ] Composite-PK suites green on all three lanes.
