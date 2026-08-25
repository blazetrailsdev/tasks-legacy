---
title: "Retire the ActiveRecord validates override — Rails has none"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/validations.ts:262` exports a `validates` that
`Base` installs as an override of ActiveModel's macro. It hand-dispatches a
fixed table — `presence` / `absence` / `length` / `numericality` / `uniqueness`
routed to the AR validator classes, everything else forwarded to
ActiveModel's `validates` through the late-bound `_parentValidates` slot
(`validations.ts:342-360`, `_setSuperValidates`).

**Rails has no such override.** `activerecord/lib/active_record/validations.rb`
defines no `validates`; grep for `def validates\b` under
`vendor/rails/activerecord/lib/` returns nothing but the `validates_*_of`
helpers. On an AR model, `validates :x, presence: true` runs ActiveModel's
macro unchanged, and its
`const_get("#{key.to_s.camelize}Validator")`
(`activemodel/lib/active_model/validations/validates.rb:121-126`) resolves
`PresenceValidator` to `ActiveRecord::Validations::PresenceValidator`
(`activerecord/lib/active_record/validations/presence.rb:5`) purely because
`ActiveRecord::Validations` is included into `Base` and its constants are on
the lookup path. Same for
`validations/absence.rb:5`, `length.rb:5`, `numericality.rb:5`,
`uniqueness.rb:9`.

So the whole override is inlined-delegation deviation (bucket c): a
12-branch table standing in for a constant lookup, plus a `_parentValidates`
slot and an `ActiveRecordError` raise that exist only to support it.

## Converged shape

PR #6963 already built the hook. `validates`
(`packages/activemodel/src/validations/validates.ts`) resolves each key by
consulting the model class's own statics first and only then the bundled
ActiveModel table:

```ts
const validator = (this as unknown as Record<string, unknown>)[key] ?? BUNDLED_VALIDATORS[key];
```

That is the trails stand-in for Ruby's constant lookup. So:

1. Hang the five AR validator classes on `Base` as statics under their Ruby
   constant names — `PresenceValidator`, `AbsenceValidator`,
   `LengthValidator`, `NumericalityValidator`, `UniquenessValidator` — which
   is what `include ActiveRecord::Validations` gives Ruby for free.
2. Delete `validates`, `_parentValidates`, `_setSuperValidates` and the
   `"ActiveRecord::Validations#validates called before Base registered the
super validates"` raise from `packages/activerecord/src/validations.ts`,
   and drop the `_setSuperValidates` call from `base.ts`.
3. Keep the `validates_*_of` helper overrides
   (`validations.ts:365+`) — those DO exist in Rails
   (`validations/presence.rb:40-42` etc.).

Note that the AR override currently also re-implements the shared
`allowNil` / `allowBlank` merge (`extractShared`, `buildOpts`) that
ActiveModel's converged `validates` now does correctly on its own
(`validates.rb:131` `defaults.merge(...)`), so deleting it removes a second
copy of that logic rather than losing behaviour.

## Acceptance criteria

- `packages/activerecord/src/validations.ts` exports no `validates`,
  `_setSuperValidates`, `_parentValidates`, `extractShared` or `buildOpts`.
- AR models still route `presence` / `absence` / `length` / `numericality` /
  `uniqueness` to the AR validator subclasses, resolved through the class
  statics rather than a branch table.
- `pnpm vitest run packages/activerecord/src/validations packages/activerecord/src/validations.test.ts`
  green, including `uniqueness-validation*.test.ts` and
  `association-validation.test.ts`.
- `pnpm parity:api:extra --package activerecord` loses `validates` from
  `validations.ts`; `pnpm parity:api:calls` / `:args` clean and no new
  baseline rows.
