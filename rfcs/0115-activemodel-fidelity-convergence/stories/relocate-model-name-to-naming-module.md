---
title: "relocate-model-name-to-naming-module"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7020
claim: "2026-08-25T00:30:08Z"
assignee: "relocate-model-name-to-naming-module"
blocked-by: null
closed-reason: null
---

## Context

`Model.modelName` is still a `model.ts` class body
(`packages/activemodel/src/model.ts`), even though Rails defines it in
`ActiveModel::Naming#model_name`
(`vendor/rails/activemodel/lib/active_model/naming.rb:270-277`) and the instance
delegate in `Naming.extended` (`naming.rb:253-256` —
`base.delegate :model_name, to: :class`). It is the last member of
`fan-out-model-serialization-conversion-access-naming-surface`'s table that did
not move, and it did not move for a structural reason.

`naming.rb`'s module carries BOTH kinds of method:

- `def model_name` (`:270`) — a module _instance_ method, which `extend
ActiveModel::Naming` (api.rb:66) copies onto the host's singleton.
- `def self.plural` / `self.singular` / `self.uncountable?` /
  `self.singular_route_key` / `self.route_key` / `self.param_key` /
  `self.model_name_from_record_or_class` (`:283-348`) — module _functions_,
  which `extend` does NOT copy.

trails' `extend()` (`packages/activesupport/src/include.ts:351-384`) copies
every own static of the module object. `naming.ts` holds the `def self.*` half
as a `namespace Naming`, so a class-module `Naming` merged with it puts both
halves on one object and `extend(Model, Naming)` copies all of them: the first
pass of PR #7010 did exactly this and put seven invented statics on `Model`
(`'plural' in Model === true`), which is why it was reverted rather than
shipped.

Ruby tells the two halves apart by singleton-vs-instance method. TS has one
object, and `extend()` has no signal to split on.

## Acceptance criteria

- A settled representation for a Ruby module that carries both `def x` and
  `def self.x`, applied to `naming.ts` — e.g. `extend()` copying a class
  module's PROTOTYPE members (which is what Ruby's `extend` actually copies)
  rather than its statics, or a documented carrier that keeps the two halves on
  separate objects. Whichever it is, `Naming.plural(record)` still answers, and
  `'plural' in Model` is `false`.
- `Model.modelName` and the instance `modelName` delegate move to `naming.ts`,
  the delegate installed by `Naming`'s `extended` hook per `naming.rb:253-256`.
- `model.ts` keeps no `model_name` body; `pnpm parity:api:extra --package
activemodel` does not regress.
- `pnpm vitest run packages/activemodel/src packages/activerecord/src/base.test.ts`
