---
title: "Seed GeneratedAttributeMethods before the class body runs, retiring uninclude()"
status: claimed
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: "2026-08-23T22:40:29Z"
assignee: "collapse-the-activerecord-secure-password-duplicate"
blocked-by: null
closed-reason: null
---

# Seed `GeneratedAttributeMethods` before the class body runs

## Context

Found while landing trails#6940 (`call-define-attribute-methods-from-init-internals`).

Rails calls `initialize_generated_modules` from the `inherited` hook
(`activerecord/lib/active_record/attribute_methods.rb:265-272`, seeded from the
`included do` at :10-11), so `@generated_attribute_methods` always holds the
`GeneratedAttributeMethods` (`:41-47`) before any class body can run. A Rails
model therefore has exactly ONE generated-methods module for its whole life and
`initialize_generated_modules` never replaces anything.

trails has no `inherited` hook, so `initializeGeneratedModules`
(`packages/activerecord/src/attribute-methods.ts:314`) runs lazily out of
`defineAttributeMethods` (`:447`). A class body that calls `attribute()` or
`aliasAttribute()` gets there first, reaching ActiveModel's lazy
`generated_attribute_methods`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:400-402`,
ours `packages/activemodel/src/attribute-methods.ts:524`), which seats a bare
`Module` and includes it. `initializeGeneratedModules` then has to REPLACE it.

Because `include()` (`packages/activesupport/src/include.ts:200`) splices a
fresh carrier object into the prototype chain on every call, that replacement
stranded the old carrier in the chain — two links answering the same generated
names, so `undefineAttributeMethods` cleared only one and the accessor survived
an undefine. #6940 fixed the symptom by carrying the old module's methods over
and un-splicing its carrier via a new `uninclude()`
(`packages/activesupport/src/include.ts:200`, tagged
`@noRailsEquivalent PERMANENT`).

`uninclude()` is debt. It only exists because trails reaches
`initialize_generated_modules` late; Rails has no un-include because Ruby's
ancestry never grows the duplicate entry in the first place.

## Converged shape

Seed the `GeneratedAttributeMethods` at the earliest AR-owned point for a
class, so the ivar is already populated before any class body can reach
ActiveModel's lazy fallback — the trails stand-in for Rails' `inherited`. Then:

- `initializeGeneratedModules` loses its replacement branch (the `previous`
  capture, the descriptor copy, and the `uninclude()` call) and matches
  `attribute_methods.rb:41-47` line for line.
- The `!(this._generatedAttributeMethods instanceof GeneratedAttributeMethods)`
  half of the gate at `attribute-methods.ts:443-445` collapses to the plain
  own-property check Rails' `@generated_attribute_methods` ivar implies.
- `uninclude()` is deleted from `packages/activesupport/src/include.ts` and its
  export dropped from `packages/activesupport/src/index.ts` — nothing else
  calls it.

Note the candidate seams are all AR-owned entry points a class body must pass
through (`Base.attribute`, `Base.aliasAttribute`, `registerModel` /
`registerSubclass`); pick one that no class body can bypass, or the bare-module
race comes back for whatever path misses it.

## Acceptance criteria

- [ ] An AR class's `_generatedAttributeMethods` is a `GeneratedAttributeMethods`
      before its class body's first `attribute()` / `aliasAttribute()` returns.
- [ ] `initializeGeneratedModules` never replaces an already-included module;
      the replacement branch is gone.
- [ ] `uninclude()` is deleted from activesupport along with its barrel export.
- [ ] `seats one generated-methods carrier when construction generates first`
      (`attribute-methods.trails.test.ts`) still passes, as do
      `attribute-methods.test.ts`, `attribute-methods.trails.test.ts` and
      `base.test.ts` on SQLite, PostgreSQL and MySQL/MariaDB.
