---
title: "encrypted-file.test.ts reads process.env directly instead of the adapter env"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 15
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/encrypted-file.test.ts:31` reads the environment
through the global directly:

```ts
originalEnv = process.env.CONTENT_KEY;
```

while the very next line writes through the adapter (`setEnv("CONTENT_KEY", undefined)`),
and the file imports `setEnv` from `./process-adapter.js` for exactly this. The
adapter already exports a readable snapshot for the read side —
`export const env` (`packages/activesupport/src/process-adapter.ts:56`) — so the
bare `process.env` has a drop-in replacement.

Pre-existing (not introduced by PR #6641, which touched the assertions in this
file and left the setup alone), but it violates the standing no-`process.*` rule
for package sources and it is the only such reference in the file.

Rails reads `ENV["CONTENT_KEY"]` (`vendor/rails/activesupport/test/encrypted_file_test.rb:9`);
the adapter's `env` is the trails spelling of `ENV`, so this is a pure
convergence with no behavioral question.

## Acceptance criteria

- `encrypted-file.test.ts` reads the env through `env` from `process-adapter.js`,
  with no `process.*` reference left in the file.
- The 15 `encrypted_file_test.rb` tests still pass and stay at 0 assertion
  mismatches.
