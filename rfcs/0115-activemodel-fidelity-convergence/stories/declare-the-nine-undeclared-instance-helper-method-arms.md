---
title: "Declare the nine undeclared instance-side HelperMethods arms on Model"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: 6989
claim: "2026-08-24T14:48:31Z"
assignee: "converge-numericality-bigint-exponent-skip"
blocked-by: null
closed-reason: null
---

## Context

Rails' `ActiveModel::Validations` does **both** `extend HelperMethods` and
`include HelperMethods` (`activemodel/lib/active_model/validations.rb:45-46`), so
all eleven `validates_*_of` macros answer on the class _and_ on an instance — the
instance arm is what a `validate do … end` body calls.

Since PR #6979 all eleven genuinely exist on `Model.prototype`: `HelperMethods`
is one module object (`packages/activemodel/src/validations/helper-methods.ts`)
and `Validations.[included]` installs it with both `extend(base, HelperMethods)`
and `include(base, HelperMethods)`. The runtime is correct.

The **types** are not. `packages/activemodel/src/model.ts`'s `interface Model`
declares only two of the instance arms:

```ts
  validatesPresenceOf(...attrNames: AttrNameArg[]): Promise<void>;
  validatesLengthOf(...attrNames: AttrNameArg[]): Promise<void>;
```

which is what `model.ts` happened to carry before the fan-out. The other nine —
`validatesAbsenceOf`, `validatesSizeOf`, `validatesNumericalityOf`,
`validatesInclusionOf`, `validatesExclusionOf`, `validatesFormatOf`,
`validatesAcceptanceOf`, `validatesConfirmationOf`, `validatesComparisonOf` —
fall through `Model`'s `[key: string]: unknown` index signature, so calling one
from a `validate` block needs a cast even though it works at runtime.

## Converged shape

Declare all eleven instance arms in `interface Model`, each returning
`Promise<void>`: the instance `validates_with`
(`activemodel/lib/active_model/validations/with.rb:144-151`, ported in
`validations/with.ts`) is async under RFC 0063, so the helper's returned value is
a promise in the instance role. The class arms are already declared as
`Extended<typeof HelperMethods>[…]` and stay `void`; see the `HelperMethodsHost`
JSDoc in `validations/helper-methods.ts` for why one body serves both roles.

## Acceptance criteria

- All eleven `validates<Kind>Of` instance arms are declared on `Model` and
  callable from a `validate` block without a cast.
- No runtime change — the module install already provides them.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
