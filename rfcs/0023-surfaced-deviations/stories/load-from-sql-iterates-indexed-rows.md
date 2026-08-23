---
title: "_loadFromSql iterates indexedRows, not toArray()"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`_loadFromSql` (`packages/activerecord/src/querying.ts`) was converged onto the
`Result` signature by PR #6901 (RFC 0106 wave 5g), but its two instantiation
arms iterate `resultSet.toArray()` where Rails iterates
`result_set.indexed_rows` — `activerecord/lib/active_record/querying.rb:91` and
`:93`:

    if result_set.includes_column?(inheritance_column)
      result_set.indexed_rows.map { |record| instantiate(record, column_types, &block) }
    else
      result_set.indexed_rows.map { |record| instantiate_instance_of(self, record, column_types, &block) }
    end

trails already has the counterpart: `Result#indexedRows`
(`packages/activerecord/src/result.ts:250`) memoizes an `IndexedRow[]`, and
`IndexedRow` (`result.ts:25`) is the port of Rails' `ActiveRecord::Result::IndexedRow`
— it holds the shared `columnIndexes` map plus one raw row array, exposing
`get` / `fetch` / `hasKey` / `keys` / `eachKey` instead of materializing a hash
per row.

The blocker is on the consumer side, not the producer: `Base._instantiate`
(`packages/activerecord/src/base.ts:2827`) and everything it feeds
(`discriminateClassForRecord`, `writeFromDatabase`) take a
`Record<string, unknown>`, so an `IndexedRow` cannot be passed without a
`toArray()`-equivalent materialization somewhere. `toArray()` is memoized via
`hashRows()`, so the current shape is correct but allocates one hash per row —
exactly the allocation Rails introduced `indexed_rows` to avoid.

The deviation is noted at the call site in `_loadFromSql`; it carries no
baseline row because the extractor does not flag `indexed_rows`.

## Acceptance criteria

- [ ] `_instantiate` (and the attribute-hydration path below it) reads rows
      through the `IndexedRow` accessors (`get` / `hasKey` / `keys`) rather than
      requiring a materialized `Record<string, unknown>`.
- [ ] Both arms of `_loadFromSql` iterate `resultSet.indexedRows`, matching
      querying.rb:91 and :93.
- [ ] The `toArray()` call-site note in `_loadFromSql` is deleted, not reworded.
- [ ] No new `call-mismatches-exclude` row and no new `@missingRailsCall` tag.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; SQLite,
      PostgreSQL and MySQL/MariaDB lanes green.
