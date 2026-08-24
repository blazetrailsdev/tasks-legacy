---
title: "Converge PostgreSQL::ExplainPrettyPrinter#pp onto Rails' positional column read"
status: draft
updated: 2026-08-03
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while converging the sqlite3 `ExplainPrettyPrinter` in #5934. Rails'
`PostgreSQL::ExplainPrettyPrinter#pp`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/explain_pretty_printer.rb`)
is column-name agnostic:

```ruby
def pp(result)
  header = result.columns.first
  lines  = result.rows.map(&:first)
  ...
end
```

trails' `packages/activerecord/src/connection-adapters/postgresql/explain-pretty-printer.ts:8-30`
instead takes `Array<Record<string, unknown>>` and guesses the plan column by
name (`row["QUERY PLAN"] ?? row.query_plan ?? row.queryplan ?? ""`), then
hardcodes the literal `"QUERY PLAN"` header rather than reading
`result.columns.first`. It also returns `""` for an empty result where Rails
still emits the header block.

This is the same class of divergence that #5934 fixed for sqlite3
(`sqlite3/explain-pretty-printer.ts`), which now takes `{ rows }` and joins
positionally. The mysql printer
(`connection-adapters/mysql/explain-pretty-printer.ts`) already takes the
generic `{ columns, rows }` shape, so pg is the last one out of line.

## Acceptance criteria

- `PostgreSQL::ExplainPrettyPrinter#pp` takes the generic `{ columns, rows }`
  (or raw `Result`) shape and reads the header from `columns[0]` and each line
  from `rows[i][0]`, matching the vendored Ruby.
- The empty-result and FORMAT JSON arms match Rails' output (the JSON pretty
  print in trails is a trails extra — either justify it at the call site or
  drop it, do not leave it undocumented).
- `packages/activerecord/src/adapters/postgresql/explain.test.ts` stays green on
  the pg lane.
