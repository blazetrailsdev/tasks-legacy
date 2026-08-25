---
title: "Drop the nil-name cast from reflection.test.ts once #4973 lands"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 10
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Reflection.create` now admits a nil `name` (PR #4975), so the Rails-form call
in `test_reflection_klass_for_nested_class_name`
(`vendor/rails/activerecord/test/cases/reflection_test.rb:126`) no longer needs
a cast:

```ruby
ActiveRecord::Reflection.create(:has_many, nil, nil, { class_name: "..." }, Customer)
```

PR #4973 rewrites that test into the Rails form and carries a
`null as unknown as string` cast plus an explanatory comment, because it was
authored against a `main` where `create` still required `string`. #4975 removed
the need for that cast but could not remove the cast itself — the two PRs were
open concurrently and editing the same test body would have conflicted.

This was an explicit acceptance criterion of `reflection-create-accepts-nil-name`
that had to be deferred; see that story's PR description.

## Acceptance criteria

- The `null as unknown as string` cast and its explanatory comment are removed
  from `reflection klass for nested class name` in
  `packages/activerecord/src/reflection.test.ts`.
- The call passes `null` directly and still typechecks.
- Test name unchanged; reflection suites stay green.

## Notes

PR #4973 merged before #4975, so this is **immediately actionable** — no blocker.
Confirmed present on `main` at
`packages/activerecord/src/reflection.test.ts:524`:

```ts
const anonymous = null as unknown as string;
```

`create` on `main` already accepts `string | null`, so the local and its cast
can be dropped and `null` passed directly.
