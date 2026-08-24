---
title: "converge-delegated-type-and-default-scope-call-set-rows"
status: done
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: 7008
claim: "2026-08-24T22:18:08Z"
assignee: "converge-delegated-type-and-default-scope-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

Two residual `kind: "set"` rows with no owning RFC named:

| shard                  | ruby method                     | missing call    | Rails                                       |
| ---------------------- | ------------------------------- | --------------- | ------------------------------------------- |
| `delegated-type.json`  | `define_delegated_type_methods` | `define_method` | `delegated_type.rb:246,250,254,265,269,273` |
| `scoping/default.json` | `build_default_scope`           | `any?`          | `default.rb:157`                            |

`define_delegated_type_methods` is six `define_method "#{role}_class"` calls.
The decomposition already converged — the generated methods live in this body —
but Ruby's `define_method` is `Object.defineProperty` on the prototype in JS, so
there is no `defineMethod` to call. RFC 0106's permanence audit ruled this file
CONVERGEABLE and its rows non-migratable to a tag, which is why it is still a
row; converging it means a ported `defineMethod` helper on the class-methods
seam, not a receipt.

`build_default_scope -> any?` is `elsif default_scopes.any?` (`default.rb:157`).
The port spells it `scopes.length === 0` and inverts it into an early return
(`packages/activerecord/src/scoping/default.ts:71-72`), so Rails' `elsif` became
a guard. Same predicate, different branch shape — CLAUDE.md's control-flow rule
(same branches, same order, no inverted guards) makes this a straight
convergence, not a receipt.

## Acceptance criteria

- [ ] `buildDefaultScope` restores Rails' branch order and calls the ported
      `any` receiver form on `defaultScopes`.
- [ ] `defineDelegatedTypeMethods` calls a ported `defineMethod`, or the row
      carries an honestly classified `@missingRailsCall` receipt written against
      the permanence audit's CONVERGEABLE finding rather than around it.
- [ ] Both rows deleted by hand; `pnpm parity:api:calls` / `:args` green; no
      `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
