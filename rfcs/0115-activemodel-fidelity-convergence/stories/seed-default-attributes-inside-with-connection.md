---
title: "Seed _default_attributes' columns inside with_connection instead of a best-effort connection probe"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails wraps `_default_attributes`' column seed in `with_connection`
(`vendor/rails/activerecord/lib/active_record/attributes.rb:241-245`):

    attributes_hash = with_connection do |connection|
      columns_hash.transform_values do |column|
        ActiveModel::Attribute.from_database(column.name, column.default, type_for_column(connection, column))
      end
    end

so `type_for_column` always has a real connection and
`connection.lookup_cast_type_from_column(column)` always resolves.

trails' `_defaultAttributes` (`packages/activerecord/src/attributes.ts`) cannot:
it is a synchronous getter, and `withConnection` is async. It instead probes for
an already-available connection — threaded (in-query), then a directly-assigned
`_adapter`, then `pool.activeConnection` — and passes `undefined` when none is
found, at which point `typeForColumn` falls back to the `value` type. A model
whose first touch is a bare `new Model()` with no leased connection therefore
seeds `value`-typed columns instead of adapter-resolved ones.

The probe is deliberate, not accidental: taking a connection here would force a
permanent checkout on the hot construction path under
`permanent_connection_checkout = :disallowed`. But the fallback is a real
divergence — a PG `uuid`/`jsonb`/`hstore` column seeded as `value` loses its
OID cast until something re-reflects.

The same sync/async seam is why `_defaultAttributes` opens with a
`!isSchemaLoaded` guard that eagerly calls `columnsHash()`; Rails needs no such
probe because `columns_hash` resolves synchronously (`model_schema.rb:427-430`).

## Converged shape

Either make schema reflection synchronously available on the construction path
(the root fix, shared with `ensureSchemaLoaded`'s existence), or give
`_default_attributes` a real `with_connection` acquisition seam of the kind
`0107-relation-ts-decomposition` built for the Arel reader
([[port-with-connection-acquisition-seam-for-the-arel-reader]]). Removing the
`value`-type fallback without one of those regresses the no-permanent-checkout
guarantee, so this is gated on that work rather than shippable alone.

## Acceptance criteria

- [ ] The column seed resolves `type_for_column` against a real connection, as
      `attributes.rb:241-245` does.
- [ ] The `value`-type fallback for an unresolvable column is gone, or the
      residual case is narrowed to genuinely tableless models.
- [ ] `connection-handling.test.ts`'s "common APIs don't permanently hold a
      connection…" still passes on all adapter lanes.
- [ ] The `!isSchemaLoaded` eager `columnsHash()` probe at the top of
      `_defaultAttributes` is removed or justified against `model_schema.rb:427-430`.
