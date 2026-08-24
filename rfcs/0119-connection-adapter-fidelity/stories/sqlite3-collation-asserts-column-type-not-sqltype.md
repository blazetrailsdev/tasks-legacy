---
title: "sqlite3 collation_test: assert column.type, not a sqlType regex"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Surfaced while converting the sqlite3 adapter sibling suites to the ambient
connection (PR #5499, RFC 0029).

`collation_test.rb` asserts the **abstract column type** Rails reflects:

```ruby
test "string column with collation" do
  column = @connection.columns(:collation_table_sqlite3).find { |c| c.name == "string_nocase" }
  assert_equal :string, column.type
  assert_equal "NOCASE", column.collation
end
```

`packages/activerecord/src/adapters/sqlite3/collation.test.ts` asserts the raw
`sqlType` against a regex instead:

```ts
expect(stringNocase.sqlType?.toLowerCase()).toMatch(/varchar|char/);
```

Same for the text case (`sqlType === "text"` where Rails asserts
`column.type == :text`) and the `add column` / `change column` cases.

`sqlType` is the DDL string echoed back by PRAGMA; `type` is the reflected
abstract type, which is what the adapter's type map actually decides. The regex
form passes for any `VARCHAR`/`CHAR` spelling and would keep passing if the type
map stopped mapping it to `:string` — the exact reflection bug the assertion is
meant to catch. Related: `project_valid_type_gate_surfaces_type_reflection_divergences`.

## Acceptance criteria

- [ ] All five cases assert `column.type` (`"string"` / `"text"`) as Rails does.
- [ ] `collation` assertions unchanged (they already match).
- [ ] If a `sqlType` assertion carries independent value, keep it as an
      additional expectation, not as the replacement for `type`.
- [ ] Test names unchanged.
