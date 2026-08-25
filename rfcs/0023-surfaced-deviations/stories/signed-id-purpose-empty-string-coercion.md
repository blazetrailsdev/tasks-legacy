---
title: "signed_id purposes carry a trails-only || undefined coercion"
status: draft
updated: 2026-08-11
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

# `signed_id` purposes carry a trails-only `|| undefined` coercion

## Context

Surfaced while deleting the trails-only `combinePurposes` wrapper in PR #6361.
The wrapper is gone and all three sites now call
`Base.combineSignedIdPurposes` (matching `self.class.combine_signed_id_purposes`
at `vendor/rails/activerecord/lib/active_record/signed_id.rb:135`, `:141`,
`:147`), but its `|| undefined` coercion had to be inlined at each site rather
than dropped:

```ts
purpose: modelClass.combineSignedIdPurposes(options?.purpose) || undefined,
```

Rails passes the combined purpose straight through — `signed_id.rb:135` is
`purpose: self.class.combine_signed_id_purposes(purpose)` with no coercion. The
coercion exists because trails' `MessageVerifier` distinguishes an empty-string
purpose from an absent one where Rails' does not, so an un-purposed,
un-scoped signed id would otherwise be generated and verified under `""`.

## Converged shape

Fix the asymmetry in the verifier rather than at the three call sites: make
`MessageVerifier#generate`/`verified`/`verify` treat an empty purpose as absent
(or confirm `combine_signed_id_purposes` can never return `""` and drop the
coercion outright — `signed_id.rb:170-172` joins `[base_class.name.underscore,
purpose]`, so it is non-empty whenever the model has a name).

## Acceptance criteria

1. The three `|| undefined` coercions in
   `packages/activerecord/src/signed-id.ts` are gone; the call sites read as
   signed_id.rb:135, :141, :147 do.
2. Signed-id round-trips still work un-purposed and purposed, with and without
   `signed_id_verifier_secret`.
3. `pnpm vitest run packages/activerecord/src/signed-id.test.ts` green.
