---
title: "unscope-create-with-is-not-a-valid-unscoping-value"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:768-771`:

```ruby
VALID_UNSCOPING_VALUES = Set.new([:where, :select, :group, :order, :lock,
                                 :limit, :offset, :joins, :left_outer_joins, :annotate,
                                 :includes, :eager_load, :preload, :from, :readonly,
                                 :having, :optimizer_hints, :with])
```

18 keys, and `:create_with` is NOT one of them — Rails raises
`ArgumentError, "Called unscope() with invalid unscoping argument ':create_with'."`
for `unscope(:create_with)`.

`packages/activerecord/src/relation/query-methods.ts:629-660` (`UnscopeType` and
`VALID_UNSCOPING_VALUES`) carries a 19th entry, `createWith`, so
`unscope("createWith")` silently resets `_createWithAttrs` where Rails raises.
The JSDoc above the type even lists `create_with` as if it were in the Rails
set. Surfaced during review of PR #6571, which touched neighbouring lines but
not this one (`except`/`only` do not validate keys, so they are unaffected).

## Converged shape

Drop `createWith` from `UnscopeType` and `VALID_UNSCOPING_VALUES`, and fix the
JSDoc key list to match `query_methods.rb:768-771` exactly.

**`EXCEPT_ONLY_KEYS` must keep `createWith`.** It is built as
`[...VALID_UNSCOPING_VALUES, ...]` and `:create_with` IS a
`Relation::VALUE_METHODS` key (relation.rb:59-60), so the removal has to add
`createWith` back explicitly on the `EXCEPT_ONLY_KEYS` side. Otherwise
`Relation#values()` and `EXCEPT_ONLY_KEYS` fall out of sync, which reds the
drift guard `values() covers exactly the Relation::VALUE_METHODS key set`
(`relation/value-accessor-semantics.test.ts`) and would make `except`/`only`
reset `create_with` unconditionally.

`resetValueForScope`'s `createWith` arm stays — `setValues` reaches it for the
value key even once `unscope!` can no longer.

## Acceptance criteria

- [ ] `VALID_UNSCOPING_VALUES` is Rails' 18 keys, camelCased; its JSDoc list
      matches.
- [ ] `unscope("createWith")` raises `ArgumentError` with Rails' message.
- [ ] `EXCEPT_ONLY_KEYS` still carries `createWith`; the `values()` drift guard
      stays green.
- [ ] A test covering the raise, named from the Rails test if one exists.
