---
title: "Make the signed-id verifier memo per-class like Rails @signed_id_verifier"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/signed-id.ts` memoizes the verifier as a static
property: `(this as any)._signedIdVerifier = verifier`. JS static property
lookup walks the prototype chain, so a subclass that has never built its own
verifier reads `Base`'s memo.

Rails memoizes into a per-class instance variable —
`@signed_id_verifier ||= ...`
(`vendor/rails/activerecord/lib/active_record/signed_id.rb:82`) — and Ruby class
ivars are NOT inherited: `Account.signed_id_verifier` builds its own from
`signed_id_verifier_secret` even when `ActiveRecord::Base` already has one.

Observable where a model's `signed_id_verifier_secret` differs from Base's, and
in `Account.instance_variable_set :@signed_id_verifier, nil`-style resets
(`vendor/rails/activerecord/test/cases/signed_id_test.rb:161,172`), where
clearing the subclass memo should fall through to a rebuild, not to Base's
cached verifier.

Surfaced while reviewing PR #5907 (converge-signed-id-verifier-secret-writer);
pre-existing, not introduced there.

## Acceptance criteria

- The verifier memo is per-class: a subclass with no memo of its own builds a
  fresh verifier rather than reading `Base`'s.
- `Base.signedIdVerifier` and `Account.signedIdVerifier` remain distinct after a
  subclass memo is cleared.
- `signed-id.test.ts` stays green; test names unchanged.
