---
title: "MessageVerifier splits on '--' instead of Rails' digest-length separator index"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `MessageVerifier#extract_encoded`
(`vendor/rails/activesupport/lib/active_support/message_verifier.rb:335-350`)
locates the signature by arithmetic, not by splitting: `separator_index_for`
computes `length - digest_length_in_hex - SEPARATOR_LENGTH` and confirms the
separator sits exactly there, where `digest_length_in_hex` comes from the
configured digest (`message_verifier.rb:356-371`).

`packages/activesupport/src/message-verifier.ts` instead does
`signed.split(SEPARATOR)`, takes the last element as the signature, and rejoins
the rest. For well-formed messages the two agree, but they diverge on a payload
whose Base64 body itself contains `--`: Rails anchors on the known digest length,
trails re-joins and can mis-split. `digest_matches_data?` also guards
`data.present? && digest.present?` before `secure_compare`, which the trails
`digestMatches` does not mirror.

This surfaced while porting `Messages::Metadata` (PR #5632): `message_verifier.rb`
sits at 11/15 in parity:api, and these are among the four missing methods.

## Acceptance criteria

- Port `separator_index_for`, `separator_at?`, `digest_length_in_hex`, and
  `digest_matches_data?` and drive `extractEncoded` from them.
- A test covers a message whose encoded body contains the separator.
- parity:api `message_verifier.rb` rises; parity:test non-negative.
