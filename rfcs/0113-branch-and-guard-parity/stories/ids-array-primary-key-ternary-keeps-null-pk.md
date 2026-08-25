---
title: "ids-array-primary-key-ternary-keeps-null-pk"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `Relation#ids` spells `Array(primary_key)` as a ternary that keeps a `null` PK

## Context

`packages/activerecord/src/relation.ts` (`ids()`, the body landed by PR #6564,
commit `17309a794`) opens with:

```ts
const primaryKey = this.model.primaryKey;
const primaryKeyArray = Array.isArray(primaryKey) ? primaryKey : [primaryKey];
```

Rails is `primary_key_array = Array(primary_key)`
(`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:372`).
Ruby's `Kernel#Array(nil)` returns `[]`; the TS ternary returns `[null]`.

`this.model.primaryKey` is `string | string[] | null` and is genuinely `null`
for a model with no primary key, so every downstream consumer of
`primaryKeyArray` receives a one-element array holding `null` where Rails hands
them an empty one:

- the `loaded?` arm takes the `length === 1` branch and calls
  `record._readAttribute(null)` (`calculations.rb:376-379` reads nothing at all,
  since `primary_key_array.one?` is false for `[]`);
- the query arm builds `arelColumns([null])` (`calculations.rb:390`
  `arel_columns(primary_key_array)`).

This is the `fetch`/truthiness idiom class from CLAUDE.md — a Ruby conversion
whose nil arm was dropped in the port — not a language shortcoming.

## Converged shape

Spell `Array(primary_key)`'s nil arm explicitly:

```ts
const primaryKeyArray =
  primaryKey == null ? [] : Array.isArray(primaryKey) ? primaryKey : [primaryKey];
```

Check whether the same ternary was copied to sibling call sites in
`relation.ts` / `relation/calculations.ts` while you are there — the shape is
short enough to have spread.

## Acceptance criteria

- [ ] `ids()` returns Rails' value for a model whose `primaryKey` is `null`,
      matching `Array(nil) == []`.
- [ ] A test covering the no-primary-key relation that fails on the pre-change
      tree.
- [ ] Any sibling `Array(...)` ternary with the same dropped nil arm is
      converged or filed.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
