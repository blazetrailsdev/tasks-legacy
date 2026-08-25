---
title: "Errors#copy! fuses Rails' deep_dup into dupWithBase — converge and drop the @missingRailsCall tag"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced by PR #6647's `@missingRailsCall` tag, added to keep
`pnpm parity:api:calls` green after `DirtyTracker.deepDup` gave the comparator a
`deepDup` name to resolve and exposed this pre-existing omission.

Rails `ActiveModel::Errors#copy!`
(`vendor/rails/activemodel/lib/active_model/errors.rb:138-143`):

```ruby
def copy!(other) # :nodoc:
  @errors = other.errors.deep_dup
  @errors.each { |error|
    error.instance_variable_set(:@base, @base)
  }
end
```

Two statements: deep-dup the array, then rebase each copy's `@base`.

trails `Errors#copyBang` (`packages/activemodel/src/errors.ts:76-78`) is one:

```ts
copyBang<U extends object>(other: Errors<U>): void {
  this._errors = other._errors.map((e) => e.dupWithBase(this._base));
}
```

`Error#dupWithBase` (`packages/activemodel/src/error.ts:334-336`) constructs a
replacement with the new base, deep-dupping the options as it goes. It exists
because `Error#base` is declared `readonly`
(`packages/activemodel/src/error.ts:83`) and TypeScript has no
`instance_variable_set` escape hatch, so Rails' rebase-after-dup cannot be
expressed as an assignment.

The `@missingRailsCall deep_dup` tag is the sanctioned form for a single
justified omission, but per CLAUDE.md a documented deviation is debt, not
permission — hence this story. `dupWithBase` is also extra surface with no Rails
counterpart (Rails has no `Error#dup_with_base`).

## Converged shape

Make `base` writable through a Rails-named private seat so the two Rails
statements can be transcribed literally, and retire `dupWithBase`:

```ts
copyBang<U extends object>(other: Errors<U>): void {
  this._errors = deepDup(other._errors);
  for (const error of this._errors) error.base = this._base;
}
```

This needs `Error#deepDup` (or `Array` deep-dup that dispatches
`Error#initializeDup` — Rails' `Error#initialize_dup` is at
`vendor/rails/activemodel/lib/active_model/error.rb:111-116` and dups
`@attribute`, `@raw_type`, `@type`, `@options`) plus a settable `base`. Note the
`@base` write is what Rails' `NestedError` relies on too, so check
`packages/activemodel/src/nested-error.ts` before flipping `readonly`.

Do NOT converge by calling `deepDup` and then `dupWithBase` — that dups every
error twice for no effect, which is why the omission was tagged rather than
papered over in #6647.

## Acceptance criteria

- `Errors#copyBang` makes the `deep_dup` call Rails makes, and the
  `@missingRailsCall deep_dup` tag is deleted from
  `packages/activemodel/src/errors.ts`.
- `Error#dupWithBase` is gone, or, if it survives, carries its own reviewed
  justification — it is not a Rails name.
- `pnpm parity:api:calls` stays green with no new baseline row, verified with
  `API_COMPARE_FORCE=1 pnpm parity:api --calls` (a plain run gates a partially
  regenerated artifact).
- `pnpm parity:api:extra --package activemodel` gains no untagged surface.
- `errors_test.rb` / `nested_error_test.rb` stay green.
