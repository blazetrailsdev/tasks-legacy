---
title: "scopeForCreate calls a trails-only _scopeAttributes wrapper where Rails inlines where_clause.to_h"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 40
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Relation#scopeForCreate` (`packages/activerecord/src/relation.ts`) opens by
calling a trails-invented private helper:

```ts
scopeForCreate(): Record<string, unknown> {
  const hash = this._scopeAttributes();
  ...
}
```

Rails inlines the expression instead — `activerecord/lib/active_record/relation.rb:1231-1235`:

```ruby
def scope_for_create
  hash = where_clause.to_h(model.table_name, equality_only: true)
  create_with_value.each { |k, v| hash[k.to_s] = v } unless create_with_value.empty?
  hash
end
```

`_scopeAttributes` has no Ruby counterpart (`protected _scopeAttributes()` in
relation.ts, `@internal`-tagged): it is exactly that one `where_clause.to_h`
call wrapped in a method Rails does not have, i.e. the "no extra abstraction"
rule (CLAUDE.md). Surfaced while converging the `empty?` call-set row for this
same method in PR #6660 — the guard and the `each` loop are now Rails-shaped,
the receiver of the first line is not.

## Converged shape

Inline the call, keeping the Rails local name `hash`:

```ts
const hash = this.whereClause.toH(this.model.tableName, { equalityOnly: true });
```

and delete `_scopeAttributes` once its other callers (grep — there is at least
one outside `scopeForCreate`) are inlined the same way, each against the Rails
line that spells the `to_h` call out.

## Acceptance criteria

- [ ] `scopeForCreate`'s first line is Rails' `where_clause.to_h(...)`
      expression, not a wrapper.
- [ ] `_scopeAttributes` deleted, or — if a remaining caller genuinely needs it
      — that caller converged too, so no trails-only wrapper survives.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow;
      `pnpm parity:api:calls` / `:args` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
