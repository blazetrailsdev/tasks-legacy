---
title: "Port database_statements_test.rb's test_exec_insert (last_inserted_id has no coverage)"
status: draft
updated: 2026-07-28
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

Surfaced while shipping `long-tail-memory-sites-ambient` (PR #5495, RFC 0029),
which rewrote `packages/activerecord/src/database-statements.test.ts` to lease
the ambient connection.

Rails' `vendor/rails/activerecord/test/cases/database_statements_test.rb`
defines **three** tests; trails ports two:

```ruby
def test_exec_insert                              # <- NOT PORTED
  result = @connection.exec_insert("INSERT INTO accounts (firm_id,credit_limit) VALUES (42,5000)", nil, [])
  assert_not_nil @connection.send(:last_inserted_id, result)
end

def test_insert_should_return_the_inserted_id      # ported
def test_create_should_return_the_inserted_id      # ported
```

`test_exec_insert` is the only one of the three that exercises
`exec_insert` + the private `last_inserted_id(result)` reader — the two ported
tests both go through `insert`/`create`, which return the id directly. So the
`last_inserted_id` path has no coverage from this file.

This was not in `long-tail-memory-sites-ambient`'s scope (that story was about
`":memory:"` sites), and the file now rides the ambient connection with
`fixtures({})`, so adding the third test is a small, self-contained follow-up
rather than a rewrite.

Worth checking while porting: `lastInsertedId` should be reachable on all three
adapters, and `execInsert`'s trails spelling/arity should be confirmed against
`abstract/database-statements.ts` before assuming the Rails signature carries
over verbatim.

## Acceptance criteria

- [ ] `database-statements.test.ts` gains `it("exec insert", ...)` matching the
      Rails test name as `parity:test` derives it from `test_exec_insert`.
- [ ] It calls `execInsert` on the ambient connection and asserts the
      `lastInsertedId` of the returned result is not null, mirroring
      `database_statements_test.rb:10-13`.
- [ ] The test runs on all three lanes (the file has no adapter gate today).
- [ ] No existing test renamed. `parity:test` delta > 0 for this file.
