---
title: "Table#columnExists/foreignKeyExists/checkConstraintExists drop Rails' trailing options"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: 7029
claim: "2026-08-25T12:46:55Z"
assignee: "converge-reverse-merge-bang-key-presence"
blocked-by: null
closed-reason: null
---

## Context

Surfaced converging the empty-`**options` splat across the `Table` forwarders
in PR #7019. That PR fixed the _arity when empty_; this story is the separate
gap that three of the exists-predicates cannot express Rails' arguments **at
all**.

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb`:

```ruby
def column_exists?(column_name, type = nil, **options)   # :744-746
  @base.column_exists?(name, column_name, type, **options)
end

def foreign_key_exists?(*args, **options)                # :920-922
  @base.foreign_key_exists?(name, *args, **options)
end

def check_constraint_exists?(*args, **options)           # :949-951
  @base.check_constraint_exists?(name, *args, **options)
end
```

trails
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts`,
`class Table`) drops the trailing options from all three:

```ts
async columnExists(columnName: string, type?: ColumnType): Promise<boolean> {
  return this._schema.columnExists(this.name, columnName, type);
}

async foreignKeyExists(toTableOrOptions?: string | Record<string, unknown>): Promise<boolean> { ... }

async checkConstraintExists(options: { name?: string; expression?: string } = {}): Promise<boolean> { ... }
```

So `t.column_exists?(:name, :string, limit: 80)` has no TS spelling at all;
`t.foreign_key_exists?(:authors, column: :author_id)` and
`t.check_constraint_exists?("price > 0", name: "price_check")` can pass the
expression **or** the options, never both — the same two-armed collapse that
`Table#removeCheckConstraint` had before PR #7019 converged it.

`SchemaStatements#columnExists` / `#foreignKeyExists` / `#checkConstraintExists`
already accept the fuller shape downstream, so this is a forwarder-only gap —
the same one-sided fix `removeCheckConstraint` took.

`index_exists?` (:768-770) is already correct and needs no change.

## Converged shape

Each of the three forwards Rails' positional args **and** the trailing options,
omitting the options argument only when it is empty — matching the guard the
sibling forwarders carry after PR #7019:

```ts
async columnExists(
  columnName: string,
  type?: ColumnType,
  options: Record<string, unknown> = {},
): Promise<boolean>
```

## Acceptance criteria

- `Table#columnExists`, `#foreignKeyExists` and `#checkConstraintExists` accept
  and forward Rails' arguments, with the trailing options omitted when empty.
- An expression/table plus options reaches the schema method as **both**, not
  one or the other.
- A `command-recorder.trails.test.ts` case (or the existing forwarder group)
  pins the recorded arity for at least the `columnExists` three-argument form.
- `parity:api:calls` / `:calls:args` non-negative; green on sqlite3, PostgreSQL
  and MySQL.
