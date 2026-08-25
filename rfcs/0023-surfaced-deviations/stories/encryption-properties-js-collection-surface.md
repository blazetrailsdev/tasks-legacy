---
title: "Converge Properties' JS-collection surface onto properties.rb's member names"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `Properties#initialize` onto Rails' `add` in PR #6493.

`vendor/rails/activerecord/lib/active_record/encryption/properties.rb:16-70`
defines a small, specific surface: `[]=`, `validate_value_type`, `add`, `to_h`,
`ALLOWED_VALUE_CLASSES`, the `DEFAULT_PROPERTIES` accessors, plus
`delegate :==, :[], :each, :key?, to: :data` and `delegate_missing_to :data`.

`packages/activerecord/src/encryption/properties.ts` instead exposes a
JS-collection surface with no Ruby counterpart — `get`, `set`, `has`,
`entries()`, `size`, `toJSON()`, `equals` — and spells the validator
`_validateType` where Rails has the PUBLIC `validate_value_type`. `isKey` is the
port of `key?`, but it sits beside `has`, which duplicates it.

Names are the primary fidelity axis, and this is a whole class's worth of them:
a Rails dev reading `properties.ts` does not recognize the file.

## Converged shape

- `set`/`get` become the ported `[]=` / `[]` spelling the repo already uses for
  Ruby index accessors elsewhere; `has` folds into `isKey` (`key?`).
- `_validateType` becomes the public `validateValueType`
  (`properties.rb:56`), with the `ALLOWED_VALUE_CLASSES` message string
  (`properties.rb:57`).
- `toJSON` becomes `toH` (`properties.rb:69`); `equals` stays as the settled
  port of `==`.
- Update the call sites (`message.ts`, both serializers, `key.ts`,
  `encryptor.ts`, tests) in the same PR.

## Acceptance criteria

- [ ] Every public member of `properties.ts` maps to a `properties.rb` member
      through `docs/ruby-ts-conventions.md`, or carries `@noRailsEquivalent`.
- [ ] `pnpm parity:api:extra --package activerecord` shows no novel names for
      `encryption/properties.ts`.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative; encryption
      suites green.
