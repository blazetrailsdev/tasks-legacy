---
title: "test-databases.ts: eachDatabase claims a Rails counterpart that does not exist"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/test-databases.ts:16` exports `eachDatabase` with the
JSDoc `Mirrors: ActiveRecord::TestDatabases.each_database`. That method does not
exist. `vendor/rails/activerecord/lib/active_record/test_databases.rb:5-23` is
the whole module, and it defines exactly one method —
`self.create_and_load_schema(i, env_name:)` (`:11`) — plus the
`Parallelization.after_fork_hook` that calls it (`:7-9`).

```ts
export async function eachDatabase(
  adapters: DatabaseAdapter[],
  callback: (adapter: DatabaseAdapter, index: number) => void | Promise<void>,
): Promise<void> {
  for (let i = 0; i < adapters.length; i++) {
    await callback(adapters[i], i);
  }
}
```

The body is a bare indexed `for` loop over an array — it carries no
ActiveRecord behaviour at all. `pnpm parity:api:extra --package activerecord`
reports it as the one remaining novel name in `test-databases.ts`.

Its sibling `createAndMigrate` was removed for exactly this reason in PR #6982
(same file, same false `Mirrors:` claim, no caller outside its own tests), which
dropped the file from 2 novel names to 1. This story finishes that file.

Note the `Mirrors:` line is the actively harmful part: it asserts a Rails
counterpart that a reader cannot check without opening `test_databases.rb`, and
it is what kept the function looking legitimate through several passes.

## Converged shape

`eachDatabase` is deleted along with its test, and every caller inlines the loop
(or, where the caller is really iterating adapters, uses a plain `for...of`).
`packages/activerecord/src/test-databases.ts` is then exactly
`test_databases.rb`'s surface: `createAndLoadSchema` and nothing else.

Check callers first — at the time of writing the only references are in
`packages/activerecord/src/test-databases.test.ts`.

## Acceptance criteria

- [ ] `eachDatabase` is gone from `packages/activerecord/src/test-databases.ts`.
- [ ] No caller regresses; each inlines the iteration it needs.
- [ ] `pnpm parity:api:extra --package activerecord` reports 0 novel names for
      `test-databases.ts`.
- [ ] `packages/activerecord/src/test-databases.test.ts` keeps its remaining
      test names verbatim and passes.
