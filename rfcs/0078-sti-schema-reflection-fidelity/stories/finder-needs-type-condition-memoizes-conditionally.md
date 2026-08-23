---
title: "finder_needs_type_condition? memoizes conditionally where Rails memoizes unconditionally"
status: ready
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
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

Rails' `finder_needs_type_condition?` (activerecord/lib/active_record/inheritance.rb:92-95) is a
plain unconditional memo:

```ruby
def finder_needs_type_condition? # :nodoc:
  :true == (@finder_needs_type_condition ||= descends_from_active_record? ? :false : :true)
end
```

trails' `isFinderNeedsTypeCondition` (packages/activerecord/src/inheritance.ts, in the
"Methods missing from parity:api" tail) instead memoizes _conditionally_: it caches `true`
eagerly, but caches `false` only when `descendsFromActiveRecordByHierarchy(modelClass)` or
`inheritanceColumn === null`, with the comment "a non-root model that descends only because its
`type` column hasn't reflected yet must recompute once schema warms."

That rationale is obsolete as of trails#6926. `isDescendsFromActiveRecord` now calls
`columnsHash()` unconditionally (Rails' `columns_hash.include?(inheritance_column)`,
inheritance.rb:82-88), which loads the schema rather than answering from a cold-cache fallback —
so there is no longer a transient "column hasn't reflected yet" answer to avoid caching.

The private helper `descendsFromActiveRecordByHierarchy` exists only to serve this conditional
memo (it is the `self == Base` / abstract-superclass-recursion prefix of
`descends_from_active_record?`, split out). Rails has no such helper, so it is extra surface.

## Converged shape

```ts
export function isFinderNeedsTypeCondition(modelClass: typeof Base): boolean {
  if (!Object.prototype.hasOwnProperty.call(modelClass, "_finderNeedsTypeCondition")) {
    (modelClass as any)._finderNeedsTypeCondition = !modelClass.isDescendsFromActiveRecord();
  }
  return (modelClass as any)._finderNeedsTypeCondition === true;
}
```

and `isDescendsFromActiveRecord` inlines Rails' three arms directly (`self == Base` → false,
`superclass.abstract_class?` → recurse, else `superclass == Base || !columns_hash.include?(...)`),
deleting `descendsFromActiveRecordByHierarchy`.

## Acceptance criteria

- [ ] `isFinderNeedsTypeCondition` memoizes unconditionally, mirroring inheritance.rb:92-95.
- [ ] `descendsFromActiveRecordByHierarchy` is deleted and `isDescendsFromActiveRecord` carries
      Rails' own three-arm control flow.
- [ ] `parity:api:extra --package activerecord` does not regress.
- [ ] Existing STI suites stay green, in particular the cold cases in
      `inheritance.trails.test.ts` and `sti-attribute-routing.trails.test.ts`.
