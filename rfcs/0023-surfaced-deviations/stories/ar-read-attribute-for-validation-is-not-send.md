---
title: "AR readAttributeForValidation is a 5-branch resolver where Rails aliases send"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6978 (`retire-seed-association-cache-test-helper`).

Rails does not implement `read_attribute_for_validation` at all — it is an
alias for `send`:

```ruby
# activemodel/lib/active_model/validations.rb:437
alias :read_attribute_for_validation :send
```

ActiveRecord does **not** override it (no other definition exists under
`activerecord/lib/`). So `record.read_attribute_for_validation(:replies)` is
literally `record.replies` — the generated association reader — and
`record.read_attribute_for_validation(:title)` is the generated attribute
reader. One dispatch, no special cases. It is called from exactly two places:
`activemodel/lib/active_model/validator.rb:152` and
`activemodel/lib/active_model/error.rb:66`.

trails' ActiveRecord override
(`packages/activerecord/src/validations.ts:202-234`) is instead a five-branch
bespoke resolver, tried in this order:

1. `this._collectionProxies.get(attribute)` when `loaded === true` or its
   `target` is a non-empty array,
2. `this.association(attribute)` when `loaded === true` or `target != null`,
3. `this._associationCache(attribute)?.target`,
4. `_preloadedHolderTarget(this, attribute)`,
5. `this.readAttribute(attribute)`.

None of these is `send`. The consequences are concrete, not theoretical:

- **A stubbed reader is invisible.** Rails'
  `association_validation_test.rb:51-59` and `i18n_validation_test.rb:29-35`
  reach their fixtures by overriding the reader itself
  (`t.define_singleton_method(:replies) { [reply.new] }`). The JS spelling —
  `Object.defineProperty(t, "replies", { get })` — is not read by ANY of the
  five branches, so those Rails tests have no literal port. PR #6978 had to
  route each one through a real association instead, which is a fine test but
  is not what the Rails test exercises.
- **It is why the trails-only `support/seed-association-cache.ts` existed** (a
  helper poking branch 3 directly). PR #6978 deleted the helper; the resolver
  that made it necessary is still here.
- **Branch order is load-bearing and undocumented against Rails**, because
  Rails has no order to mirror. Branch 1's `target.length > 0` arm in
  particular makes an unloaded-but-nonempty proxy win over the holder, a rule
  no Rails line states.

## Converged shape

`readAttributeForValidation` becomes the JS spelling of `send`: read the member
named `attribute` off the record and return it, so a declared association
resolves through its generated reader (`associations/builder/association.rb:102-108`)
and a column through its generated attribute reader — the same single dispatch
Rails has. The five branches then live (if they are needed at all) inside the
readers they belong to, not in front of them.

Expect this to be the hard part: trails' association readers are async where
Ruby's are not, which is likely the original reason for the bypass. Establish
whether the sync reader path (`_associationCache` / preloaded holder) can be
reached THROUGH the reader rather than around it before writing code; if a
genuine TypeScript shortcoming blocks full convergence, converge as far as the
language allows and cite it at the call site.

## Acceptance criteria

- [ ] `readAttributeForValidation` dispatches by member name, mirroring
      `activemodel/lib/active_model/validations.rb:437`'s `alias … :send`.
- [ ] A test overriding an association reader on a single instance (the JS
      spelling of `define_singleton_method`) is seen by validation, so
      `association_validation_test.rb:51-59` can be ported literally.
- [ ] Any branch that survives is justified at the call site as a language
      shortcoming, not as a preference.
- [ ] The AR validations suites stay green with unchanged test names.
