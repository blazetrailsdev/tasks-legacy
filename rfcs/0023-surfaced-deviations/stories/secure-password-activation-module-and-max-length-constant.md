---
title: "has_secure_password: InstanceMethodsOnActivation never instantiated, MAX_PASSWORD_LENGTH_ALLOWED inlined"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

# `has_secure_password`: `InstanceMethodsOnActivation` is never instantiated and `MAX_PASSWORD_LENGTH_ALLOWED` is inlined as 72

## Context

Spotted while converging `has_secure_password`'s argument rows in PR #6557
(the two `-> new` rows there carry reviewed reasons). Two separate residues in
`packages/activemodel/src/secure-password.ts`:

**First.** `vendor/rails/activemodel/lib/active_model/secure_password.rb:127`:

        include InstanceMethodsOnActivation.new(attribute, reset_token: reset_token)

trails DEFINES `InstanceMethodsOnActivation`
(secure-password.ts:201-206) — holding only an `attribute` field — and never
constructs or uses it; the accessors are installed with `defineProperty`
directly in `hasSecurePassword`. TS cannot `include` a dynamically-built
module, so the `include` itself cannot port, but a class that exists solely
to be unused is worse than either porting the shape or removing it.

**Second.** `secure_password.rb:153` guards on
`ActiveModel::SecurePassword::MAX_PASSWORD_LENGTH_ALLOWED`; trails inlines
the literal `72` (secure-password.ts:159) and the constant does not exist.

## Converged shape

Carry `MAX_PASSWORD_LENGTH_ALLOWED = 72` on `SecurePassword` under the Rails
name and use it at the guard. For `InstanceMethodsOnActivation`, either route
the accessor installation through an instance of it (so the class earns its
place and matches the Rails structure) or delete it and record the `include` as
unportable — decide with the Rails source in front of you, do not leave it
defined-and-unused.

## Acceptance criteria

- [ ] `MAX_PASSWORD_LENGTH_ALLOWED` exists under the Rails name and is used by
      the too-long guard.
- [ ] `InstanceMethodsOnActivation` is either used or removed; if removed, the
      reason is recorded at the `hasSecurePassword` call site.
- [ ] `pnpm parity:api:extra --package activemodel` shows no new novel surface.
