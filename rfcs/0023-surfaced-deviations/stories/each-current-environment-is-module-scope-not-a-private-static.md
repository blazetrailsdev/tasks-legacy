---
title: "each_current_environment is a module-level export, not a DatabaseTasks private"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "trailties"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #6185 while converging the `tasks/database-tasks.ts` unrouted-
privates cluster. Seven of the eight privates in that cluster moved onto the
`DatabaseTasks` class at their Rails names; `each_current_environment` is the
one that did not, because it has an external consumer.

Rails, `vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:592-596`:

```ruby
private
  def each_current_environment(environment, &block)
    environments = [environment]
    environments << "test" if environment == "development" && !ENV["SKIP_TEST_DATABASE"] && !ENV["DATABASE_URL"]
    environments.each(&block)
  end
```

It is a **private** class method on `DatabaseTasks`. Ours is a module-level
`export function eachCurrentEnvironment(environment: string): string[]` at the
bottom of `packages/activerecord/src/tasks/database-tasks.ts`, re-exported from
`packages/activerecord/src/index.ts:307` and imported by
`packages/trailties/src/commands/db.ts:23` (used at :906).

The body itself is faithful — this is purely a placement/visibility deviation,
and it is the last member of the cluster still sitting at module scope. It
leaves `eachCurrentConfiguration` (now a private static, `:582-590`) calling a
free function for its inner loop, which reads oddly next to the Ruby.

## Converged shape

Move it to `private static eachCurrentEnvironment(environment: string): string[]`
on `DatabaseTasks`, beside `eachCurrentConfiguration`. The blocker is the
`trailties` caller: `db.ts:906` maps over the returned array to build per-env
work. Rails' equivalent trailtie code would call a DatabaseTasks entry point
that is itself public rather than reaching for the private, so the convergence
is to find (or port) the public Rails method `db.ts:906` should really be
using — check `railties/lib/rails/tasks/` and `databases.rake` for what drives
the same loop upstream — and drop the `index.ts:307` export.

## Acceptance criteria

- `eachCurrentEnvironment` is a private static on `DatabaseTasks`.
- The `index.ts` re-export is deleted and `trailties/src/commands/db.ts` reaches
  the same behaviour through whatever public entry point Rails' rake task uses.
- `pnpm parity:api:extra --package activerecord` does not gain a novel name.
- `pnpm parity:api:calls` stays green.
