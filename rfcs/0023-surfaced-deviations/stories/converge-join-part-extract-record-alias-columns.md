---
title: "JoinPart#extract_record takes an alias string where Rails takes column/alias pairs"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`JoinPart#extract_record` is ported structurally differently from Rails, and
the divergence hides behind a parameter that this PR (#6420) could only
half-converge.

Rails (`vendor/rails/activerecord/lib/active_record/associations/join_dependency/join_part.rb:47-67`):

```ruby
def extract_record(row, column_names_with_alias)
  # This code is performance critical as it is called per row.
  hash = {}
  index = 0
  length = column_names_with_alias.length
  while index < length
    column = column_names_with_alias[index]
    hash[column.name] = row[column.alias]
    index += 1
  end
  hash
end

def instantiate(row, aliases, column_types = {}, &block)
  base_klass.instantiate(extract_record(row, aliases), column_types, &block)
end
```

`column_names_with_alias` is an **array of column objects**, each carrying a
`name` (the model attribute) and an `alias` (the SELECT-list alias). The body
is an index loop that maps `row[column.alias]` → `hash[column.name]`.

trails (`packages/activerecord/src/associations/join-dependency/join-part.ts:118-151`)
instead takes a **single alias string** and reconstructs the record by string
matching: first a `^t(\d+)$` regex for the JoinDependency `t{n}_r{n}` form,
then a generic `${aliases}_` prefix scan over `Object.entries(row)`. There is
no column array and no `.name`/`.alias` pair anywhere on the path.

Consequences:

- Any attribute whose column name would collide with the prefix scan is
  resolved by string shape rather than by the alias map Rails builds.
- `extract_record`'s Rails parameter is `column_names_with_alias`, but the TS
  parameter cannot honestly carry that name while it holds a bare string.
  #6420 renamed it `aliases` — correct for `instantiate`'s argument
  (join_part.rb:65) and an improvement on the previous `columnAlias`, but
  `extractRecord`'s own parameter is still not at its Rails name. That last
  step is blocked on the structural fix, which is why it is filed here rather
  than done there.
- `column_types` — Rails' third `instantiate` parameter — has no TS
  counterpart on this path either.

Discovered during review of #6420 (RFC 0096 wave 2, naming-only), which
deliberately left it alone: converging the structure is a behavior-bearing
change well outside a rename PR.

## Acceptance criteria

- [ ] `extractRecord` takes the column/alias collection Rails passes, with the
      Rails parameter name `columnNamesWithAlias`, and maps
      `row[column.alias]` → `record[column.name]` — no regex on the alias and
      no `startsWith` prefix scan.
- [ ] The caller (`JoinPart#instantiate`, join_part.rb:65) passes that
      collection through under the Rails name `aliases`, and the
      `JoinDependency` side builds it rather than handing down a bare string.
- [ ] `instantiate`'s `columnTypes` parameter is ported or its absence is
      justified at the call site with a Rails cite.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` stay green and no
      baseline row is added; the eager-loading and join-dependency tests pass
      on all three adapters.
