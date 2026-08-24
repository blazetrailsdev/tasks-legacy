---
title: "Port Promise::Complete and close the @async arms that drop it (ids' loaded? arm)"
status: ready
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Relation#ids` closes its `loaded?` arm with

```ruby
return @async ? Promise::Complete.new(result) : result
```

(`activerecord/lib/active_record/relation/calculations.rb:382`). trails' port
(`packages/activerecord/src/relation.ts`, `async ids()`, landed in PR 6565)
returns the bare `result` — `asyncIds()` is `return this.ids()`, so the `@async`
arm has no counterpart at all and `Promise::Complete` is unmodelled.

This is not only an `ids` gap. `Promise::Complete` is Rails' "already-resolved"
`ActiveRecord::Promise` (`activerecord/lib/active_record/promise.rb`), the value
every `async_*` reader hands back when the work is already done, and it carries
real surface a native JS promise does not: `#value`, `#pending?`, `#then`
chaining onto an `ActiveRecord::Promise` rather than a thenable, and `#inspect`
rendering as `#<ActiveRecord::Promise ...>`.

The gap has already cost a shape in a shipped body. Because trails' `ids` records
no constructor in the `loaded?` arm, the `has_include?` arm below could not use
the obvious `[...new Set([...])]` dedupe that the sibling `pluck` arm uses: with
Rails recording `new` (the `Promise::Complete.new`) BEFORE `has_include?`, a
`new Set` in the second arm becomes the ported body's FIRST constructor and lands
after `hasInclude`, which `pnpm parity:api:calls` flags as
`relation.ts ids order:hasInclude,constructor`. The shipped body works around it
with `.filter((spec, i, all) => all.indexOf(spec) === i)` and a call-site comment.
Modelling `Promise::Complete` puts the constructor back where Rails has it and
retires that workaround.

## Converged shape

Port `ActiveRecord::Promise` / `Promise::Complete` (`promise.rb`), then close the
`@async` arms that currently drop it — starting with
`calculations.rb:382` (`ids`) and the `async_*` readers that return an
already-materialized value. `ids`' `loaded?` arm ends
`return this._async ? new Promise_.Complete(result) : result`, and the
`has_include?` arm's dedupe reverts to `[...new Set([...])]` to match `pluck`.

## Acceptance criteria

- [ ] `Promise::Complete` exists at the Rails name with `value` / `pending?` /
      `then` / `inspect`.
- [ ] `ids`' `loaded?` arm returns it under `@async`, mirroring
      `calculations.rb:374-383`.
- [ ] The `has_include?` arm's `.filter`/`indexOf` dedupe reverts to
      `new Set` and `pnpm parity:api:calls` stays green with no new row.
- [ ] The call-site comment explaining the `.filter` workaround is deleted.
