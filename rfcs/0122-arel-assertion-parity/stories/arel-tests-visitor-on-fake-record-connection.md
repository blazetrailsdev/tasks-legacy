---
title: "Arel tests build visitors on the abstract-adapter quoter where Rails uses FakeRecord"
status: done
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: 7027
claim: "2026-08-25T12:22:53Z"
assignee: "converge-attribute-deep-dup-onto-ruby-dup"
blocked-by: null
closed-reason: null
---

## Context

Rails' arel suite builds every visitor on the FakeRecord double —
`@conn = FakeRecord::Base.new; @visitor = ToSql.new @conn.lease_connection`
(`vendor/rails/activerecord/test/cases/arel/visitors/to_sql_test.rb:10-14`),
and the same two lines open `select_manager_test.rb`, `predications_test.rb`,
`attributes/attribute_test.rb` and the rest. FakeRecord's `quote` is a specific,
small thing (`test/cases/arel/support/fake_record.rb:71-87`): `true`/`false`
quote as `'t'`/`'f'`, `Date`/`DateTime` use `strftime`, and the `else` arm
escapes `'` as `\'`.

trails has a faithful port of it — `fakeRecordConnection`
(`packages/arel/src/test-helpers/connection.ts:90-135`) — but most arel test
files still build visitors on `testConnection`, which is `defaultQuoter`
(`test-helpers/default-quoter.ts`), a port of the ActiveRecord **abstract
adapter's** quoting: `true` renders `TRUE`, `'` escapes as `''`, values go
through `quoted_date`. Those are different renderings, so a test written against
`testConnection` cannot assert the Rails literal, and the assertion-value
mismatch is unfixable inside the test.

PR #7022 converged `visitors/to-sql.test.ts` by moving it onto
`fakeRecordConnection`, which is what let `should visit_TrueClass` assert
`"users"."bool" = 't'` and `should visit string subclass` assert the backslash
escape. Every remaining RFC 0122 file will hit the same wall on its own
`true`/`false`/quote-escape assertions.

## Converged shape

Move each arel test file that mirrors a Rails arel test onto
`fakeRecordConnection`, matching the Rails `before` block, and take the Rails
literals in the assertions that change as a result. `testConnection` stays for
trails-only tests that deliberately exercise adapter-shaped quoting (binary,
JSON, non-finite numerics) — those have no Rails counterpart and are not what
this story touches.

## Acceptance criteria

- Every test file with a `to_sql`-style Rails counterpart builds its visitor on
  `fakeRecordConnection`, per each file's own Rails `before` block.
- Assertions that change as a result take the Rails literal, not a re-derived
  trails one.
- No test name is renamed. arel suite green;
  `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry tightens.
