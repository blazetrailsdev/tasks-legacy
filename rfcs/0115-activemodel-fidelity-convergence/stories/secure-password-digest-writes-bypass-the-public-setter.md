---
title: "has_secure_password writes the digest via _writeAttribute, not public_send"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: 6956
claim: "2026-08-23T22:12:31Z"
assignee: "date-suite-is-not-run-by-any-ci-job"
blocked-by: null
closed-reason: null
---

## Context

Rails' generated `password=` writes the digest through the PUBLIC setter, twice:

```ruby
self.public_send("#{attribute}_digest=", nil)                                    # :188
self.public_send("#{attribute}_digest=", BCrypt::Password.create(..., cost: cost)) # :193
```

(`vendor/rails/activemodel/lib/active_model/secure_password.rb:186-194`.)

trails writes it through the private attribute API instead —
`packages/activemodel/src/secure-password.ts`, inside the `password` property's
`set`:

```ts
this._writeAttribute(digestAttr, null);
this._writeAttribute(digestAttr, bcrypt.hashSync(String(unencryptedPassword), cost));
```

`public_send` on a setter is not decoration in Rails: it dispatches through
whatever the model has defined for `password_digest=`, so a class that
overrides the digest writer (or an `attribute :password_digest` with a custom
type, or an AR attribute alias) sees the write. `_writeAttribute` bypasses all
of that and stores straight into the attribute set. The same substitution
appears in the salt reader and the authenticate method, which read
`this._readAttribute(digestAttr)` where Rails does
`public_send("#{attribute}_digest")` (`:189`, `:212`).

This predates PR #6941 — that PR converged the STORAGE decision (the plaintext
ivar pair, deleting the invented `after_initialize`) and deliberately left the
digest write mechanism alone as out of its scope.

## Converged shape

Dispatch through the generated public accessor, as Ruby's `public_send` does:
read and write `password_digest` by property on the record rather than through
`_readAttribute` / `_writeAttribute`, at all four sites (the two writes in the
setter, the salt reader, and `authenticate`).

Check the ordering constraint before doing it: the digest accessor is generated
by `modelClass.attribute(digestAttr, "string")` at `hasSecurePassword` time, so
confirm the property is actually present on the prototype when the writer runs
(see CLAUDE.md "Generated attribute readers are properties" — reader generation
skips a name the class body already answers). If it is not reachable, that
ordering IS the finding, and the story converges it rather than ratifying the
bypass.

## Acceptance criteria

- [ ] The password writer writes the digest through the public
      `password_digest` accessor, both the nil arm and the hash arm.
- [ ] The salt reader and `authenticate` read it the same way.
- [ ] A test proves dispatch: a subclass overriding the digest writer sees the
      write (Rails' `public_send` semantics), which the current code fails.
- [ ] `packages/activemodel/src/secure-password.test.ts` and
      `packages/activerecord/src/secure-password.test.ts` stay green with no
      test renamed.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
