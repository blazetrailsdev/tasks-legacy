---
title: "to-sql.test.ts still builds ~130 visitors on the abstract-adapter quoter"
status: claimed
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 240
priority: null
pr: null
claim: "2026-08-25T15:30:56Z"
assignee: "port-attribute-method-pattern-match-struct"
blocked-by: null
closed-reason: null
---

## Context

`arel-tests-visitor-on-fake-record-connection` (PR #7027) moved every arel test
file with a Rails counterpart onto `fakeRecordConnection`, the port of the
FakeRecord quoting double (`packages/arel/src/test-helpers/connection.ts:90-135`
← `vendor/rails/activerecord/test/cases/arel/support/fake_record.rb:55-90`).
Rails' `Arel::Spec` sets `Table.engine = FakeRecord::Base.new`
(`vendor/rails/activerecord/test/cases/arel/helper.rb:31-33`), so the whole
upstream suite — `visitors/to_sql_test.rb` included, which additionally does
`@visitor = ToSql.new @conn.lease_connection`
(`vendor/rails/activerecord/test/cases/arel/visitors/to_sql_test.rb:10-14`) —
compiles through that double.

`packages/arel/src/visitors/to-sql.test.ts` was deliberately left as PR #7022
converged it: its shared `visitor` (`:30`) is on `fakeRecordConnection`, but
roughly 130 individual sites still construct `new Visitors.ToSql(testConnection)`
inline — `testConnection` being `defaultQuoter`
(`packages/arel/src/test-helpers/default-quoter.ts`), the ActiveRecord _abstract
adapter's_ quoting. The two render differently: `true` is `'t'` vs `TRUE`, `'`
escapes as `\'` vs `''`.

Nobody has audited which of those 130 sites mirror a Rails `to_sql_test.rb` case
(and so must move, taking the Rails literal) versus which are trails-only extras
that deliberately exercise adapter-shaped quoting (binary, JSON, non-finite
numerics) and correctly stay. Doing that audit inside PR #7027 would have pushed
it past the LOC ceiling, and it is a different kind of work from the mechanical
per-file swap that story covered.

This is adjacent to but distinct from `arel-assertion-mark-to-zero`, which
targets the mark counters rather than the connection each visitor is built on.

## Converged shape

Every site in `to-sql.test.ts` whose test mirrors a `to_sql_test.rb` case builds
its visitor on `fakeRecordConnection`, and any assertion that changes as a result
takes the Rails literal rather than a re-derived trails one. Trails-only extras
that exist to exercise adapter quoting keep `testConnection` and say so at the
call site. Ideally the trails-only ones relocate to `to-sql.trails.test.ts`, so
the Rails-mirroring file needs only the one shared visitor.

## Acceptance criteria

- [ ] Each `new Visitors.ToSql(testConnection)` site in
      `packages/arel/src/visitors/to-sql.test.ts` is classified: Rails-mirroring
      (move to `fakeRecordConnection`) or trails-only adapter-quoting coverage
      (keep, with a call-site note).
- [ ] Assertions that change take the Rails literal from `to_sql_test.rb`.
- [ ] No test name is renamed. arel suite green;
      `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry does not
      regress.
