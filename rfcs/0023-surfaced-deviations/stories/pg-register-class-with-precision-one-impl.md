---
title: "One register_class_with_precision, not two"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# One `register_class_with_precision`, not two

## Context

Rails has a single `AbstractAdapter.register_class_with_precision(mapping, key,
klass, **kwargs)` (`abstract_adapter.rb`), which every adapter reaches through
`self.class.register_class_with_precision` — including PG's instance
`initialize_type_map` (`postgresql_adapter.rb:747-749`).

trails has **two** ports of it, and they are not interchangeable:

- `connection-adapters/abstract-adapter.ts:2392` — `static
registerClassWithPrecision(mapping: TypeMap, ...)`. Its block is
  `(...args: string[]) => this.extractPrecision(args.at(-1)!)`.
- `connection-adapters/postgresql/type-map-init.ts:125` — `export function
registerClassWithPrecision(mapping: HashLookupTypeMap, ...)`. Its block is
  `(_key, ...args) => extractPrecision(sqlTypeFromArgs(args))`.

The difference is real, not cosmetic. `HashLookupTypeMap` forwards
`(lookupKey, ...args)` to the registered block
(`type/hash-lookup-type-map.ts:17`, `:83`), so on a keyless
`typeMap.lookup(oid)` — the shape `PostgreSQLAdapter#lookupCastType` uses after
PR #6581 (`postgresql/quoting.rb:195`) — the abstract static's `args.at(-1)`
resolves to the **OID number** where the PG-local one correctly resolves to
`undefined`. Calling `PostgreSQLAdapter.registerClassWithPrecision` from PG's
instance `initialize_type_map` would therefore change `extractPrecision`'s input
for `time` / `timestamp` / `timestamptz` lookups.

Surfaced in review of PR #6581, which converged PG's instance
`initialize_type_map` (`postgresql_adapter.rb:744-751`) and had to keep the bare
imported PG-local function where Rails writes `self.class.` — justified at the
call site in `postgresql-adapter.ts`, but the underlying duplication is debt,
and the `self.class` dispatch Rails uses is unavailable until it is retired.

`sqlite3-adapter.ts:3065-3067` and `abstract-adapter.ts:2343-2355` use the
static; `postgresql/type-map-init.ts:262-264` and `postgresql-adapter.ts` use
the PG-local one.

## Acceptance criteria

- [ ] One `register_class_with_precision` implementation, on `AbstractAdapter`
      at the Rails name, handling both the `TypeMap` and `HashLookupTypeMap`
      block-argument shapes (the key-first forward is the only difference).
- [ ] `postgresql/type-map-init.ts`'s duplicate is deleted, along with
      `sqlTypeFromArgs`/`fmodFromArgs` if they lose their last caller.
- [ ] PG's instance `initializeTypeMap` calls it through
      `(this.constructor as typeof PostgreSQLAdapter).registerClassWithPrecision`,
      matching `postgresql_adapter.rb:747-749`, and the call-site justification
      added by #6581 is deleted.
- [ ] Regression coverage that fails on the baseline: a `typeMap.lookup(oid)`
      with no sql_type arg for `timestamp` must not pick up a precision derived
      from the OID.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow;
      `pnpm parity:api:calls` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
