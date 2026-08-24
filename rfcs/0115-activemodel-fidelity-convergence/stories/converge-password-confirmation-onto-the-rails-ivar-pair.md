---
title: "has_secure_password stores the confirmation in an attribute, not the attr_accessor ivar"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: 6956
claim: "2026-08-23T22:12:31Z"
assignee: "date-suite-is-not-run-by-any-ci-job"
blocked-by: null
closed-reason: null
---

## Context

`secure_password.rb:197` is:

```ruby
attr_accessor :"#{attribute}_confirmation", :"#{attribute}_challenge"
```

Both are plain ivar accessor pairs — the same storage decision as the
`attr_reader attribute` / `define_method("#{attribute}=")` pair at
`secure_password.rb:184-192` that PR #6941 converged for the plaintext itself.

trails splits them. `packages/activemodel/src/secure-password.ts` backs
`passwordChallenge` with a `challengeCache` WeakMap (the ivar, correct), but
declares `passwordConfirmation` as a real ATTRIBUTE:

```ts
if (!modelClass._defaultAttributes().isKey(confirmationAttr)) {
  modelClass.attribute(confirmationAttr, "string");
}
```

with a property whose accessors route to `_readAttribute` / `_writeAttribute`.
The validation then reads it back with `record._readAttribute(confirmationAttr)`.

So one `attr_accessor` line ports as two different storage mechanisms, and the
confirmation half puts a value in `_attributes` that Rails keeps in an ivar —
the same divergence class the plaintext had. It shows in behaviour a Rails dev
would notice: `password_confirmation` participates in the attribute set, so it
reaches dirty tracking, `attributes`, serialization and `assign_attributes`
round-trips where Rails' is invisible to all of them.

Rails: `vendor/rails/activemodel/lib/active_model/secure_password.rb:197`.
trails: `packages/activemodel/src/secure-password.ts` (the `confirmationAttr`
`modelClass.attribute(...)` declaration, its `Object.defineProperty` accessors,
and the `_readAttribute(confirmationAttr)` read inside the `validate` block).

## Converged shape

Back `passwordConfirmation` with a per-instance slot exactly as
`passwordChallenge` already is — a `confirmationCache` WeakMap beside
`challengeCache` — and delete the `modelClass.attribute(confirmationAttr,
"string")` declaration. The validation reads the slot instead of
`_readAttribute`.

Keep the two halves of `attr_accessor` spelled the same way as each other:
after this, `password`, `password_confirmation` and `password_challenge` are
three ivar pairs and NONE of them is an attribute, which is what the Ruby says.

Note the digest is different and must stay an attribute — `password_digest` is
a real database column, written through `public_send("#{attribute}_digest=")`.

## Acceptance criteria

- [ ] `hasSecurePassword` declares no attribute for the confirmation.
- [ ] `passwordConfirmation` is a reader/writer pair over a per-instance slot;
      the value never enters `_attributes`.
- [ ] The confirmation validation reads the slot, not `_readAttribute`.
- [ ] `packages/activemodel/src/secure-password.test.ts` and
      `packages/activerecord/src/secure-password.test.ts` stay green with no
      test renamed.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
