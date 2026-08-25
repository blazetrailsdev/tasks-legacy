---
title: "nested-attributes-reject-helper-member-order"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Restore Rails member order for the nested-attributes reject helpers

## Context

Surfaced while reviewing PR #6406 (RFC 0099). Predates that PR — the diff did
not touch member order — so it was flagged rather than fixed there.

`packages/activerecord/src/nested-attributes.ts` declares the four private
reject helpers in the REVERSE of Rails' order. Rails
`activerecord/lib/active_record/nested_attributes.rb`:

- `reject_new_record?` — :589
- `call_reject_if` — :598
- `will_be_destroyed?` — :610
- `allow_destroy?` — :614

trails declares them `isAllowDestroy`, `isWillBeDestroyed`, `callRejectIf`,
`isRejectNewRecord` — exactly backwards.

Member order within a file is a stated fidelity axis (CLAUDE.md "Decomposition"
/ "file layout"), and `blazetrails/rails-file-structure-method-order` already
encodes the rule — it just isn't armed for `activerecord` yet, which is why this
went unnoticed.

The four bodies are `this`-typed as of #6406, so this is a pure move: no
signature or call-site change. Check whether the surrounding private helpers in
the same file (`has_destroy_flag?`, `raise_nested_attributes_record_not_found!`,
`find_record_by_id`, `assign_to_or_mark_for_destruction`) are also out of order
against `nested_attributes.rb:520-630` and move them in the same pass.

## Acceptance criteria

1. The four helpers appear in `nested-attributes.ts` in Rails' declaration
   order, verified against `vendor/rails/activerecord/lib/active_record/nested_attributes.rb`.
2. Any other member of the same file found out of order against the same Rails
   file is moved too.
3. Pure move: no body, signature, or call-site change.
4. `pnpm parity:api`, `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
   the nested-attributes suite stay green.
