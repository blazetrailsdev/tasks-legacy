---
title: "converge EncryptedAttributeType#serialize onto downcase? alone"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 15
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`EncryptedAttributeType#serialize` downcases on `downcase?` alone
(`activerecord/lib/active_record/encryption/encrypted_attribute_type.rb:132`:
`casted_value = casted_value&.downcase if downcase?`), because
`Scheme#initialize` already folds `ignore_case` into `@downcase`
(`activerecord/lib/active_record/encryption/scheme.rb:21`:
`@downcase = downcase || ignore_case`).

trails re-does the fold at the call site:
`packages/activerecord/src/encryption/encrypted-attribute-type.ts:264` reads
`this.scheme.downcase || this.scheme.ignoreCase`. The fold in `Scheme` landed
in PR #6497, so the call-site version is now redundant as well as divergent.

## Acceptance criteria

- [ ] `serialize` consults `this.isDowncase` (the `downcase?` delegate,
      encrypted_attribute_type.rb:15) alone, with no `ignoreCase` term.
- [ ] `pnpm vitest run packages/activerecord/src/encryption` stays green,
      including the `ignore_case` suites in `encryption-schemes.test.ts` and
      `encryptable-record.test.ts`.
