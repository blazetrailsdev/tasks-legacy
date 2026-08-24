---
title: "Converge get_primary_key's schema_cache.primary_keys / table_exists? rows"
status: ready
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 140
priority: 3
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two of the 39 remaining `activerecord` `kind: "set"` rows sit on one Ruby body,
`AttributeMethods::PrimaryKey::ClassMethods#get_primary_key`
(`primary_key.rb:101-108`), and unlike most of the residual population **neither
is owned by another RFC**. Their own reasons say so outright: "the async/sync
split is convergeable work, so the row is what tracks it." Nothing else tracks
it — that is the gap this story closes.

`scripts/api-compare/call-mismatches-exclude/activerecord/attribute-methods/primary-key.json`:

| rubyName          | call            |
| ----------------- | --------------- |
| `get_primary_key` | `primary_keys`  |
| `get_primary_key` | `table_exists?` |

Rails:

    def get_primary_key(base_name)
      if base_name && primary_key_prefix_type == :table_name
        ...
      elsif ActiveRecord::Base != self && table_exists?      # primary_key.rb:103
        pk = schema_cache.primary_keys(table_name)           # primary_key.rb:104
        suppress_composite_primary_key(pk)
      else
        "id"
      end
    end

trails (`packages/activerecord/src/attribute-methods/primary-key.ts:359-377`)
reaches the same values through a lease-free, cache-only path instead:

    const primaryKeys = cachedSchemaCacheFor(this)?.getCachedPrimaryKeys?.(tableName);
    if (primaryKeys !== undefined) return primaryKeys;

- `schema_cache.primary_keys` is **async** in trails; `getCachedPrimaryKeys`
  (declared at `primary-key.ts:165`) is its synchronous cache-only view.
- `table_exists?` is likewise async, and its sync cache-only view
  (`cachedTableExists`) **leases a connection** to reach the cache — which
  `primaryKey` must not do, because it is called from synchronous paths. The
  existing reason argues the two collapse: a table absent from the schema cache
  has no cached primary keys either, so the `undefined` check subsumes the
  `table_exists?` guard.

That argument is plausible but it has never been written down as a decision — it
lives only in a baseline reason string. The RFC's rule is that a row leaves
either as a converged call or as a reviewed `@missingRailsCall` receipt at the
call site, and this pair currently has neither.

Related: `cachedSchemaCacheFor` (`primary-key.ts:189`) is the shared helper both
arms would route through, and the sibling `getPrimaryKeyAttr` /
`primaryKeyAttr` accessors (`:222`, `:264`) are the synchronous callers that
constrain the design. The RFC 0073 connection-lease convergence owns the broader
"`withConnection` returns a Promise" problem; this story is deliberately scoped
to the two rows on this one body and should NOT fork that flip.

## Acceptance criteria

- [ ] Decide the disposition **per row, against `primary_key.rb:101-108`**:
      either `getPrimaryKey` calls the ported `primaryKeys` / `tableExists`
      names, or each call site carries a `@missingRailsCall` receipt naming the
      sync/async split as the reason — classified CONVERGEABLE, not PERMANENT,
      since a synchronous schema-cache read is expressible in TypeScript once
      RFC 0073 settles the lease shape.
- [ ] If the `table_exists?` row is retired by the "absent table has no cached
      primary keys" argument, that argument is stated at the call site in
      `primary-key.ts`, not only in a baseline reason.
- [ ] Both rows are gone from
      `scripts/api-compare/call-mismatches-exclude/activerecord/attribute-methods/primary-key.json`,
      deleted by hand via `serializeBaseline` — no `--write`, no reseed.
- [ ] `pnpm parity:api:calls`, `pnpm parity:api:calls:args` and
      `pnpm parity:api:extra` green, with
      `pnpm parity:api:calls:tighten` run on the affected shard.
- [ ] No connection is leased from `getPrimaryKey` / `getPrimaryKeyAttr` — the
      synchronous callers at `primary-key.ts:222` and `:264` still work with a
      cold cache and no configured connection (the existing `catch` fallthrough
      to `"id"` keeps its behaviour).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
