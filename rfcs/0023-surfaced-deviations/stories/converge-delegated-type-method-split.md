---
title: "Restore Rails' direction between delegatedType and defineDelegatedTypeMethods"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Raised by the wide call-mismatch ratchet in PR #5503.

trails inverts Rails' split between the two delegated-type methods.

Rails (`activerecord/lib/active_record/delegated_type.rb`): `delegated_type` is
the public macro and calls the private `define_delegated_type_methods`, which
defines `#{role}_class` (`public_send(role_type).constantize`), `#{role}_name`,
`build_#{role}`, and the per-type predicates/scopes.

trails: `defineDelegatedTypeMethods`
(`packages/activerecord/src/delegated-type.ts:232`) is a one-line delegator —
`delegatedType(modelClass, role, { ...options, types })` — and `delegatedType`
(`:64`) owns every definition, including the `constantize` call at `:113` and
`:148`. `base.ts:2085` adds a third hop, a static that forwards to the module
function.

Consequence: the wide ratchet flags `define_delegated_type_methods` for omitting
`constantize` in both `delegated-type.ts` and `base.ts`, because the call is
lexically in the sibling. PR #5503 baselined both with that reason rather than
restructuring mid-review.

## Acceptance criteria

- `defineDelegatedTypeMethods` owns the `#{role}_class` / `#{role}_name` /
  `build_#{role}` / predicate / scope definitions, and `delegatedType` calls it
  — matching Rails' direction.
- The two baselined entries are REMOVED from
  `scripts/api-compare/call-mismatches-wide-exclude/activerecord/delegated-type.json`
  and `.../base.json` (the `define_delegated_type_methods` / `constantize`
  pair), not re-justified. If `base.ts`'s thin static genuinely cannot carry the
  call, its entry stays with that reason alone.
- `delegated-type.test.ts` and `delegated-type-scope.test.ts` stay green.

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
