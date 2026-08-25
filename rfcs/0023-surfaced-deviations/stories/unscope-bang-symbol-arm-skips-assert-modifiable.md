---
title: "unscope!'s Symbol arm skips assert_modifiable! — a loaded relation mutates instead of raising ImmutableRelation"
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

## Context

Found while landing PR #6773 (`retire-join-clauses-sidecar-store`), which
inlined `resetValueForScope` back into `unscope!`'s Symbol arm and put the
Rails line side by side with the trails one.

Rails calls `assert_modifiable!` at three sites in `query_methods.rb`:

- `query_methods.rb:179` — the generated `#{method_name}_values=` writer
- `query_methods.rb:821` — `unscope!`'s Symbol arm, immediately before
  `@values.delete(scope)`
- `query_methods.rb:1746` — the definition itself

trails calls `assertModifiableBang` from only one of them: the generated
value-writer's `set` accessor (`packages/activerecord/src/relation/query-methods.ts:260`).
`unscopeBang`'s Symbol arm deletes the key directly —

```ts
delete this._values[scope as UnscopeType];
```

— with no `assertModifiableBang.call(this)` in front of it.

So `relation.unscope(:order)` on a loaded relation silently mutates it in
trails, where Rails raises `ImmutableRelation`. The Hash (`where:`) arm goes
through the `whereClause=` writer and so is already guarded; only the Symbol
arm is unguarded.

`assertModifiableBang` already exists and is exported
(`query-methods.ts:2100`, mirroring `query_methods.rb:1746-1750`), so this is
a one-line call, not new machinery.

## Converged shape

```ts
assertModifiableBang.call(this);
delete this._values[scope as UnscopeType];
```

placed exactly where Rails places it (`query_methods.rb:821`) — after the
`VALID_UNSCOPING_VALUES` check, before the delete.

## Acceptance criteria

- [ ] `unscopeBang`'s Symbol arm calls `assertModifiableBang` before deleting
      the value key, matching `query_methods.rb:820-822`.
- [ ] Regression test (failing on baseline): `unscope(:order)` on a loaded
      relation raises `ImmutableRelation`, mirroring Rails' `assert_modifiable!`
      behaviour; check `vendor/rails/activerecord/test/cases/relations_test.rb`
      for an existing Rails test to mirror by name before writing a trails-only one.
- [ ] `pnpm parity:api:calls` / `:args` add zero rows (this should REMOVE a
      call-mismatch row if `unscope!` carries one).
