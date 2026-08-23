---
title: "decompose-instantiate-onto-allocate-init-with-attributes"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`activerecord/querying.json` still carries one `kind: "set"` row —
`_load_from_sql -> instantiate_instance_of` (querying.rb:92). PR #6901 (RFC
0106 wave 5g) converged the file's other row (`inheritance_column`) by making
`_loadFromSql` take the `Result`, but could not converge this one.

The blocker is not in `querying.ts`. Rails' helper is two lines
(`activerecord/lib/active_record/persistence.rb:311-313`):

    def instantiate_instance_of(klass, attributes, column_types = {}, &block)
      attributes = klass.attributes_builder.build_from_database(attributes, column_types)
      klass.allocate.init_with_attributes(attributes, &block)
    end

trails has all three pieces by name — `attributesBuilder`
(`model-schema.ts:711`), and `initWithAttributes` (`core.ts:392`) — but
`initWithAttributes` is only `core.rb:509-510`: it sets `_newRecord` and
`_attributes` and stops. It does NOT do `init_internals`, the `yield self`, or
`_run_find_callbacks` / `_run_initialize_callbacks` (`core.rb:512-517`). All of
that lives instead inside `Base._instantiate` (`base.ts:2827`), together with
the STI re-dispatch, the sync `loadSchema` call, and the
`_suppressInitializeCallback` / `_suppressAbstractCheck` bookkeeping.

So `_instantiate` is trails' single seat for Rails'
`allocate` + `build_from_database` + `init_with_attributes` sequence, and every
caller reaches the sequence through it. Making `_loadFromSql` (or
`instantiate`) call `instantiateInstanceOf` literally does not converge
anything — PR #6901 measured this: exporting the helper so the call could be
made simply relocated the same gap onto a fresh
`instantiate_instance_of -> init_with_attributes` row, so the export was
reverted and the row kept where it is.

## Acceptance criteria

- [ ] `initWithAttributes` (`core.ts:392`) is the full `core.rb:508-520` body:
      `init_internals`, the block yield, `_run_find_callbacks` and
      `_run_initialize_callbacks`, returning `self`.
- [ ] `instantiateInstanceOf` (`persistence.ts:1951`) is Rails' two lines —
      `attributesBuilder().buildFromDatabase(attributes, columnTypes)` then
      `allocate` + `initWithAttributes` — not a `_instantiate` forwarder.
- [ ] `Base._instantiate` keeps only what Rails' callers keep (the STI
      discrimination that `instantiate` does at `persistence.rb:102`, and
      trails' sync schema-load guard), delegating the rest to the two above.
- [ ] `activerecord/querying.json` and the
      `persistence.ts instantiate -> instantiate_instance_of`
      `@missingRailsCall` receipt both converge by deletion.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; SQLite,
      PostgreSQL and MySQL/MariaDB lanes green.
