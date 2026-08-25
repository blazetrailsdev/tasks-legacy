---
title: "beforeValidation/afterValidation drop Rails' method-name (Symbol) arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while porting `validations_test.rb`'s models in PR #6638.

Rails' validation callbacks take `*args`, so a method name is a first-class
argument: `before_validation(*args, &block)` /
`after_validation(*args, &block)` both `args.extract_options!` and forward to
`set_callback(:validation, :before|:after, *args, options, &block)`
(`vendor/rails/activemodel/lib/active_model/validations/callbacks.rb:55-61`
and `:71-78`). `activemodel/test/models/topic.rb:19` uses exactly that arm:

```ruby
after_validation :perform_after_validation
```

trails' `Model.beforeValidation` / `Model.afterValidation`
(`packages/activemodel/src/model.ts:991` and `:1006`) accept only
`((record) => …) | CallbackObject` — there is no method-name arm. Porting
`topic.rb` therefore had to wrap the method in an arrow:

```ts
this.afterValidation((t: Topic) => t.performAfterValidation());
```

This is an inconsistency, not a language shortcoming: the sibling
`Model.validate` already takes `methodOrFn: string | ((record) => unknown)`
(`model.ts:673`) and resolves the string against the record at call time
(`typeof r[methodOrFn] === "function"`). The same arm belongs on the two
validation callbacks, which is why this is a convergence story and not a
ratification of the wrapper.

## Converged shape

Widen both signatures to accept the Rails arm and resolve it the way
`Model.validate` already does:

```ts
static beforeValidation<T extends typeof Model>(
  this: T,
  methodOrFn: string | ((record: InstanceType<T>) => void | boolean | Promise<void | boolean>) | CallbackObject,
  conditions?: ValidationCallbackConditions<InstanceType<T>>,
): void
```

with the string arm dispatching `record[methodOrFn]()` at run time, so a method
defined after the `static {}` block still resolves (Ruby resolves the Symbol
when the callback fires, not when it is registered).

Check `beforeSave` and the other lifecycle callbacks in the same region of
`model.ts` for the identical gap while in there — Rails' `define_model_callbacks`
family is uniformly `*args`.

## Acceptance criteria

- `Model.beforeValidation` / `Model.afterValidation` accept a method-name string
  and dispatch it against the record when the callback runs.
- `packages/activemodel/src/validations.test.ts`'s `Topic` mirrors
  `activemodel/test/models/topic.rb:19` literally
  (`this.afterValidation("performAfterValidation")`), dropping the arrow wrapper.
- `pnpm parity:api:calls` / `parity:api:calls:args` stay clean; no baseline row
  is added for this.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
