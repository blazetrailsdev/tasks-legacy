---
title: "Route validates_each through validates_with and delete _registerValidator"
status: draft
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails `validates_each` has one body
(`activemodel/lib/active_model/validations.rb:190-192`):

```ruby
def validates_each(*attr_names, &block)
  validates_with BlockValidator, _merge_attributes(attr_names), &block
end
```

Registration and callback wiring both come free from `validates_with`
(`validations/with.rb:88-105`). trails' `Model.validatesEach`
(`packages/activemodel/src/model.ts`, `static validatesEach`) instead builds the
`BlockValidator` itself, calls the trails-invented private
`Model._registerValidator`, and then calls `this.validate(...)` — a second copy
of the `_validators[...] << validator` half that PR #6976 inlined into
`validations/with.ts` per Rails.

`_registerValidator` survives only because of this one caller. PR #6976 narrowed
it to the single tier that caller needs and cited `with.rb:95-101` on it, but the
helper is invented surface with no Rails counterpart and the duplication is real.

The blocker is the block: `validates_with` must accept Ruby's `&block` and
forward it to `klass.new(options.dup, &block)` (`with.rb:92`) before
`validatesEach` can route through it. `BlockValidator`'s ctor already takes
`(options, block)` (`packages/activemodel/src/validator.ts`), so the work is
picking the trails spelling for the block slot on a variadic
`validatesWith(...args)` and threading it through.

## Converged shape

- `validatesWith` (class arm, `validations/with.ts`) takes the block and passes
  it to each `new klass(...)`, mirroring `with.rb:92`.
- `Model.validatesEach` becomes the one-line
  `this.validatesWith(BlockValidator, this._mergeAttributes(attrNames), block)`,
  matching `validations.rb:190-192`.
- `Model._registerValidator` is deleted — no caller remains and Rails has no
  such method.

## Acceptance criteria

- `_registerValidator` is gone from `model.ts`.
- `validatesEach`'s body is Rails' single `validates_with` send.
- `pnpm parity:api:extra --package activemodel` shows `model.ts` at no worse
  than its current novel/moved counts.
- `pnpm vitest run packages/activemodel/src` green; parity deltas non-negative;
  `pnpm parity:api:calls` / `:args` clean, no reseed.
