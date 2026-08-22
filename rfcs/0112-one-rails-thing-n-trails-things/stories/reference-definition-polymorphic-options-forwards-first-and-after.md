---
title: "ReferenceDefinition#polymorphicOptions drops first/after positioning"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ReferenceDefinition#polymorphicOptions`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:760-766`)
forwards only `null` from the caller's options:

```ts
private polymorphicOptions(): ColumnOptions {
  return {
    ...this.asOptions(this.polymorphic),
    ...this.conditionalOptions(),
    ...(this.options.null !== undefined ? { null: this.options.null } : {}),
  };
}
```

Rails' `polymorphic_options`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:259`)
forwards three keys:

```ruby
def polymorphic_options
  as_options(polymorphic).merge(conditional_options).merge(options.slice(:null, :first, :after))
end
```

So `t.references :taggable, polymorphic: true, after: :id` positions only the
`taggable_id` column (which receives the full `options` via `_columns`) and
silently drops the positioning for the `taggable_type` column, splitting the
pair. Same for `first:`.

Root cause is that `ColumnOptions`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:~458`)
does not model MySQL column positioning at all — there is no `first` or `after`
field on the interface, so the keys cannot be sliced even if
`polymorphicOptions` wanted to. This is why the gap is wider than one line.

Surfaced during review of #5485 / #5480 (`ReferenceDefinition#add`). Not
introduced by either.

## Acceptance criteria

- [ ] `ColumnOptions` models `first?: boolean` and `after?: string`, matching
      the MySQL positioning clause Rails supports.
- [ ] `polymorphicOptions` forwards `null`, `first` and `after`, mirroring
      `options.slice(:null, :first, :after)` (`schema_definitions.rb:259`).
- [ ] Regression coverage that fails on baseline: a polymorphic reference with
      `after:` positions both the `_type` and `_id` columns.
- [ ] `parity:api --package activerecord` and
      `parity:test --package activerecord` deltas are non-negative;
      `parity:test --gates --check` exits 0.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
