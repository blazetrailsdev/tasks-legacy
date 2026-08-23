---
title: "Remove or relocate the preprocessQuery re-entrancy guard Rails does not have"
status: draft
updated: 2026-07-29
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`preprocessQuery`
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:1927-1944`)
wraps the transformer loop in a `_inQueryTransformers` re-entrancy guard stored
on the adapter host: if the flag is already set, it clears it and returns `sql`
**without running any transformer at all**.

Rails has no such guard. `preprocess_query`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:575-587`)
does the write checks and then runs the loop unconditionally:

```ruby
ActiveRecord.query_transformers.each do |transformer|
  sql = transformer.call(sql, self)
end
```

The guard is a trails invention, surfaced (not introduced) while converging
`queryTransformers` onto the `ActiveRecord` accessor in PR #5565 — that PR
deliberately left it untouched as out of scope. Its observable effect is that a
transformer which itself issues a query (or any nested `preprocessQuery` call on
the same adapter) causes the _outer_ SQL to skip tagging entirely, and it leaves
the flag toggled in a way that depends on nesting depth. Whatever hazard it was
added for should be identified before removal — if it guards a real infinite
recursion (e.g. `QueryLogs` reading from the DB), the fix is likely at the
transformer, not in `preprocess_query`.

## Acceptance criteria

- Determine what the `_inQueryTransformers` guard was protecting against
  (git-blame the line; check whether any shipped transformer re-enters).
- If nothing re-enters, delete the guard so `preprocessQuery` matches Rails
  line-for-line; add a regression test that a nested `preprocessQuery` still
  tags the outer SQL.
- If a real recursion exists, move the protection to the offending transformer
  and record the reason at the call site per the deviation convention.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.

## Update after PR #6913 (2026-08-23)

The guard is now smaller and purer, which makes this story easier, not moot.

`_inQueryTransformers` used to carry TWO jobs: a one-shot "skip the transformer
pass" flag that `executeBatch` set per statement, and a re-entrancy guard.
PR #6913 routed `executeBatch` through `rawExecute` on the abstract base and
mysql2 (matching `abstract/database_statements.rb:594-598` and
`mysql2/database_statements.rb:17-21`), which is BELOW `preprocess_query` —
`preprocess_query` is `internal_execute`'s step (`:589-591`) — so batch SQL now
skips the transformers structurally, exactly as in Rails, and the batch arm was
deleted.

What remains in `preprocessQuery`
(`connection-adapters/abstract/database-statements.ts`) is only:

```ts
const host = this as DatabaseStatementsHost & { _inQueryTransformers?: boolean };
if (host._inQueryTransformers) return sql;
host._inQueryTransformers = true;
try { ...transformers... } finally { host._inQueryTransformers = false; }
```

Rails' `preprocess_query` (`abstract/database_statements.rb:574-584`) has **no
guard at all** — `check_if_write_query`, `mark_transaction_written_if_write`,
then the `ActiveRecord.query_transformers.each` loop, nothing else. So the
whole flag is now a single invented arm with one job, and the converged shape
is simply to delete it and the `_inQueryTransformers` field on
`DatabaseStatementsHost`.

The one thing to establish before deleting: what a synchronous, SQL-issuing
query transformer does without the guard. Rails tolerates it because nothing
re-enters synchronously there; confirm no trails transformer does either
(the flag is set and cleared within one synchronous stretch, so it never spans
an await).
