---
title: "sqlite3 quoting_test: assert quoting methods, not round-tripped rows"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 130
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

`vendor/rails/activerecord/test/cases/adapters/sqlite3/quoting_test.rb` creates
**no tables at all**. Every case asserts a quoting/type-cast return value
directly off the leased connection:

```ruby
def test_quote_string
  assert_equal "''", @conn.quote_string("'")
end

def test_type_cast_true
  assert_equal 1, @conn.type_cast(true)
end

def test_quote_numeric_infinity
  assert_equal "'Infinity'", @conn.quote(Float::INFINITY)
end
```

`packages/activerecord/src/adapters/sqlite3/quoting.test.ts` keeps the Rails
test names but the bodies do something else entirely: each one creates an ad-hoc
table (`quote_test`, `q`, `"my table"`, `bin_enc`, `bool_test`, `bool_test2`,
`bd_test`, `bin_quote`, `time_test`, `time_norm`, `time_utc`, `time_local`,
`inf_test`, `nan_test`), INSERTs a value, SELECTs it back and asserts the
round-tripped datum. It never calls `quoteString`, `quoteColumnName`,
`quoteTableName`, `typeCast` or `quote` — the five methods the suite exists to
pin. A regression in any of them would not fail this file.

Two cases carry call-site notes admitting the substitution, e.g. "Rails asserts
`quote_string("'") == "''"` (escaped content, no wrapping quotes). Trails'
`quoteString` wraps — round-trip the same `'` datum."

PR #5499 only moved the connection these bodies run on; the divergence predates
it and is untouched.

## Acceptance criteria

- [ ] Each case asserts the adapter method's return value, as Rails does, rather
      than a round-tripped row.
- [ ] `test_quote_string` pins the actual contract. Trails' `quoteString` wraps
      in quotes where Rails' returns escape-only content — reconcile against
      RFC 0077 story `dialect-quotestring-returns-literal-not-escape-only`
      (that story owns the behaviour; this one must not fork it) and assert
      whatever the converged contract is.
- [ ] `quote_column_name` / `quote_table_name` assert both the instance and the
      class receiver, as Rails loops `[@conn, @conn.class]`.
- [ ] `quote_numeric_infinity` / `quote_float_nan` assert the rendered
      `'Infinity'` / `'-Infinity'` / `'NaN'` literals. Today the tests assert
      the bound value lands as NULL, which is a different (also real) behaviour
      — if both matter, keep the NULL assertion as a separate trails-only case
      in a `.trails.test.ts`, not under the Rails test name.
- [ ] All 14 ad-hoc tables are gone, along with the teardown drop list.
- [ ] Test names unchanged.
