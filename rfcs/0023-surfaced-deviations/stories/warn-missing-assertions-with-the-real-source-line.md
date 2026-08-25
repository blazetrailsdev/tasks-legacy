---
title: "tests_without_assertions warns with file:0 — populate the real source line"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`tests_without_assertions.rb:13-14` prints the failing test's source location:

```ruby
file, line = method(name).source_location
warn "Test is missing assertions: `#{name}` #{file}:#{line}"
```

trails' port (`packages/activesupport/src/testing/tests-without-assertions.ts`,
PR #6516) takes that pair as `RunningTest.sourceLocation`, which
`test-case.ts`'s `_runningTest` fills from vitest's task
(`task.file?.filepath`, `task.location?.line`). vitest only populates
`task.location` when the `includeTaskLocation` config option is on, so every
warning currently prints `<file>:0` — the file is right, the line is always
zero.

## Converged shape

Turn `includeTaskLocation` on in the shared vitest config (it is a collection
-time cost only) so `task.location.line` is populated, and assert the rendered
`file:line` in `setup-and-teardown.trails.test.ts`, which today asserts a
hardcoded pair. If the option turns out to cost measurable collection time
across the AR suite, the fallback is to resolve the line from the test's own
stack at warn time — but measure before reaching for it.

## Acceptance criteria

- A test with no assertions warns with the real line number, not `:0`.
- `setup-and-teardown.trails.test.ts` covers the rendered location.
