---
title: "MySQL::Column takes unsigned/auto_increment/virtual/extra as constructor options where Rails derives them"
status: draft
updated: 2026-08-13
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`MySQL::Column` derives its three predicates at read time from state it already
holds, and delegates `extra` to the type metadata
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/column.rb:7-24`):

```ruby
delegate :extra, to: :sql_type_metadata, allow_nil: true

def unsigned?
  /\bunsigned(?: zerofill)?\z/.match?(sql_type)
end

def auto_increment?
  extra == "auto_increment"
end

def virtual?
  /\b(?:VIRTUAL|STORED|PERSISTENT)\b/.match?(extra)
end
```

trails' `packages/activerecord/src/connection-adapters/mysql/column.ts:11-40`
instead carries `unsigned`, `autoIncrement`, `virtual` and `extra` as four
readonly fields seated from constructor options, and
`newColumnFromField`
(`packages/activerecord/src/connection-adapters/mysql/schema-statements.ts:390-400`)
computes all four from `field["Type"]` / `field["Extra"]` at construction time
and passes them in. So the regexes live at the call site rather than on the
column, `extra` is a stored copy rather than a delegation to
`sqlTypeMetadata.extra`, and any other construction path (a column built from
a TypeMetadata that already carries `extra`) silently gets `false` for all
three.

Surfaced while shipping
`mysql-new-column-from-field-folds-on-update-into-default-generated` (PR #6451),
which converged the `DEFAULT_GENERATED` branch of the same method and left the
column-construction options untouched as out-of-scope.

## Converged shape

- `extra` becomes a getter delegating to `this.sqlTypeMetadata.extra`
  (`allow_nil: true` — null metadata answers null, not `""`).
- `isUnsigned` / `isAutoIncrement` / `isVirtual` become predicates on
  `MySQL::Column` computing from `sqlType` / `extra`, with Rails' exact regexes
  (`/\bunsigned(?: zerofill)?$/` end-anchored, no `/i`;
  `/\b(?:VIRTUAL|STORED|PERSISTENT)\b/`) and Rails' `extra == "auto_increment"`
  equality.
- `newColumnFromField` stops computing and passing them; the constructor's
  `unsigned` / `autoIncrement` / `virtual` / `extra` options go away.
- Note the two predicates are also spelled `auto_incremented_by_db?`
  (`alias_method`, column.rb:20) and `case_sensitive?` (`:14-16`) — check both
  are present while in the file.

## Acceptance criteria

- [ ] `MySQL::Column` computes `unsigned?`, `auto_increment?` and `virtual?`
      from `sql_type` / `extra`; no caller passes them in.
- [ ] `extra` delegates to `sqlTypeMetadata`, nil-safe.
- [ ] MySQL and MariaDB lanes green, including schema dumps of generated
      (VIRTUAL/STORED) and unsigned columns.
