---
title: "Hoist the owner-based module super and UnboundMethod#bind_call spellings into activesupport's Module"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing PR #6778 (`converge-accepts-multiparameter-time-to-a-real-mixin`).

Ruby resolves `super` from the OWNER of the running method within the
receiver's ancestry, so a method defined in a mixed-in module reaches the
includer's superclass, and a class-body override of a mixin method reaches the
mixin's own version. Two shapes in that PR had to spell that by hand, because
TypeScript's `super` is typed and bound against the DECLARED base class only
and knows nothing about the carrier `include()` splices in:

1. `packages/activemodel/src/type/helpers/accepts-multiparameter-time.ts` —
   a file-local `superOf(receiver, name, self)` that walks the prototype chain
   for the LAST link owning `name` and answers from that link's prototype,
   used by the mixin's `cast` / `assertValidValue`
   (`vendor/rails/activemodel/lib/active_model/type/helpers/accepts_multiparameter_time.rb:17-31`,
   whose `else super(value)` arms are the ones being spelled).
2. `packages/activemodel/src/type/date.ts`, `date-time.ts`, `time.ts` — each
   class-body `valueFromMultiparameterAssignment` override reaches the mixin's
   version through
   `acceptsMultiparameterTime.instanceMethod("valueFromMultiparameterAssignment")!.value.call(this, ...)`,
   the trails spelling of Ruby's
   `instance_method(:value_from_multiparameter_assignment).bind_call(self, ...)`
   — the `super` in `date.rb:75-78` and `date_time.rb:77-83`.

Both are correct and language-forced, but they are hand-rolled at their call
sites and the second is repeated three times with a six-line cast each. Ruby
spells them with two core APIs trails' `Module` already half-mirrors
(`packages/activesupport/src/include.ts`): `Module#instance_method` is there
(returning a `PropertyDescriptor`, the trails stand-in for `UnboundMethod`),
`UnboundMethod#bind_call` is not, and there is no owner-based `super` helper.

Ruby's rule that a module is never held twice in an ancestry is what makes the
LAST-owner walk correct; trails' `include()` will happily splice the same
carried function twice (a subclass re-including a mixin its parent already
has), which is the crash the first `superOf` had.

## Converged shape

The two Ruby primitives live once in `activesupport/src/include.ts` next to
`Module#instance_method`, at their Ruby names, and the ActiveModel call sites
use them instead of a local walker and an inline descriptor cast:

- `bindCall` for `UnboundMethod#bind_call` — takes the descriptor
  `instanceMethod` returns, a receiver, and the args.
- an owner-based `super` lookup for a module method (Ruby's implicit `super`),
  documented with the never-twice-in-an-ancestry invariant it relies on.

Consider whether `include()` should instead refuse to re-splice a carrier
already present in the target's chain, which is what Ruby's `include` does and
would make the owner walk trivial.

## Acceptance criteria

- [ ] `superOf` and the three inline
      `instanceMethod(...)!.value as (...) => ...` casts are gone from
      `activemodel/src/type/`, replaced by the shared `activesupport` spellings.
- [ ] A test covers the double-inclusion shape (subclass re-including a mixin
      its parent already has) — it crashed before PR #6778's fix.
- [ ] `pnpm parity:api:extra` does not grow for either package;
      `pnpm parity:api:calls` / `:args` clean.
