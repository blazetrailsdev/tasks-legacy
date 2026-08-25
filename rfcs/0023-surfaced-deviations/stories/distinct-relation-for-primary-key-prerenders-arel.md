---
title: "distinct_relation_for_primary_key pre-renders arel where Rails passes the tree to select_rows"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`distinct_relation_for_primary_key` passes the **arel object** to `select_rows`
(`activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb`,
the `distinct_relation_for_primary_key` body):

```ruby
select_rows(relation.arel, "SQL")
```

The port
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1726`)
pre-renders it to a string first:

```ts
const sql = typeof arel === "string" ? arel : (arel?.toSql?.() ?? String(arel));
... this.selectRows(sql, "SQL")
```

Rendering at the call site means the statement never reaches `to_sql_and_binds`
as an Arel tree, so it cannot produce binds — the exact class of gap recorded in
`project_bind_path_not_exercised_by_create_find_roundtrip`. It surfaced as an
RFC 0095 `naming` call-argument row (`select_rows` ruby `ref:arel` vs ts
`ref:sql`) and PR #6353 left it as an invented conversion rather than a rename.

## Converged shape

Pass the Arel object through to `selectRows` and let the adapter's
`to_sql_and_binds` render it, as every other `select_*` call site does. If
`selectRows` cannot currently accept an Arel tree, widening it is the work —
that is where Rails' bind path lives.

## Acceptance criteria

1. `distinctRelationForPrimaryKey` passes the Arel relation (not a pre-rendered
   string) to `selectRows`, matching schema_statements.rb.
2. The rendered SQL is unchanged for the existing callers, with a test that
   pins the emitted statement.
3. The `select_rows` `naming` row is gone from
   `pnpm parity:api:calls:args:report`, not baselined.
