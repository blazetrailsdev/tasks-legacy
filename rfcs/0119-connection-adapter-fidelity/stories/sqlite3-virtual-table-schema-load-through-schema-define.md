---
title: "sqlite3 virtual_table_test schema load: go through Schema.define"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converting the sqlite3 adapter sibling suites to the ambient
connection (PR #5499, RFC 0029).

`virtual_table_test.rb#test_schema_load` exercises the **schema-definition
path**, not the adapter method:

```ruby
def test_schema_load
  original, $stdout = $stdout, StringIO.new

  ActiveRecord::Schema.define do
    create_virtual_table :emails, :fts5, ["content", "meta UNINDEXED"]
  end

  assert @connection.virtual_table_exists?(:emails)
ensure
  $stdout = original
end
```

`packages/activerecord/src/adapters/sqlite3/virtual-table.test.ts` calls
`adapter.createVirtualTable("emails", ...)` directly, with a call-site note
saying it "mirrors Schema.define creating the table". It does not — it skips
`Schema.define` entirely, so the test named `schema load` never loads a schema
and would still pass if `Schema.define` lost `createVirtualTable` support.

The sibling `schema dump` case is fine: it goes through `SchemaDumper.dump`.

## Acceptance criteria

- [ ] `schema load` drives table creation through `Schema.define` (trails'
      `ActiveRecordSchema.define` equivalent), as Rails does.
- [ ] Rails silences `$stdout` around the define block; do the equivalent if
      trails' `Schema.define` writes to stdout, so the suite stays quiet.
- [ ] `virtualTableExists("emails")` assertion retained.
- [ ] The "mirrors Schema.define creating the table" comment is removed once it
      is true, or corrected if the direct call is kept for a stated reason.
- [ ] Test names unchanged.
