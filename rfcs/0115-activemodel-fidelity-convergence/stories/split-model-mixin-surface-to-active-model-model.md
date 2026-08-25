---
title: "split-model-mixin-surface-to-active-model-model"
status: ready
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 7
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::Model` is two `include`s
(`vendor/rails/activemodel/lib/active_model/model.rb:42-45`):

```ruby
module Model
  extend ActiveSupport::Concern
  include ActiveModel::API
  include ActiveModel::Access
end
```

and `ActiveModel::API` contributes `initialize` and `persisted?` (api.rb:82,
:94). trails' `Model` also mixes in Attributes, AttributeRegistration,
AttributeMethods, Dirty, Callbacks, Serialization and Serializers::JSON — the
surface `ActiveRecord::Base` composes in Rails, hoisted onto `ActiveModel::Model`.

`fan-out-model-serialization-conversion-access-naming-surface` (PR #7010) moved
every relocatable member body out of `model.ts`; what is left is the constructor,
`dup`, `isPersisted`, `isAttributeMethod`, 156 lines of type-only `declare` /
`interface Model` members, and the `include()` / `extend()` / `prepend()` calls.
That story's original `0 moved` and `<= 200 code lines` criteria were re-scoped
here, because neither moves by relocating bodies:

- `extra-surface` scores a NAME against the allow-set `model.rb` + its Ruby
  include chain builds. `model.ts`'s 61 `moved` names are exactly the mixins
  above. They are counted whether spelled as a `declare static`, an
  `interface Model extends Dirty`, or only as an `include(Model, Dirty)` call —
  `include()` of a class module pushes the module onto the host's `extends`
  (`scripts/api-compare/extract-ts-api.ts:1191-1227`), and interface-`extends`
  members carry no `declaredIn`, so they are counted
  (`extract-ts-api.ts:14-27`).
- The type-only lines exist because TypeScript cannot type a runtime
  `include(Model, X)` without a declaration. Deleting them does not shrink the
  mixin set; it makes `Model.validates` untyped for every caller.

So both numbers are one question: should `ActiveModel::Model` carry the
ActiveRecord-shaped mixin set at all, and if not, where does each mixin move —
`ActiveRecord::Base`, or a documented trails host that `Base` extends?

## Acceptance criteria

- A decision on `Model`'s mixin set, applied: each mixin trails' `Model`
  includes that `ActiveModel::Model` does not either moves to the Rails host
  that includes it, or is ratified once, centrally, with the reason.
- `pnpm parity:api:extra --package activemodel` reports `model.ts` at
  0 novel / 0 moved, and `model.ts` is <= 200 code lines.
- The activemodel and activerecord suites stay green; parity deltas
  non-negative; `pnpm parity:api:calls` / `:args` clean.

## Definition of done

`model.ts` reads as the port of `model.rb` + `api.rb` + `access.rb`, and nothing
else.
