---
title: "SetupAndTeardown failures is a module global, not per-test storage"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`setup_and_teardown.rb:44-53` appends a raising teardown callback to
`self.failures` — the failure list on the Minitest instance, which lives for
exactly one test:

```ruby
def after_teardown
  begin
    run_callbacks :teardown
  rescue => e
    self.failures << Minitest::UnexpectedError.new(e)
  rescue Minitest::Assertion => e
    self.failures << e
  end
  super
end
```

trails' port (`packages/activesupport/src/testing/setup-and-teardown.ts`, PR #6516)
has no per-test receiver, so `failures` is a module-global array, the
same stand-in `testing/tagged_logging.rb`'s `@tagged_logger` takes. The
consumer, `TestCase.afterTeardown` (`test-case.ts`), splices off everything
recorded during its own call and re-raises, so a single sequential test is
correct — but the storage is shared across every test in the worker, so a
concurrent or nested `after_teardown` could attribute another test's failure.

## Converged shape

Give the failure list per-test storage keyed off the running test rather than a
module global — vitest's per-test context (the same `context.task` `_runningTest`
already reads) is the natural key, or `AsyncLocalStorage`-style isolation if the
package already has one for this. Keep the Rails name `failures` and keep
`after_teardown`'s body reading it as a plain list.

## Acceptance criteria

- Two tests whose teardowns raise cannot see each other's failure.
- `setup_and_teardown.rb`'s body is unchanged in shape; only the storage moves.
