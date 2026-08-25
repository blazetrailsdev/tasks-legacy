---
title: "Converge PreloaderTest's essay-category assignment and class-name reads onto Rails' form"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two call sites in `PreloaderTest` (`packages/activerecord/src/associations.test.ts`,
landed by PR #5618) spell Rails' body in a trails-specific way that is not
needed — the faithful form works.

**1. `preload with only some records available with through associations`**

Rails assigns the belongs_to by record
(`vendor/rails/activerecord/test/cases/associations_test.rb:1345`):

```ruby
mary_essay.update!(category: mary_category)
```

The port writes the foreign key by hand instead:

```ts
await (maryEssay as any).update({ category_id: (tech as any).name });
```

The deviation was assumed, not verified. A probe confirms
`await essay.update({ category: tech })` works in trails today (Essay
`belongs_to :category, primary_key: :name`, and the setter dispatch resolves the
association writer), so the hand-written foreign key is a gratuitous divergence
that also hides the `primary_key: :name` behaviour Rails' form exercises.

**2. `multi database polymorphic preload with same table name`**

Rails reads the class name off the record
(`associations_test.rb:1240`, `:1245`):

```ruby
dog_comment.origin_type = dog.class.name
```

The port uses `dog.constructor.name`. That resolves correctly under vitest but
is exposed to bundler class renaming — the hazard already recorded for canonical
model imports under esbuild. Prefer whatever trails' stable model-name reader is
(the registered model name), not the JS constructor identifier.

## Acceptance criteria

- The essay/category assignment uses the belongs_to association form
  (`update({ category: … })`), matching `associations_test.rb:1345`, with no
  hand-written `category_id`.
- `origin_type` is derived from a rename-stable model-name reader rather than
  `constructor.name`, in both places the test sets it.
- `PreloaderTest` stays green on every lane; parity:test
  assertion-count/kind counts for `associations_test.rb` do not regress
  (post-#5618 baseline: 37 count, 89 kind).
