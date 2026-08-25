---
title: "extend() copies module functions Ruby's extend never carries"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby's `extend SomeModule` copies the module's INSTANCE methods onto the
receiver's singleton, and never its module functions (`def self.x`). trails'
`extend()` (`packages/activesupport/src/include.ts:351-384`) copies every own
key of the module object instead — non-enumerable statics for a class module,
enumerable keys for a plain object — so a ported module that carries both kinds
leaks its module functions onto every host that extends it.

`ActiveModel::Naming` is the live instance and the one that surfaced this
(`vendor/rails/activemodel/lib/active_model/naming.rb:252-348`):

- `def model_name` (`:270`) — a module instance method; `extend
ActiveModel::Naming` (api.rb:66) copies it.
- `def self.plural`, `self.singular`, `self.uncountable?`,
  `self.singular_route_key`, `self.route_key`, `self.param_key`,
  `self.model_name_from_record_or_class` (`:283-348`) — module functions Ruby
  does NOT copy; they stay `ActiveModel::Naming.plural(record)`.

PR #7010 first moved `model_name` onto a class module merged with the existing
`namespace Naming` and shipped seven invented statics on `Model`
(`'plural' in Model === true`, verified at runtime, reverted in `d9ab0ff`). It
landed instead by hand-drawing the line at `extend()`'s enumerable-own-key rule
in `packages/activemodel/src/naming.ts` — the module functions are re-defined
non-enumerable at the bottom of the file, `model_name` is defined enumerable.

That works and is cited, but it is a per-file convention nothing enforces: the
next module with both method kinds leaks silently, and nothing fails when it
does. `foldClassMethodsModules` in `scripts/parity/conventions.ts` already knows
which Ruby members are `ClassMethods`, so the information to check this exists.

## Converged shape

`extend()` copies only the module's instance half, and a ported module declares
its two halves so that is unambiguous — the settled carrier for "Ruby module
with both `def x` and `def self.x`", applied to `naming.ts` first. Whatever the
carrier, `naming.ts`'s hand-rolled enumerability loop goes away, and a leak is
caught rather than relied on not happening:

- `Naming.plural(record)` still answers.
- `'plural' in Model` is `false`, and stays false without a per-file loop.
- Ideally a lint or a `parity:api` check fails on a module function reaching a
  host through `extend()`, so the next instance is not silent.

## Acceptance criteria

- One settled representation for a Ruby module carrying both method kinds,
  documented where the other mixin idioms are (CLAUDE.md § "Module mixins").
- `packages/activemodel/src/naming.ts` uses it; the `Object.keys(Naming)`
  enumerability loop at the bottom of that file is deleted.
- A regression that fails on the leak (`'plural' in Model`) — it must fail on
  the pre-fix shape.
- `pnpm vitest run packages/activemodel/src packages/activerecord/src/base.test.ts`;
  parity deltas non-negative.

## Definition of done

Not done while the guarantee rests on a per-file loop that a new module can
forget.
