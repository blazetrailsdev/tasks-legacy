---
title: "load() calls toArray() where Rails' to_ary calls records calls load"
status: ready
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' load chain runs `to_ary` → `records` → `load` → `exec_queries`:

```ruby
# activerecord/lib/active_record/relation.rb:337-339
def to_ary
  records.dup
end
alias to_a to_ary

# :342-345
def records
  load
  @records
end

# :1179-1186
def load(&block)
  if !loaded? || scheduled?
    @records = exec_queries(&block)
    @loaded = true
  end
  self
end
```

trails runs it backwards: `load()` calls `toArray()`
(`packages/activerecord/src/relation.ts`, the `load()` body), and `toArray`
carries the null-relation guard, the `withConnection` wrapper, the load
token and the `loadRecords` assignment.

PR #6905 converged the readers onto the `loaded?` / `records` seams and gave
`load()` Rails' `loaded?` guard, which is what keeps `toArray`'s two
`await this.records()` calls from re-entering. But the chain itself is still
inverted, and that inversion is why `AssociationRelation` and
`DisableJoinsAssociationRelation` hang their overrides off `toArray` where
Rails hangs them off `exec_queries` / `load`.

## Converged shape

```ts
// to_ary (relation.rb:337-339)
async toArray(): Promise<T[]> {
  return [...(await this.records())];
}

// load (relation.rb:1179-1186)
async load(): Promise<LoadedRelation<this>> {
  if (this.isNullRelation()) return stripThenable(this);
  if (!this.isLoaded) {
    const token = this._loadToken;
    const records = await this.withConnection(() => this.execQueries());
    if (token === this._loadToken) this.loadRecords(records);
  }
  return stripThenable(this);
}
```

`records()` already has Rails' shape (`load; @records`) and needs no change.
Keep the `_loadToken` guard — it covers the trails-only mid-await reset race
Rails cannot have because `exec_queries` is synchronous.

## Dependencies

Must land AFTER both override moves, or the inversion silently routes around
them:

- `converge-association-relation-inverse-wiring-onto-exec-queries`
- `converge-disable-joins-association-relation-onto-load`

## Acceptance criteria

- `toArray` is `records.dup` and nothing else.
- `load` carries the null-relation guard, the query, and the assignment.
- The `isNullRelation()` chokepoint still runs on every load attempt (the
  stale new-owner seed rebase depends on it).
- `pnpm parity:api:calls` / `:args` add zero rows; relation.rb -> relation.ts
  stays 401/401.
