---
title: "134 tests use bespoke inline models; some cannot exercise their named Rails behaviour"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/associations/has-many-associations.test.ts` declares
**270 bespoke inline `class X extends Base` models across 134 of its 320
tests**. Beyond the canonical-models rule in CLAUDE.md, these carry a failure
mode that is worse than schema bloat: a bespoke model can quietly lack the
property the test's Rails name is about, so the test passes while exercising
nothing.

A confirmed instance, fixed in PR #6743:

`it("included in collection for composite keys")` mirrors Rails'
`test_included_in_collection_for_composite_keys`
(`vendor/rails/activerecord/test/cases/associations/has_many_associations_test.rb:2030-2035`):

```ruby
great_author = cpk_authors(:cpk_great_author)
book = great_author.books.first
assert great_author.books.include?(book)
```

The trails port declared `InclAuthor`/`InclPost` on `authors`/`posts` — both
**single**-primary-key — then asserted `posts.some((p) => p.id === post.id)`
against a raw `findHasManyTarget`. It never constructed a composite key and
never called `include?` at all. The test named after composite-key `include?`
could not fail for any composite-key reason.

That mattered: `CollectionAssociation#include?` was passing composite keys to
`scope.exists?` as a bare tuple rather than Rails'
`klass.primary_key.zip(record.id).to_h`
(`activerecord/lib/active_record/associations/collection_association.rb:266`),
which `buildWhereClause` rejects with an `ArgumentError`. The stub test held
green over that bug until #6743 rewrote it onto canonical `CpkAuthor`/`CpkBook`,
at which point it failed on the real defect.

The same class of hole plausibly hides elsewhere in the 134: any test whose
Rails name turns on a model property (composite PK, STI, polymorphic, counter
cache, `:through`, custom primary key) that its bespoke inline model does not
actually have.

## Converged shape

Audit the 134 tests and convert them to the canonical models in
`packages/activerecord/src/test-helpers/models/` with `fixtures({ ... })`,
prioritising those whose Rails name names a model property the inline model
lacks — those are the ones that can be silently vacuous, not merely bloated.
Test names stay verbatim (`parity:test` matches on them); only bodies change.

Because this is larger than one PR, land it in waves and file follow-on stories
rather than fanning out.

## Acceptance criteria

- An inventory of the 134 tests, each marked "vacuous" (cannot exercise its
  named behaviour) or "bloat-only" (exercises it, just with a bespoke model).
- Every **vacuous** test converted to canonical models and shown to fail against
  the pre-existing defect it should have caught, or explicitly noted as having
  no defect behind it.
- No test renamed; no new bespoke table or inline model added.
- Composite-PK conversions verified on PostgreSQL/MariaDB as well as SQLite —
  `cpk_books.id` is not auto-increment under its composite PK, so a null key
  part passes on SQLite and reds on the strict engines (this exact gap reded
  MariaDB on #6743 after the local SQLite run was green).
