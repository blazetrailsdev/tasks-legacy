---
title: "Remove isValidType's dead snake-to-camel native-type bridge"
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

`AbstractAdapter#isValidType`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`) carries a
snake_case-to-camelCase bridge on top of the direct hash lookup:

```ts
const camel = type.replace(/_([a-z])/g, (_m, c: string) => c.toUpperCase());
return camel !== type && types[camel] != null;
```

Its own comment states the premise: "trails instead keys by its camelCase DSL
name (e.g. `bitVarying`) while `Column#type` stays Rails-snake
(`bit_varying`)". That premise is **false as of PR #5570**, which converged all
three `NATIVE_DATABASE_TYPES` hashes onto Rails' snake_case keys
(`abstract/native-database-types.ts`; Rails at `sqlite3_adapter.rb:69`,
`abstract_mysql_adapter.rb:31`, `postgresql_adapter.rb:134`). Every lookup the
bridge existed to rescue — `primary_key`, `bit_varying` — is now a direct hit.

Rails' `valid_type?` is a bare `native_database_types.key?(type)` with no
spelling normalization
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb`),
so the bridge is now both dead and a divergence: it would still report `true`
for a camelCase spelling Rails rejects.

## Acceptance criteria

- [ ] The camel bridge is removed; `isValidType` is the direct
      `nativeDatabaseTypes()[type] != null` membership test Rails performs.
- [ ] A regression test asserts a camelCase spelling (`bitVarying`) is **not**
      a valid type on PostgreSQL, while `bit_varying` is.
- [ ] `adapter.test.ts`'s "valid column" sweep (every native key is valid)
      still passes on all three lanes.
- [ ] No parity:test or parity:api regression.
