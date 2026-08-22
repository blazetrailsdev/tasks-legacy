---
title: "Port json_shared_test_cases.rb (25 cases) as a shared module"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 400
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converting the sqlite3 adapter sibling suites to the ambient
connection (PR #5499, RFC 0029).

`vendor/rails/activerecord/test/cases/adapters/sqlite3/json_test.rb` is a
9-line file whose substance comes from the module it includes:

```ruby
class SQLite3JSONTest < ActiveRecord::SQLite3TestCase
  include JSONSharedTestCases
  ...
end
```

`vendor/rails/activerecord/test/cases/json_shared_test_cases.rb` is 286 lines
with **25 test cases** — `test_default`, `test_change_table_supports_json`,
`test_schema_dumping`, `test_cast_value_on_write`, `test_type_cast_json`,
`test_rewrite`, `test_partial_update`, `test_assigning_non_square_hash`,
`test_reload_with_query`, `test_update_all`, and so on.

`packages/activerecord/src/adapters/sqlite3/json.test.ts` ports **2** of them
(`test_assigning_string_literal`, `test_default`). There is no
`json-shared-test-cases` module anywhere in `packages/activerecord/src` — the
only reference to the name is the header comment in this one file. The PG and
MySQL JSON suites, which include the same module in Rails, would share the port.

## Acceptance criteria

- [ ] Port `json_shared_test_cases.rb` as a shared, includable test module
      mirroring Rails' layout, so the sqlite3 / PG / MySQL JSON suites all draw
      from it rather than each re-implementing cases.
- [ ] All 25 cases ported, or the un-portable ones justified individually at the
      call site (Ruby-only semantics) rather than silently dropped.
- [ ] Rails test names preserved verbatim so `parity:test` matches them.
- [ ] Split across PRs to respect the LOC ceiling; register the remainder as
      sibling stories rather than fanning out PRs.
- [ ] `json_data_type` stays an ad-hoc table created in setup, as Rails does —
      it is not canonical schema.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
