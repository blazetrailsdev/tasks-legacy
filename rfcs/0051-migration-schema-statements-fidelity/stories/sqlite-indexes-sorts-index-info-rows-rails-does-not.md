---
title: "SQLite3 indexes sorts PRAGMA index_info rows by seqno; Rails maps them in PRAGMA order"
status: draft
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 15
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `SQLite3::SchemaStatements#indexes` in #7044
(`sqlite-attached-schema-notion-has-no-rails-counterpart`), which removed that
body's ATTACHed-schema split. This sort survived the pass and is the last
non-Rails call left in the method.

Rails maps the `PRAGMA index_info` rows in the order the PRAGMA returns them,
with no sort:

```ruby
# activerecord/lib/active_record/connection_adapters/sqlite3/schema_statements.rb:27-29
columns = internal_exec_query("PRAGMA index_info(#{quote(row['name'])})", "SCHEMA").map do |col|
  col["name"]
end
```

trails adds one:

```ts
// packages/activerecord/src/connection-adapters/sqlite3/schema-statements.ts:131
const columnNames = cols.sort((a, b) => a.seqno - b.seqno).map((c) => c.name);
```

Being explicit about the disposition, since it decides how this is closed:
**this is a no-op in practice.** SQLite documents `PRAGMA index_info` as
returning one row per column _in index order_, keyed by `seqno`, so the sort
can never reorder anything. There is no bug to fix and no failing case to
write — the finding is that the port makes a call Rails does not make, which is
extra surface inside a body that is otherwise now a structural mirror.

Recording it rather than leaving it as review prose. It is legitimately small;
triage may well decide the belt-and-braces sort is not worth a PR of its own
and fold it into the next pass over this file.

## Converged shape

Drop the `.sort(...)`, leaving `cols.map((c) => c.name)`, so the body matches
`schema_statements.rb:27-29`. The `seqno` field then goes unused and comes out
of the row type annotation on `:130` with it.

Do **not** replace it with a different ordering guarantee — Rails' contract here
is "whatever the PRAGMA returned", and matching that is the whole point.

## Acceptance criteria

- [ ] `indexes` maps `PRAGMA index_info` rows in PRAGMA order with no sort,
      mirroring `sqlite3/schema_statements.rb:27-29`.
- [ ] The now-unused `seqno` is dropped from the row type.
- [ ] Multi-column and composite-PK index reflection still reports columns in
      index order (existing coverage in
      `sqlite3-introspection.trails.test.ts` should be sufficient — confirm
      rather than add).
- [ ] SQLite lane green.
