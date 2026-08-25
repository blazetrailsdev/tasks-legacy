---
title: "InsertAll#ensure_valid_options_for_connection! ports only 1 of Rails' 4 guards"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing #7053 (`insert-all-constructor-tail-deferred-to-execute`).

Rails' `InsertAll#ensure_valid_options_for_connection!`
(`vendor/rails/activerecord/lib/active_record/insert_all.rb:173-188`) raises on
four independent conditions:

```ruby
raise ArgumentError, "#{connection.class} does not support :returning"        if returning && !connection.supports_insert_returning?
raise ArgumentError, "#{connection.class} does not support skipping duplicates" if skip_duplicates? && !connection.supports_insert_on_duplicate_skip?
raise ArgumentError, "#{connection.class} does not support upsert"            if update_duplicates? && !connection.supports_insert_on_duplicate_update?
raise ArgumentError, "#{connection.class} does not support :unique_by"        if unique_by && !connection.supports_insert_conflict_target?
```

trails ports only the first
(`packages/activerecord/src/insert-all.ts:383-390`), and even that arm diverges:
it throws plain `Error`, not `ArgumentError`, with the message
`"... does not support INSERT...RETURNING"` rather than Rails'
`"... does not support :returning"`.

Before #7053 the three missing arms would have needed async support predicates;
that blocker is gone — the method is now synchronous and reads the pre-resolved
`_facts`, so `supportsInsertOnDuplicateSkip` / `supportsInsertOnDuplicateUpdate`
only have to join `ResolvedConnectionFacts`.

## Converged shape

`ensureValidOptionsForConnectionBang` has all four guards, in Rails' order,
each raising `ArgumentError` with Rails' exact message string.
`ResolvedConnectionFacts` grows `supportsInsertOnDuplicateSkip` and
`supportsInsertOnDuplicateUpdate`, resolved in `resolveConnectionFacts`
alongside the two predicates already there.

## Acceptance criteria

- [ ] All four `insert_all.rb:173-188` guards present, in Rails' order.
- [ ] Every arm raises `ArgumentError` with Rails' message string, including
      the `:returning` arm (currently plain `Error` with an invented message).
- [ ] `insert-all.test.ts` / `upsert-all.test.ts` stay green, test names
      unchanged.
