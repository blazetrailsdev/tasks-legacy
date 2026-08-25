---
title: "Port SQLite3 remove_foreign_key's own option comparison instead of reusing defined_for?"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AbstractSQLite3Adapter#removeForeignKey`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`, the
`fkToRemove` matcher) resolves the constraint by calling the shared
`ForeignKeyDefinition#isDefinedFor`
(`connection-adapters/abstract/schema-definitions.ts`).

Rails' SQLite3 override does **not** call `defined_for?`. It hand-rolls its own
comparison (`sqlite3/schema_statements.rb:79-80`):

```ruby
options = options.slice(*fk.options.keys)
fk_to_table == table && options.all? { |k, v| fk.options[k].to_s == v.to_s }
```

`to_s` stringifies the _whole_ value, so a Ruby array becomes `"[\"a\", \"b\"]"`
and never equals a scalar. `defined_for?` (`schema_definitions.rb:161-166`)
instead compares element-wise via `Array(...).map(&:to_s)`, which treats
`["a"]` and `"a"` as equal.

Landed in #5450 with a call-site comment justifying the reuse: the two coincide
for every scalar option, and array-valued options are unreachable because
SQLite's `add_foreign_key` is single-column. Reviewer flagged it twice as a
naming/behavior fork worth tracking rather than a bug worth blocking on.

Reproducing Rails exactly needs a Ruby `Array#to_s` inspect-format helper in TS
— check whether one already exists (there is a
`consolidate-ruby-inspect-to-s-helpers` story in this RFC) before writing a new
one.

## Acceptance criteria

- [ ] SQLite's `removeForeignKey` matcher uses the comparison from
      `sqlite3/schema_statements.rb:79-80`, not the generic `isDefinedFor`, or a
      note records why the reuse is the better end state and the call-site
      comment is updated to match.
- [ ] Array-valued vs scalar option values compare the way Rails' SQLite path
      does.
- [ ] `packages/activerecord/src/migration/foreign-key.test.ts` stays green on
      all three adapters.
