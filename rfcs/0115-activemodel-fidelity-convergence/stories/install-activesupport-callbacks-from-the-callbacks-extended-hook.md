---
title: "Install ActiveSupport::Callbacks from an ActiveModel::Callbacks extended hook"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-24T14:19:54Z"
assignee: "website-vitest-missing-activesupport-subpath-alias"
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::Callbacks` is a module whose `self.extended` hook installs
`ActiveSupport::Callbacks` on whatever extends it:

```ruby
# activemodel/lib/active_model/callbacks.rb:66-70
module Callbacks
  def self.extended(base) # :nodoc:
    base.class_eval do
      include ActiveSupport::Callbacks
    end
  end
```

`ActiveModel::Validations`' `included do` block reaches it with
`extend ActiveModel::Callbacks` (`activemodel/lib/active_model/validations.rb:42`),
so a class that includes Validations gets the `ActiveSupport::Callbacks` include
transitively, from the module — never from its own body.

trails splits that in two. Since PR #6979, `Validations.[included]`
(`packages/activemodel/src/validations.ts`) does
`extend(base, { defineModelCallbacks })` — the one member
`packages/activemodel/src/callbacks.ts` exports — while the
`ActiveSupport::Callbacks` half is still hand-wired in `model.ts`'s own body:

```ts
// packages/activemodel/src/model.ts (bottom)
extend(Model, ASCallbacks.ClassMethods);
include(Model, ASCallbacks.InstanceMethods);
```

So `model.ts` carries wiring `callbacks.rb:66-70` owns, and any other class that
extends trails' Callbacks has to repeat it.

## Converged shape

`callbacks.ts` exports a `Callbacks` class module carrying the module's own
members as statics — `defineModelCallbacks` (`callbacks.rb:72`) and the three
private definers `_define_before_model_callback` / `_define_around_model_callback`
/ `_define_after_model_callback` (`callbacks.rb:144-158`) — plus the `extended`
hook, keyed by the `extended` symbol from `@blazetrails/activesupport`
(`packages/activesupport/src/include.ts:122`), which `extend()` fires:

```ts
export class Callbacks {
  static [extended](base: AnyClass): void {
    // callbacks.rb:67-69 — base.class_eval { include ActiveSupport::Callbacks }
    extend(base, ASCallbacks.ClassMethods);
    include(base, ASCallbacks.InstanceMethods);
  }
  static defineModelCallbacks = defineModelCallbacks;
  // …the three definers
}
```

`Validations.[included]` then issues plain `extend(base, Callbacks)`, and the two
`ASCallbacks` lines leave `model.ts`.

The blocker to doing this inside #6979 was a name collision: `callbacks.ts`
already exports `type Callbacks = CallbacksClassMethods`. Reshaping that exported
type is this story's first step — check its importers across activerecord before
renaming or removing it.

## Acceptance criteria

- `packages/activemodel/src/callbacks.ts` exports a `Callbacks` module whose
  `[extended]` hook installs `ActiveSupport::Callbacks`, mirroring
  `callbacks.rb:66-70`.
- `Validations.[included]` issues `extend(base, Callbacks)`; `model.ts` no longer
  wires `ASCallbacks.ClassMethods` / `ASCallbacks.InstanceMethods` itself.
- The existing `Callbacks` type export is resolved without breaking activerecord.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Verification

```bash
pnpm vitest run packages/activemodel/src/callbacks.test.ts packages/activemodel/src/validations.test.ts
pnpm parity:api --package activemodel && pnpm lint --fix
```
