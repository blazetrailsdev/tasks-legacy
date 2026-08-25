---
title: "weak_etag?/strong_etag? return boolean where Rails returns nil"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
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

`packages/actionpack/src/action-dispatch/http/cache.ts` returns a strict
`boolean` from `isWeakEtag` and `isStrongEtag`. Rails returns `nil` or a boolean
from both, because each is an `&&` chain fronted by the value-returning `etag?`:

- `def weak_etag?; etag? && etag.start_with?('W/"'); end`
  (`vendor/rails/actionpack/lib/action_dispatch/http/cache.rb:132-134`) — with no
  ETag header this evaluates to `nil`, not `false`.
- `def strong_etag?; etag? && !weak_etag?; end` (`cache.rb:138-140`) — same.

PR #5637 converged `isEtag` itself to return `string | undefined` (matching
`cache.rb:129`) but deliberately kept these two narrowed to `boolean`: they are
declared `() => boolean` on `Response`
(`packages/actionpack/src/action-dispatch/http/response.ts:350-351`) and every
call site uses them as booleans, so widening them was out of that story's scope.

## Acceptance criteria

- Decide whether the `nil` return is worth reproducing. If yes, widen both to
  Rails' union and update the `Response` `declare`s plus every call site; if no,
  record the narrowing as an accepted deviation at the call site per
  `feedback_justify_deviations_at_call_site_not_pr_body`.
- Any behavior change is covered by a test that fails on baseline, mirroring
  Rails' own cases — do not invent test names.
- `pnpm exec tsx scripts/api-compare/lint-call-mismatches-wide.ts` stays clean.
