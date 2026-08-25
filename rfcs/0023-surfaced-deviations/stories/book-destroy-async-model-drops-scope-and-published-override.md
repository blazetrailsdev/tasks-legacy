---
title: "BookDestroyAsyncWithScopedTags drops Rails' has_many scope; BookDestroyAsync drops published!"
status: draft
updated: 2026-08-20
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

# `BookDestroyAsync` / `BookDestroyAsyncWithScopedTags` drop Rails' association scope and `published!` override

## Context

Found while reading the canonical models for
`destroy-async-test-port-and-model-flip` (PR #6764's bundle; that story was
closed on 2026-08-21 when ActiveJob was descoped, and its work now lives in
`port-destroy-association-async-test-and-flip-models` — but these two gaps are
independent of it and are pure model-fidelity misses).

`vendor/rails/activerecord/test/models/book_destroy_async.rb`:

```ruby
class BookDestroyAsync < ActiveRecord::Base
  self.table_name = "books"
  ...
  enum :status, [:proposed, :written, :published]

  def published!
    super
    "do publish work..."
  end
end

class BookDestroyAsyncWithScopedTags < ActiveRecord::Base
  self.table_name = "books"

  has_many :taggings, as: :taggable, class_name: "Tagging"
  has_many :tags, -> { where name: "Der be rum" }, through: :taggings, dependent: :destroy_async
end
```

`packages/activerecord/src/test-helpers/models/book-destroy-async.ts` drops
both:

- `BookDestroyAsyncWithScopedTags.hasMany("tags", { through: "taggings", ... })`
  carries **no scope**. Rails' `-> { where name: "Der be rum" }` is the entire
  point of the model — `destroy_association_async_test.rb`'s test
  "destroying a scoped has_many through only deletes within the scope deleted"
  asserts exactly one of two tags is destroyed. Without the scope the model is
  indistinguishable from `BookDestroyAsync` and the test it exists for cannot
  be ported correctly.
- `BookDestroyAsync#published!` (the enum bang override calling `super` and
  returning `"do publish work..."`) is absent. trails declares
  `publishedBang: () => Promise<true | undefined>` from the plain `enum`
  generation with no override.

The `dependent:` values are deliberately left alone here — flipping those to
`"destroyAsync"` is `port-destroy-association-async-test-and-flip-models`'s job
(the ActiveJob-dependent ActiveRecord work RFC, tasks PR #75) and is blocked on
trails having ActiveJob; this story is only the scope and the override, which
are correct to add regardless of which `dependent:` arm the models end up on.

(Repointed 2026-08-21: this previously named `destroy-async-test-port-and-model-flip`
under RFC 0106, closed on 2026-08-21 when ActiveJob was descoped.)

## Converged shape

- Give `BookDestroyAsyncWithScopedTags`'s `tags` association the reflection
  scope Rails gives it (`where({ name: "Der be rum" })`), in trails' settled
  scope-argument spelling for `hasMany`.
- Add the `publishedBang` override that calls the generated bang and returns
  `"do publish work..."`, matching `published!`.

## Acceptance criteria

- [ ] `book-destroy-async.ts` carries the association scope and the
      `published!` override at the same places Rails does.
- [ ] No canonical schema/fixture changes (both models are on the existing
      `books` / `taggings` / `tags` tables).
- [ ] Green on SQLite, PostgreSQL and MySQL/MariaDB.
