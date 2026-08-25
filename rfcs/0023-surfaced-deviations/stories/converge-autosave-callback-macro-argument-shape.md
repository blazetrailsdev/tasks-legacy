---
title: "Autosave callback macros take a closure where Rails passes the save-method name"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6420 (RFC 0096 wave 2). Five `naming` call-argument rows in
`packages/activerecord/src/autosave-association.ts` cannot be closed by
renaming, because the trails callback macros take a different argument shape
than Rails' class-level macros.

Rails
(`vendor/rails/activerecord/lib/active_record/autosave_association.rb:189-213`):

```ruby
def add_autosave_association_callbacks(reflection)
  save_method = :"autosave_associated_records_for_#{reflection.name}"

  if reflection.collection?
    around_save :around_save_collection_association
    define_non_cyclic_method(save_method) { save_collection_association(reflection) }
    after_create save_method
    after_update save_method
  elsif reflection.has_one?
    define_non_cyclic_method(save_method) { save_has_one_association(reflection) }
    after_create save_method
    after_update save_method
  else
    define_non_cyclic_method(save_method) { throw(:abort) if save_belongs_to_association(reflection) == false }
    before_save save_method
  end
```

Rails defines the method under `save_method` and then hands the **method
name** to the macro: `after_create save_method`. The macro resolves it against
the instance at fire time.

trails defines the method the same way (`defineNonCyclicMethod.call(model,
saveMethod, ...)`) but then registers a **closure over the model** instead:

```ts
afterCreate(model, async (record: any) => {
  await record[saveMethod]();
});
```

so the first argument is `model`, not `saveMethod`. The extractor reports
`ruby ["ref:saveMethod"]` vs `ts ["ref:model"]` for each of the five
`after_create` / `after_update` / `before_save` registrations.

This is the free-function form of a Ruby class macro — an instance of the
`this`-typed-function mixin idiom in CLAUDE.md — but the _argument shape_ is a
trails choice, not a language constraint: the callback registry could accept a
method name and dispatch it on the record at fire time, exactly as Rails does,
which would also make the registered callback introspectable by name.

The converged shape is `afterCreate(model, saveMethod)` — the macro taking the
method name — with dispatch moved into the callback runner.

Note the closure is also why these five registrations are invisible to
anything that inspects registered callbacks by name.

## Acceptance criteria

- [ ] The trails callback macros accept a method-name string and dispatch it
      on the record at fire time (`record[saveMethod]()`), matching
      `after_create save_method` (autosave_association.rb:197-198,209-210,213).
- [ ] `addAutosaveAssociationCallbacks` passes `saveMethod`, not a closure, at
      all five sites; the five `naming` rows drop out of
      `pnpm parity:api:calls:args:report`.
- [ ] If the registry cannot take a name, the deviation is justified at the
      call site with a Rails cite — but per CLAUDE.md that is the fallback,
      not the goal.
- [ ] `pnpm parity:api:calls` / `:args` green, no baseline row added;
      autosave-association tests pass on all three adapters.
