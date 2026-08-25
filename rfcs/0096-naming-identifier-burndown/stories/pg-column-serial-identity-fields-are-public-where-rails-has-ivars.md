---
title: "PG Column#serial and #identity are public fields where Rails keeps ivars behind predicates"
status: done
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: 7061
claim: "2026-08-25T18:50:35Z"
assignee: "pg-column-serial-identity-fields-are-public-where-rails-has-ivars"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `adapter-column-subclasses-expose-ivars-rails-keeps-private`
(PR #7058), which made PG `generated` and SQLite3 `autoIncrement` private ivars.
The same divergence remains for the PG subclass' two other flags.

`packages/activerecord/src/connection-adapters/postgresql/column.ts:13-14`
still declares:

```ts
serial: boolean;
identity: string | null;
```

In Rails these are ivars with no reader —
`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/column.rb:9-13`
writes `@serial` / `@identity` in `initialize`, `:16-22` exposes them ONLY as
`identity?` / `serial?`, and `:50-60` round-trips them through
`init_with` / `encode_with`. There is no `attr_reader :serial` or
`attr_reader :identity` anywhere in the file.

They do not show up in `pnpm parity:api:extra` because the `isSerial` /
`isIdentity` predicates exist and absorb the match, which is exactly why
PR #7058 scoped them out — the measurement is silent here, but the public
field is still surface Rails does not have.

## Converged shape

- `serial` → `_serial`, `identity` → `_identity` (matching the `_generated`
  spelling PR #7058 established in the same file).
- Writers: the constructor (`column.rb:9-13`) and `initWith` (`:50-55`).
- Readers: `isSerial()` / `isIdentity()` per `:16-22`, `isAutoIncrementedByDb()`
  via those predicates per `:24-26`, and `encodeWith` (`:57-61`), which keeps
  the `"serial"` / `"identity"` coder keys unchanged.
- Repoint every caller at the predicate. `schema-dumper.ts:356` reads
  `(col as any).isSerial === true` as a FIELD-shaped check on the getter — it
  must become a call once the getter becomes a method, or stay a getter if the
  predicate is kept as an accessor.

Note `isSerial` / `isIdentity` are currently getters, not methods, unlike the
`isAutoIncrement()` method PR #7058 added on the sqlite3 sibling; decide one
spelling for the pair while converging.

## Acceptance criteria

- No public `serial` / `identity` field on `PostgreSQL::Column`; both are
  private ivars read only through the predicates.
- `encodeWith` / `initWith` keep the `"serial"` / `"identity"` coder keys, so
  the schema-cache round-trip is unchanged.
- No caller reads the field directly (`schema-dumper.ts` included).
- `pnpm parity:api:calls` / `:args` clean; `extra-surface-mark.json` tightened
  if the totals move, never raised. SQLite, PostgreSQL and MySQL/MariaDB lanes
  green.
