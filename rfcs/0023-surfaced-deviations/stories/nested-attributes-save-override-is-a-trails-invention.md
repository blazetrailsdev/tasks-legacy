---
title: "Retire the nested-attributes prototype.save override in favour of the autosave callback path"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #5248. `acceptsNestedAttributesFor`
(`packages/activerecord/src/nested-attributes.ts:171-185`) monkey-patches
`modelClass.prototype.save` to flush pending nested attributes _after_ the
parent is persisted:

```ts
const originalSave = modelClass.prototype.save;
if (!(modelClass as any)._nestedSaveWrapped) {
  (modelClass as any)._nestedSaveWrapped = true;
  modelClass.prototype.save = async function (this, options) {
    const result = await originalSave.call(this, options);
    if (!result) return false;
    await processNestedAttributes(this);
    return true;
  };
}
```

Rails has no such override. `accepts_nested_attributes_for` only flips
`reflection.autosave = true` and defines the writer
(`vendor/rails/activerecord/lib/active_record/nested_attributes.rb:355-370`);
the child records are persisted by the ordinary autosave save callbacks inside
the parent's save transaction.

Concrete consequences of the wrapper being a separate post-save step:

1. It was argument-opaque until PR #5248 — `save({ validate: false })` was
   silently dropped for _every_ model with nested attributes. That class of bug
   recurs for any option added to `save` later, because the signature is
   duplicated here by hand.
2. The flush runs **after** the parent's save transaction has committed, so a
   failing nested write is not rolled back with the parent — unlike Rails,
   where it is part of the same transaction.
3. It returns `true` unconditionally after a successful flush, discarding
   whatever `processNestedAttributes` might have wanted to report.
4. `originalSave` is captured at declaration time and the `_nestedSaveWrapped`
   flag is set on the class, so the interaction with subclassing and with a
   second `acceptsNestedAttributesFor` call on the same class is order-dependent.

## Acceptance criteria

- Move the nested-attributes flush onto the autosave callback path (inside the
  parent's save transaction), matching `nested_attributes.rb`, and delete the
  `prototype.save` override and its `_nestedSaveWrapped` flag.
- A regression test showing a failing nested write rolls back with the parent.
- Keep `nested-attributes.trails.test.ts`'s
  "forwards save options through the nested-attributes save wrapper" green (or
  replace it with the equivalent assertion once there is no wrapper).
