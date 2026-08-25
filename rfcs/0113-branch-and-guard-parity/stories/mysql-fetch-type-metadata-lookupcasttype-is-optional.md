---
title: "Make lookupCastType required and drop fetchTypeMetadata's invented fallback arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6483, which converged `new_column_from_field` /
`default_type` onto Rails' `(table_name, field_name)` signatures and
introduced `MysqlColumnReflectionHost` as the `self` those two reach
`create_table_info` / `lookup_cast_type` through.

Rails always routes type metadata through `lookup_cast_type`. There is no
"no type map" arm — `fetch_type_metadata` calls `super`, and the abstract
implementation is `lookup_cast_type(sql_type)`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb:221-223`,
`abstract/schema_statements.rb`):

```ruby
def fetch_type_metadata(sql_type, extra = "")
  MySQL::TypeMetadata.new(super(sql_type), extra: extra)
end
```

trails keeps two divergences from that:

1. `MysqlColumnReflectionHost.lookupCastType` is declared **optional**
   (`packages/activerecord/src/connection-adapters/mysql/schema-statements.ts`,
   the host interface), and `newColumnFromField` branches on its presence.
2. The free `fetchTypeMetadata(sqlType, extra, lookupCastType?)` carries a
   third parameter Rails does not have, plus a fallback arm that strips
   `unsigned` / `zerofill` with a regexp when no lookup is supplied — a
   hand-rolled second implementation of type resolution.

The optionality exists only so unit tests can build a host stub without a
type map. That is a test-shape problem, not a language shortcoming.

## Converged shape

- Make `lookupCastType` required on the host and drop the presence branch
  in `newColumnFromField`.
- Drop `fetchTypeMetadata`'s third parameter and its modifier-stripping
  fallback; it takes `(sqlType, extra)` and reaches `lookupCastType`
  through `this`, as Rails' `fetch_type_metadata` reaches it through
  `super`.
- Update `schema-statements.test.ts`'s `reflectionHost` helper to supply a
  real `lookupCastType` (the adapter's own, or a small map over the sql
  types the tests use) rather than relying on the fallback.

## Acceptance criteria

- [ ] No optional `lookupCastType`, no fallback arm, no third parameter.
- [ ] `fetchTypeMetadata`'s signature matches Rails' arity and order.
- [ ] MySQL/MariaDB column-reflection suites pass.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
