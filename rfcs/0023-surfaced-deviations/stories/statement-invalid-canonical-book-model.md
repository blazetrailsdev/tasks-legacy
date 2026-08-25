---
title: "statement_invalid port declares an ad-hoc Book instead of the canonical model"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/statement-invalid.test.ts` declares an ad-hoc model
instead of using the canonical one:

```ts
class Book extends Base {
  static override _tableName = "books";
  static {
    this.attribute("author_id", "integer");
    this.attribute("cover", "string");
  }
}
```

Rails' `activerecord/test/cases/statement_invalid_test.rb:4-8` does
`require "models/book"` and `fixtures :books`, i.e. the canonical `Book` model
(`vendor/rails/activerecord/test/models/book.rb`, ours at
`packages/activerecord/src/test-helpers/models/`). The table is the canonical
`books`, so only the model is invented.

This matters beyond tidiness: declaring `attribute()` explicitly suppresses DB
reflection for those columns, so the ad-hoc model exercises a different
attribute-resolution path from every other `books` test, and it silently
diverges from the canonical `Book` as that model grows (enums, scopes,
`status` defaults).

The test's `fixtures({}, { useTransactionalTests: false })` call also loads no
fixture set, where Rails declares `fixtures :books`.

## Acceptance criteria

- The test imports the canonical `Book` from
  `packages/activerecord/src/test-helpers/models/` and drops the local class and
  its `attribute()` declarations plus the `Book.loadSchema()` `beforeAll`.
- `fixtures({ books: ... })` mirrors `statement_invalid_test.rb:9`'s
  `fixtures :books`.
- Both tests keep the shape PR #6736 landed (`assertRaises` +
  `assertNot` / two `toEqual`s) and the file stays at 0/0/0 in
  `pnpm parity:test -- --package activerecord --assertions`.
