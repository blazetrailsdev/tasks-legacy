---
title: "assert_equal on AR records is Core#==, not a deep-equality walk"
status: draft
updated: 2026-08-25
rfc: "0082-ruby-ts-idiom-conversion-classes"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #7024 (RFC 0115 `retire-attribute-set-narrow-to`), which routed
`Base.instantiate` through `AttributeSet::Builder#build_from_database`. That
changed which internal fields a hydrated record carries — legitimately, and
matching Rails — and immediately reddened a test that had nothing to do with
the behaviour under test:

```text
packages/activerecord/src/relations.test.ts > RelationTest > loading with one association with non preload
AssertionError: expected { Object (_attributes, errors, ...) } to deeply equal { Object (_attributes, errors, ...) }
```

The two records had the same id and class. They differed only in the
`additionalTypes` map inside their `LazyAttributeSet` — index-keyed entries that
Rails carries too (`ActiveRecord::Result#column_type` looks up by index before
name, `activerecord/lib/active_record/result.rb`, which is why the PG adapter
keys them that way).

Rails' assertion cannot see any of that:

```ruby
# vendor/rails/activerecord/test/cases/relations_test.rb:768-772
def test_loading_with_one_association_with_non_preload
  posts = Post.eager_load(:last_comment).order("comments.id DESC")
  post = posts.find { |p| p.id == 1 }
  assert_equal Post.find(1).last_comment, post.last_comment
end
```

`assert_equal` on two AR records dispatches to `ActiveRecord::Core#==`
(`activerecord/lib/active_record/core.rb:632-638`), which is:

```ruby
def ==(comparison_object)
  super ||
    comparison_object.instance_of?(self.class) &&
    primary_key_values_present? &&
    comparison_object.id == id
end
```

— identity, then exact class plus id. Never a structural walk of the object.

A port that writes `expect(a).toEqual(b)` for that is strictly stronger than
Rails: it asserts over every internal field, so any legitimate change to a
record's internals (a new memo, a lazily-populated map, a builder swap) reds a
test whose name and Rails counterpart are about association loading. The one
instance found was converged in #7024 to `expect(a.equals(b)).toBe(true)` —
`equals` is trails' port of `Core#==` at
`packages/activerecord/src/core.ts:160`. The assertion kind is unchanged for
`parity:test` purposes (`toBe` and `toEqual` both normalize to `equal`).

This is an idiom-conversion class, not a one-off: Ruby's `assert_equal` uses the
receiver's `==`, and JS `toEqual` is structural, so every AR-record comparison
ported as `toEqual`/`toStrictEqual` is a latent false failure. Related classes
already tracked here: [[track-getter-vs-method-shape]],
[[track-ruby-truthiness-residuals]].

## Converged shape

Wherever a Rails test asserts `assert_equal`/`assert_not_equal` on two
ActiveRecord _records_ (not attribute hashes, not scalars), the port compares
through `Core#==` — `expect(a.equals(b)).toBe(true)` — rather than deep-equality
on the objects. Collections of records compare element-wise the same way, which
is what Rails' `assert_equal [rec], relation` does via `Array#==` → element
`==`.

Deep `toEqual` stays correct where Rails is comparing something that really is
a value: attribute hashes, `to_a` of scalars, serialized payloads.

## Acceptance criteria

- Sweep AR test files for `toEqual`/`toStrictEqual` whose operands are AR record
  instances, and convert those to `Core#==` comparisons via `equals`.
- No test is renamed or reworded — names are how `parity:test` matches.
- `pnpm parity:test:assertions` stays green and the assertion-kind counts do not
  regress (both spellings normalize to `equal`, so converged rows should be
  neutral).
- Note in the story when done whether any converted assertion was load-bearing
  — i.e. caught a real bug that `Core#==` would miss — so the class can be
  narrowed rather than applied blindly.
