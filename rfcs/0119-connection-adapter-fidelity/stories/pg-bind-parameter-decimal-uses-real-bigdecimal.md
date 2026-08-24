---
title: "PG bind-parameter decimal case stands in a Number for the real BigDecimal"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/adapters/postgresql/bind-parameter.test.ts:53-56`
justifies its decimal case with a claim that is false:

```ts
// Rails passes `BigDecimal(0)` and expects `0.0`. JS has no BigDecimal;
// the nearest value is a plain `Number`, which the adapter renders `0`.
await assertQuotedAs("0", 0);
```

trails _does_ have a real `BigDecimal`
(`packages/activesupport/src/core-ext/big-decimal/conversions.ts`, exported from
`activesupport/src/index.ts:270`), and `quote` dispatches on it ahead of the
numeric arm via `toString("F")`
(`packages/activerecord/src/connection-adapters/abstract/quoting.ts:126-134`,
mirroring Rails `connection_adapters/abstract/quoting.rb:81`). So the stand-in is
unnecessary: the case can assert Rails' literal expectation directly.

PR #5502 did exactly that for the sqlite3 sibling —
`adapters/sqlite3/bind-parameter.test.ts` now passes `new BigDecimal(0)` and
asserts `0.0`, matching
`vendor/rails/activerecord/test/cases/adapters/sqlite3/bind_parameter_test.rb:27-29`.
The PG suite was left alone because it is a different file owned by a different
story. Rails' PG counterpart
(`vendor/rails/activerecord/test/cases/adapters/postgresql/bind_parameter_test.rb`)
has the same `assert_quoted_as "0.0", BigDecimal(0)` case, so confirm the PG
adapter's rendering of a `BigDecimal` before pinning the expectation — PG's
`quote` may differ from SQLite's.

While in the file: the rational case at `:58-62` carries the same stand-in but is
already tracked by RFC 0082 `rational-value-quoting-analogue` — do not duplicate
that work here; only add the citation comment if this story lands first.

## Acceptance criteria

- [ ] `test_where_with_decimal_for_string_column_using_bind_parameters` in the PG
      suite passes a real `BigDecimal` zero and asserts the literal Rails
      expectation for the PG adapter's quoting.
- [ ] The false "JS has no BigDecimal" comment is deleted.
- [ ] Test names unchanged.
