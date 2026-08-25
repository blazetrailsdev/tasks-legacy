---
title: "composed_of's :constructor option is spelled constructorFn and the rename is ratified in the parity tooling"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced landing PR #6828 (`composed-of-local-derivations`, RFC 0099), which
ported `composed_of`'s six local derivations
(`vendor/rails/activerecord/lib/active_record/aggregations.rb:225-245`).

Rails' option is `:constructor` (`aggregations.rb:236`, `:240`). trails spells
it `constructorFn` in `ComposedOfOptions`
(`packages/activerecord/src/aggregations.ts:44`) and the parity tooling
ratifies the rename in two places:

- `scripts/api-compare/options-keys.ts:15-19` — `OPTION_KEY_RENAMES`
- `scripts/api-compare/call-args.ts:295-297` — the kwargs normalizer comment

The stated reason is that `constructor` is "reserved as a JS object-property
name". It is not reserved: it is a legal key on an object literal. The real
hazard is narrow — a plain object with no own `constructor` key answers
`options.constructor` with `Object`, so a bare `options.constructor ?? "new"`
read is wrong — and it has a one-line fix that Rails' spelling survives:
`Object.hasOwn(options, "constructor") ? options.constructor : "new"`.

## Converged shape

`ComposedOfOptions.constructor`, read through an own-property check in
`composedOf`, with `OPTION_KEY_RENAMES` in `scripts/api-compare/options-keys.ts`
emptied of the `constructor` entry and the `call-args.ts` comment updated.
Call sites: `packages/activerecord/src/test-helpers/models/customer.ts:137`.

## Acceptance criteria

- [ ] `composed_of`'s constructor option is spelled `constructor`, as Rails does.
- [ ] The `constructor -> constructorFn` row is gone from `OPTION_KEY_RENAMES`;
      `scripts/api-compare/options-keys.test.ts` and `call-args.test.ts` updated.
- [ ] `pnpm parity:api` option-key diff for `composed_of` does not regress.
