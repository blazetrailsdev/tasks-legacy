---
title: "InsertBuilder#firstColumn bundles Rails' keys.first + quote_column_name, so no adapter buildInsertSql can mirror rb:638-682"
status: closed
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Delivered on origin/main. (1) InsertBuilder#firstColumn() no longer exists: git grep -n firstColumn origin/main -- packages/activerecord/src returns only unrelated hits in relation/calculations.ts. (2) The MySQL adapter now mirrors abstract_mysql_adapter.rb:638-640 directly — abstract-mysql-adapter.ts:1111-1112 reads 'const [first] = insert.keys; const noOpColumn = first !== undefined ? this.quoteColumnName(first) : undefined', i.e. raw keys off the builder plus the dialect's own quote_column_name. (3) InsertBuilder now exposes 'readonly keys: Set<string>' (insert-all.ts:604) per insert_all.rb:228, and Builder#valuesList() exists (insert-all.ts:695). (4) The two baselined rows are gone with their whole shard: scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json does not exist in origin/main. Residual: into() still bundles the compiled VALUES list rather than Rails' two-fragment split — documented at insert-all.ts:588-592, a separate smaller item, not this story's premise."
---

## Context

Baselined in PR #6577 (RFC 0106 wave 3b): rows `build_insert_sql | first` and
`build_insert_sql | quote_column_name` in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`.

`abstract_mysql_adapter.rb:638-640`:

    def build_insert_sql(insert)
      # Can use any column as it will be assigned to itself.
      no_op_column = quote_column_name(insert.keys.first) if insert.keys.first

The adapter reads the builder's raw key list and quotes it ITSELF, through its
own `quote_column_name` — which is the dialect's, so each adapter quotes in its
own style.

trails' `InsertBuilder` contract (`insert-all.ts:585-595`) exposes no raw keys.
It offers `firstColumn(): string | undefined` (`:789-792`), which does the
`keys.first` AND the quoting internally:

    firstColumn(): string | undefined {
      const [first] = this._insertAll.keys;
      return first === undefined ? undefined : this.quoteColumn(first);
    }

So the MySQL adapter body reads `insert.firstColumn()` and never calls
`quoteColumnName`. Two Rails calls collapse into one builder method that Rails
does not have. The same bundling shows up in the `into()` contract, which the
interface docstring already flags as diverging from Rails' two-fragment
`"INSERT #{insert.into} #{insert.values_list}"` split.

This is a contract-shape divergence in `insert-all.ts`, not a MySQL bug — the
emitted SQL is correct today. It is filed because it makes the adapter bodies
unable to mirror Rails, and it will block
`mysql-build-insert-sql-missing-raw-alias-arm` (RFC 0106), whose raw-alias arm
needs `insert.model.table_name` and `insert.values_list` as SEPARATE fragments.

## Converged shape

`InsertBuilder` exposes what Rails' `InsertAll::Builder` exposes — at minimum
the raw `keys` and a separate `valuesList()` alongside `into()` — and the
adapter bodies do their own `quote_column_name` / `quote_table_name`, as
rb:638-682 does. `firstColumn()` and the bundled `into()` are retired once the
three dialect bodies stop depending on them.

Check PG and SQLite `buildInsertSql` before changing the interface: both consume
the same contract.

## Acceptance criteria

- [ ] `InsertBuilder` exposes Rails' fragment set; `firstColumn()` and the
      bundled `into()` are gone or reduced to Rails' own methods.
- [ ] MySQL, PG and SQLite `buildInsertSql` bodies each call
      `quoteColumnName` / `quoteTableName` where Rails does.
- [ ] The two `build_insert_sql` rows deleted by hand from the shard (no reseed).
- [ ] Emitted INSERT/UPSERT SQL is byte-identical to today's on all three
      dialects (this is a refactor, not a behaviour change).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
