---
title: "resetColumnInformation's re-warm thenable and its 135 void markers have no Rails counterpart"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Shipped in PR #6292 (RFC 0064
`reset-column-information-leaves-sync-readers-cold`).

Rails' `reset_column_information` is synchronous and returns nothing
(`activerecord/lib/active_record/model_schema.rb:523-530`):

    def reset_column_information
      connection_pool.active_connection&.clear_cache!
      ([self] + descendants).each(&:undefine_attribute_methods)
      schema_cache.clear_data_source_cache!(table_name)

      reload_schema_from_cache
      initialize_find_by_cache
    end

It clears and stops; its readers re-read the cleared entry lazily on the next
access, blocking on a connection checkout, so the clear is invisible to the
caller. trails' sync readers (`columnsHash()`, `columnNames()`,
`columnDefaults`) cannot block, so a cleared entry reads as empty.

The shipped workaround, in `packages/activerecord/src/model-schema.ts`:

- `resetColumnInformation` returns `PromiseLike<void> | void` instead of `void`
  — the private `rewarmDataSourceCache` thenable, which re-reflects the table
  entry when it is first awaited.
- The thenable is deliberately lazy. An eager fetch races: a caller that does
  not await (a `beforeEach` reset, a `finally` cleanup) can have its in-flight
  reflection resolve after a later DDL statement and write the pre-DDL columns
  back over the fresh entry. That cost a three-lane CI red on #6292 before the
  laziness landed — do not "simplify" it to an `async` function.
- 135 call sites across the suite carry a `void` operator so
  `@typescript-eslint/no-floating-promises` stays green.

None of that exists in Rails. The `void` markers in particular are noise in
ported test bodies whose Rails counterpart is a bare
`Model.reset_column_information`.

## Converged shape

Retire the re-warm entirely: once the sync readers can reach a connection at
read time, `resetColumnInformation` goes back to a plain `void` that only
clears, exactly as Rails does, and every `void` marker and the
`rewarmDataSourceCache` thenable are deleted.

This is gated on the same premise as the other sync-reader debt in
`schema-cache.ts` (`setColumns`' `@internal` note: "once they can block on a
checkout, every population path is `add(pool, tableName)` again") — RFC 0073,
the permanent connection-checkout flip. Do not attempt this story before that
lands; it is filed now so the deviation is tracked rather than settled.

## Acceptance criteria

- [ ] `resetColumnInformation` returns `void` and only clears, matching
      `model_schema.rb:523-530` line for line.
- [ ] `rewarmDataSourceCache` is deleted.
- [ ] Every `void Model.resetColumnInformation()` marker in the suite loses the
      `void`, and the ported bodies read as their Rails counterparts do.
- [ ] The bodies that currently `await` it (`adapters/postgresql/array.test.ts`
      `test_default` / `test_default_strings` /
      `test_change_column_default_with_array`, `persistence.test.ts`
      `becomes default sti subclass` / `reset column information resets
children`) still assert non-vacuously — each must fail on a deliberately
      broken reflection.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
