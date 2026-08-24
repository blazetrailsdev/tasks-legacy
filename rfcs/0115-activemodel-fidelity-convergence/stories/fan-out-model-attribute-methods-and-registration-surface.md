---
title: "Fan out the attribute-methods and attribute-registration surface from model.ts"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-dirty-surface-to-dirty-ts
deps-rfc: []
est-loc: 300
pr: null
claim: "2026-08-24T18:16:30Z"
assignee: "fan-out-model-attribute-methods-and-registration-surface"
blocked-by: null
closed-reason: null
---

## Context

`model.ts` shadows 25 members belonging to
`attribute_methods.rb` (24 code lines, 13 members),
`attribute_registration.rb` (25 lines, 8), `attributes.rb` (17 lines, 4) and
`attribute_assignment.rb` (16 lines, 4):

- attribute_methods: `aliasAttribute` `:373`, `attributeMethodPrefix` `:1584`,
  `attributeMethodSuffix` `:1585`, `attributeMethodAffix` `:1586`,
  `undefineAttributeMethods` `:1587`, `attributeAliases` `:1588`,
  `attributeMethodPatterns` `:1590`, `isRespondToWithoutAttributes` `:1592`,
  `_readAttribute` `:1775`, `attributeMissing` `:1795`, `respondTo` `:2692`,
  `isAttributeMethod` `:701`, `resolveAttributeName` `:338`
- attribute_registration: `decorateAttributes` `:316`, `attributeTypes` `:317`,
  `typeForAttribute` `:318` / `:2703`, `pendingAttributeModifications` `:322`,
  `resetDefaultAttributesBang` `:327`, `resolveTypeName` `:343`,
  `hookAttributeType` `:351`
- attributes: `defineMethodAttribute` `:313`, `_writeAttribute` `:1809`,
  `attribute` (static assign `:312`), `attributeNames` `:362` / `:1890`
- attribute_assignment: `assignAttributes` `:2607`, `_assignAttributes`
  `:2616`, `_assignAttribute` `:2625`, `attributeWriterMissing` `:2671`

Every destination file already defines the same name —
`packages/activemodel/src/attribute-methods.ts` has `attributeMissing` `:169`,
`isRespondToWithoutAttributes` `:194`, `isAttributeMethod` `:204`,
`matchedAttributeMethod` `:209`, `_readAttribute` `:236`,
`attributeMethodPrefix` `:256`, `resolveAttributeName` `:493`;
`attribute-registration.ts` has `typeForAttribute` `:26`/`:253`,
`resolveTypeName` `:354`, `hookAttributeType` `:370`,
`resetDefaultAttributesBang` `:322`; `attributes.ts` has `_writeAttribute`
`:72`, `attribute` `:124`, `attributeNames` `:281`.

So this too is a shadow deletion. Many of the `model.ts` entries are already
one-line `X.call(this, …)` thunks (e.g. `:322`, `:327`, `:338`) — exactly the
shape CLAUDE.md's "Module mixins" section says to replace with `include()` /
`Included<>`, and the shape RFC 0107 retired wholesale from `relation.ts`.

**Coordinate with open PR #6738**, which edits `_assignAttribute`,
`matchedAttributeMethod` and the `attribute=` alias on `Model`. Branch from
`main` after it merges.

## The mixin idiom to use (RFC finding F0)

All three mechanisms this story needs are already ported and exported, and
activemodel currently uses none of them — see this RFC's F0. Do not hand-roll a
fourth spelling:

- **`classAttribute()`** — `packages/activesupport/src/class-attribute.ts:70`,
  exported from the package index (`:387`). Its contract is exactly Rails'
  `class_attribute`: _"reads walk the constructor chain; writes are local to the
  class"_. It has **zero** callers in activemodel today.
- **`extend()` / `Extended<>`** — `packages/activesupport/src/include.ts:335`.
  The TS spelling of `extend SomeModule`, i.e. the `ClassMethods` half of a
  Concern. **Zero** callers in activemodel; 65 in activerecord.
- **`include()` / `Included<>`** — `include.ts:184`, plus the symbol-keyed
  `[included]` / `[extended]` hooks fired at `include.ts:193,272,371`, which are
  the TS spelling of an `included do` block. The hooks are keyed by
  `Symbol.for(...)`, so they never surface to `parity:api:extra` and do not
  collide with the `SKIP_GROUPS` ban on a string-named `included` member
  (`scripts/parity/conventions.ts:444`, `tsMirrorIsDrift: true`). CLAUDE.md's
  "Module mixins" section still says these hooks have no TS equivalent; that is
  stale for `included`/`extended` and true only for `inherited`.

## Acceptance criteria

- `model.ts` defines none of the 25; `Model` reaches the instance half through
  `include()` / `Included<>` and the **class** half through `extend()` /
  `Extended<>`. The `static X = X` assignment block at `model.ts:312-319`,
  `:373` and `:1571-1591` — 22 lines that are a hand-rolled
  `extend ClassMethods` — is gone (F0). `extend()` currently has zero callers
  in activemodel and 65 in activerecord.
- `attribute_methods.rb:70-73`'s `included do class_attribute … end` is issued
  from the `[included]` hook with `classAttribute()`, not as hard-coded static
  state on `Model` (`model.ts:278-290` `_attributeMethodPatterns`,
  `_attributeAliases`).
- `typeForAttribute` keeps the "never raises, returns the nil type" behaviour
  pinned by the existing tests (`type_for_attribute` does not raise in Rails).
- Where the `Model` wrapper diverged from the destination body, the destination
  body wins and the divergence is stated in the PR.
- `pnpm parity:api:extra --package activemodel` loses these rows from
  `model.ts`.
- `pnpm lint --fix` after `pnpm parity:api` (activemodel is in the
  `rails-file-structure-method-order` set).
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed.

## Verification

```bash
pnpm vitest run packages/activemodel/src/attribute-methods.test.ts packages/activemodel/src/attribute-methods.trails.test.ts packages/activemodel/src/attribute-registration.test.ts packages/activemodel/src/attributes.test.ts packages/activemodel/src/attribute-assignment.test.ts
```
