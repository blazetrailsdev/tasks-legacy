---
title: "build_default_scope inverts Rails' elsif default_scopes.any? into an early return"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# `build_default_scope` inverts Rails' `elsif default_scopes.any?` into an early return

## Context

Surfaced converging the `scoping/default.json` call-set rows in PR #6735 (RFC
0106 wave 4c); left baselined as `build_default_scope -> any?`.

Rails
(`vendor/rails/activerecord/lib/active_record/scoping/default.rb:145-170`) is a
three-arm `if / elsif / (implicit nil)`:

```ruby
if default_scope_override
  evaluate_default_scope { relation.scoping { default_scope } }
elsif default_scopes.any?
  evaluate_default_scope do
    default_scopes.inject(relation) { ... }
  end
end
```

trails (`packages/activerecord/src/scoping/default.ts:71-72`) turns the second
arm's condition inside out:

```ts
const scopes: DefaultScope[] = this.defaultScopes ?? [];
if (scopes.length === 0) return undefined;
```

Same predicate, opposite polarity, and the `elsif` becomes a guard clause — one
of the shapes CLAUDE.md's "Control flow" rule names explicitly ("Do not collapse
two Rails branches into one, invert a guard, reorder side-effect-free calls").
It reads correctly today but the next edit to either arm has no structural
partner to line up against.

## Converged shape

Restore the `if / elsif` shape with the condition in Rails' polarity:

```ts
if (this.defaultScopeOverride) {
  return evaluateDefaultScope.call(this, () => { ... });
} else if ((this.defaultScopes ?? []).length > 0) {
  return evaluateDefaultScope.call(this, () => { ... });
}
return undefined;
```

`length > 0` is the spelling of Ruby's `Array#any?` for a plain array — there is
no `any?` port to call, so the row is retired by the branch shape, not by a new
helper. Do NOT invent an `any` helper for this.

## Acceptance criteria

- [ ] `buildDefaultScope` reads as Rails' `if / elsif`, in Rails' polarity, with
      the implicit `undefined` third arm.
- [ ] The `scoping/default.json` `build_default_scope -> any?` row is deleted,
      then `pnpm parity:api:calls:tighten activerecord/scoping/default.json`.
- [ ] No new helper is added for `any?`.
- [ ] Default-scope and STI suites green on all three lanes.
