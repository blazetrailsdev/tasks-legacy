---
title: "Table forwarders pass no trailing options when empty, as Ruby's **options does"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: 16
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6175, which ported `test_invert_change_table`
(`vendor/rails/activerecord/test/cases/migration/command_recorder_test.rb:94`).
That case asserts the recorded command is `[:remove_column, [:fruits, :name,
:string], nil]` — **three** args, no trailing options hash.

Every forwarder on Ruby's `Table`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb`,
`class Table`) ends in a `**options` splat, and `**{}` passes no argument at
all:

```ruby
def column(column_name, type, index: nil, **options)   # :730
  @base.add_column(name, column_name, type, **options)  # :732
end
def index(column_name, **options);   @base.add_index(name, column_name, **options)   end   # :756
def timestamps(**options);           @base.add_timestamps(name, **options)           end   # :786
def remove(*column_names, **options); @base.remove_columns(name, *column_names, **options) end # :829
def remove_index(column_name = nil, **options); @base.remove_index(name, column_name, **options) end # :842
def remove_timestamps(**options);    @base.remove_timestamps(name, **options)        end   # :852
def references(*args, **options);    @base.add_reference(name, ref_name, **options)  end   # :871
def remove_references(*args, **options); @base.remove_reference(name, ref_name, **options) end # :885
def foreign_key(*args, **options);   @base.add_foreign_key(name, *args, **options)   end   # :899
def remove_foreign_key(*args, **options); @base.remove_foreign_key(name, *args, **options) end # :910
def check_constraint(*args, **options);  @base.add_check_constraint(name, *args, **options) end
def remove_check_constraint(*args, **options); @base.remove_check_constraint(name, *args, **options) end
```

The TS `Table` in
`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts`
(~:1560-1640) passes its (defaulted-to-`{}`) options object unconditionally in
every one of these. That is invisible against a real adapter, which ignores an
empty hash — but `CommandRecorder#record` stores the arguments **verbatim**
(`command_recorder.rb`), so under `change_table` inside a `revert` the recorded
command grows a trailing `{}` that Rails does not have, and any test or
`replay` that compares command tuples sees the wrong arity.

PR #6175 converged `column` only (guarding the call with
`Object.keys(colOpts).length === 0`), because that was the one its ported test
exercised. The sibling forwarders are unconverged.

## Converged shape

Each forwarder omits the trailing options argument when the object it would
pass has no own keys, exactly as `**{}` does — the same guard `column` now
carries. Prefer factoring it the way Rails does (it does not factor: the splat
is language-level), i.e. repeat the guard rather than introducing a helper
`Table` does not have.

## Acceptance criteria

- Every `Table` forwarder listed above omits the trailing options argument
  when empty.
- A `command-recorder.test.ts` case (or the existing `invert change table`
  group) covers at least the `remove` / `index` / `timestamps` paths recording
  Rails' arity.
- `parity:api:calls` non-negative; green on sqlite3, PostgreSQL and MySQL.
