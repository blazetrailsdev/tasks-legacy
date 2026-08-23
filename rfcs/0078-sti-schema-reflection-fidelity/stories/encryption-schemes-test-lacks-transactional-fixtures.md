---
title: "encryption-schemes.test.ts lacks transactional fixtures, forcing a trails-only deleteAll"
status: claimed
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-23T17:44:08Z"
assignee: "encryption-schemes-test-lacks-transactional-fixtures"
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/encryption/encryption-schemes.test.ts` runs without
transactional fixtures: it calls `freshAdapter()` per case against the shared
canonical `authors` table and never rolls back. Rails' counterpart
(`activerecord/test/cases/encryption/encryption_schemes_test.rb`, an
`ActiveRecord::EncryptionTestCase` → `ActiveRecord::TestCase`) gets
`use_transactional_tests` for free, so each `Class.new(Author) { encrypts :name
... }` case starts from an empty table.

PR #6925 hit this converging `don't use global previous schemes with a
different deterministic nature when performing queries`
(encryption_schemes_test.rb:166-180): the row created by the sibling case at
:120-133 carries the identical deterministic ciphertext for "STEPHEN KING", so
`find_by_name("STEPHEN KING")` returns the older row and the
`assert_equal author, ...` identity check fails. The shipped workaround is a
`deleteAll()` before the create, with a call-site comment — a trails-only line
Rails does not have.

## Acceptance criteria

- [ ] `encryption-schemes.test.ts` runs under transactional fixtures (the
      `fixtures([...])` / `withTransactionalFixtures` shape used by
      `extended-deterministic-queries.test.ts`), matching Rails'
      `use_transactional_tests`.
- [ ] The `deleteAll()` line and its comment in the "... when performing
      queries" case are removed.
- [ ] Test names unchanged; `encryption/` suites pass.
