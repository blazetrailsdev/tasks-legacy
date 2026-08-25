---
title: "HABTM middle has_many passes an owner foreign key Rails derives"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Builder::HasAndBelongsToMany#middle_options`
(`activerecord/lib/active_record/associations/builder/has_and_belongs_to_many.rb:69-75`)
sets `:foreign_key` ONLY when the declaration supplied one, and lets the middle
`has_many` derive the owner key from `lhs_model.name.demodulize` — so
`Publisher::Article` yields `article_id`.

trails cannot: a JS class name is flattened (`Publisher::Article` is the class
`PublisherArticle`), so demodulizing it yields `PublisherArticle` and the
derived key would be `publisher_article_id`. PR #6836 pushed that compensation
down to one seat —
`packages/activerecord/src/associations/builder/has-and-belongs-to-many.ts`
resolves the owner key in `throughModel`'s `add_left_association` off
`_demodulizedName ?? name`, and `middleOptions`' else-branch passes
`joinModel.leftReflection.foreignKey` on — but it is still an argument Rails
does not pass, in a method Rails leaves alone.

The same flattening drives the `left_side` belongs_to's `foreignKey:`, which
Rails also omits (it lets the belongs_to derive the never-read `left_side_id`).

## Converged shape

The Ruby leaf name is available to `demodulize` wherever a model's Rails name is
needed, so `middle_options` sets `:foreign_key` only under
`if options.key?(:foreign_key)` and `add_left_association` passes
`anonymous_class:` alone, as Rails does. `_demodulizedName` is the seat that
already carries the leaf name; the derivation reading `model.name` is what has
to move.

## Acceptance criteria

- [ ] `middleOptions` has no else-branch: `foreignKey` is set only when the
      declaration supplied one.
- [ ] `addLeftAssociation` is called with `anonymousClass` alone.
- [ ] The namespaced-owner HABTM tests still resolve `article_id`, not
      `publisher_article_id`.
