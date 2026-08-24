---
title: "data_source_exists?/table_exists?/view_exists? return false for a blank name where Rails returns nil"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 17
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `data_source_exists?` / `table_exists?` / `view_exists?` return `false` for a blank name, Rails returns `nil`

## Context

Spotted while converging the `.any?` call-set rows in these three methods
(PR #6560, RFC 0106). The `any?` half converged; this arm did not and was left
alone as out of scope for that row.

Rails guards with a **trailing `if` modifier**, so a blank name falls off the end
of the method and the return value is `nil`, not `false`:

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:44-48`

```ruby
def data_source_exists?(name)
  query_values(data_source_sql(name), "SCHEMA").any? if name.present?
rescue NotImplementedError
  data_sources.include?(name.to_s)
end
```

Same shape at `schema_statements.rb:59-63` (`table_exists?`) and
`schema_statements.rb:74-78` (`view_exists?`).

trails writes an explicit early return instead
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
`dataSourceExists` / `tableExists` / `viewExists`):

```ts
if (!isPresent(name)) return false;
```

`nil` and `false` are both falsy in Ruby _and_ in JS, so no current caller
observes the difference — but this is a value-returning predicate, and a caller
that distinguishes "not asked" from "asked and absent" (or that serializes the
result) would. It is the `bang`/predicate return-value class from
CLAUDE.md's "Ruby idioms that do not translate literally".

## Converged shape

Return `null` rather than `false` on the blank arm, and widen the return type to
`Promise<boolean | null>` — mirroring the `if` modifier's fall-through.

## Acceptance criteria

- [ ] All three methods return `null` (not `false`) when the name is blank,
      matching the `if name.present?` modifier at schema_statements.rb:45,60,75.
- [ ] Every call site is checked against Ruby truthiness rules — a caller reading
      the result as a boolean must still behave identically.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
