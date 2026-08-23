---
title: "Converge benchmark's level dispatch onto Rails' public_send (drop the not-a-function guard)"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: ["activesupport"]
deps: []
deps-rfc: []
est-loc: 60
priority: 6
pr: 6931
claim: "2026-08-23T17:44:08Z"
assignee: "encryption-schemes-test-lacks-transactional-fixtures"
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::Benchmarkable#benchmark` dispatches the log line with
`logger.public_send(options[:level], "%s (%.1fms)" % [ message, ms ])`
(`vendor/rails/activesupport/lib/active_support/benchmarkable.rb:47`). Ruby
duck-types the logger: if it does not answer to that level, `public_send`
raises `NoMethodError`.

trails' `packages/activesupport/src/benchmarkable.ts` instead reads the level
method off the logger and silently skips the log line when it is not a
function:

```ts
const write = (logger as Record<string, unknown>)[options.level!];
if (typeof write === "function") { ... }
```

The guard exists only because `BenchmarkLogger` (same file) declares every
level method optional (`debug?`, `info?`, `warn?`, `error?`, `fatal?`), which
is the trails spelling of Ruby's duck-typed logger. Flagged on PR #6870 as a
non-blocking finding: a caller reaching `benchmark` through `as any` with a
logger missing the requested level gets silence where Rails raises.

`ActiveSupport::Logger` answers to all five levels, and `BenchmarkOptions.level`
is already narrowed to the four `benchmark` can request, so nothing in-repo
needs the optionality.

## Converged shape

Make `debug`/`info`/`warn`/`error`/`fatal` required on `BenchmarkLogger` and
drop the `typeof write === "function"` guard, so the call site is the direct
dispatch `benchmarkable.rb:47` makes and a logger missing the level fails
loudly. Check the actionpack and ActiveRecord hosts (`AbstractController::Logger`,
`ActiveRecord::Base.logger`) still satisfy the narrowed type; one trails test,
`"tolerates a logger whose \`info\` is not a function"`in`packages/actionpack/src/abstract-controller/logger.test.ts`, asserts the
current silence and goes with the guard.

## Acceptance criteria

- [ ] `benchmark` dispatches the level method directly, mirroring
      `benchmarkable.rb:47`; no `typeof` guard around it.
- [ ] `BenchmarkLogger`'s level methods are required.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows.
