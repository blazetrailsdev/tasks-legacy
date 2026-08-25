---
title: "Converge TopicWithEvent: drop the class-level uniqueness declaration"
status: ready
updated: 2026-07-27
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

`TopicWithEvent` in trails declares a uniqueness validation on the class:

```ts
// packages/activerecord/src/validations/uniqueness-validation.test.ts:1009
class TopicWithEvent extends Topic {
  static {
    this.belongsTo("event", { foreignKey: "parent_id" });
    this.validatesUniqueness("event"); // <- not in Rails
  }
}
```

Rails' model carries only the association
(`vendor/rails/activerecord/test/models/topic.rb`):

```ruby
class TopicWithEvent < Topic
  belongs_to :event, foreign_key: :parent_id
end
```

Rails' `test_uniqueness_on_relation`
(`vendor/rails/activerecord/test/cases/validations/uniqueness_validation_test.rb:766`)
declares the validator inside the test and clears it in an `ensure` block.

Our test cannot follow that shape: redeclaring runs the validator twice (#4986
saw exactly this — `2 instead of 1 queries were executed`), and clearing it
would strip the class-level declaration for every later test in the file. #4986
worked around it by neither declaring nor clearing, with a comment in place.

The workaround is sound but the underlying model still diverges, so the test no
longer exercises Rails' declare/clear lifecycle.

## Acceptance criteria

- `TopicWithEvent` matches Rails: association only, no class-level
  `validatesUniqueness`.
- `uniqueness on relation` in `uniqueness-validation.test.ts` is restored to
  Rails' shape — declares the validator in the test, clears it in the `finally`
  — and its `assertNoQueries` / `assertQueriesCount(1)` assertions still hold.
- Any other test relying on `TopicWithEvent`'s implicit validator is updated;
  check `TopicWithUniqEvent` (which declares via `validates(..., uniqueness: true)`)
  is unaffected.
