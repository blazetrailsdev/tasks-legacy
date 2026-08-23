---
title: "Fan out the validates_*_of macros from model.ts to their validator files"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-validation-runner-surface-to-validations
deps-rfc: []
est-loc: 220
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails defines each `validates_<kind>_of` macro **in the validator's own file**,
inside `module HelperMethods` — e.g. `validates_presence_of` in
`validations/presence.rb`, `validates_length_of` / `validates_size_of` in
`validations/length.rb`, `validates_numericality_of` in
`validations/numericality.rb`, and so on for absence, inclusion, exclusion,
format, acceptance, confirmation and comparison. `_merge_attributes` — the one
shared helper — is `validations/helper_methods.rb:7`.

trails puts all of them on `Model` (42 code lines across 15 members):

`validatesPresenceOf` `:958`, `validatesAbsenceOf` `:966`, `validatesLengthOf`
`:974`, `validatesNumericalityOf` `:982`, `validatesInclusionOf` `:990`,
`validatesExclusionOf` `:998`, `validatesFormatOf` `:1006`,
`validatesAcceptanceOf` `:1019`, `validatesConfirmationOf` `:1032`,
`validatesComparisonOf` `:1040`, `validatesSizeOf` `:1048`, plus the instance
arms `validatesWith` `:2049` (owned by another story),
`validatesPresenceOf` `:2079` and `validatesLengthOf` `:2087`, and
`_mergeAttributes` `:1672` (Rails: `helper_methods.rb:7`).

Each is a 3–6 line `this.validatesWith(XValidator, this._mergeAttributes(
attrNames))`, which is exactly Rails' body — so this story is a **pure
relocation**, the cheapest in Phase 1. Its value is that
`blazetrails/rails-file-structure-method-order` can then order each
`validations/*.ts` file against its own Rails file, which it cannot do while
the macros live somewhere else.

Note the instance-level arms at `:2079` and `:2087`: Rails' `HelperMethods` is
included on both sides, so both arms exist upstream — port both, do not collapse
to one.

## Acceptance criteria

- Every `validates<Kind>Of` is defined in
  `packages/activemodel/src/validations/<kind>.ts`, and `_mergeAttributes` in
  `packages/activemodel/src/validations/helper-methods.ts` (create it if the
  path table in `docs/ruby-ts-conventions.md` says that is the name).
- `validatesSizeOf` sits in `validations/length.ts`, next to
  `validatesLengthOf`, as `length.rb` has it.
- Both the class and instance arms survive.
- `Model` reaches them through `include()` / `Included<>`.
- `pnpm lint --fix` run after `pnpm parity:api`, so
  `blazetrails/rails-file-structure-method-order` can order the touched files.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Verification

```bash
pnpm vitest run packages/activemodel/src/validations
pnpm parity:api --package activemodel && pnpm lint --fix
```
