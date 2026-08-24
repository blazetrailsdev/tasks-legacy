---
rfc: "0119-connection-adapter-fidelity"
title: "Connection-adapter fidelity — converge connection_adapters/ onto the Rails tree"
status: draft
created: 2026-08-24
updated: 2026-08-24
owner: "@deanmarano"
packages:
  - "activerecord"
clusters:
  - "rails-deviation"
related-rfcs:
  - "0023-surfaced-deviations"
---

# RFC 0119 — Connection-adapter fidelity

## Summary

Converge `packages/activerecord/src/connection-adapters/**` onto
`vendor/rails/activerecord/lib/active_record/connection_adapters/**`, method body by
method body. This RFC is a **carve-out from `0023-surfaced-deviations`**: 118 open
stories filed there over 2026-07 and 2026-08 all name a divergence inside one Rails
directory tree, and 0023 — a deliberately plan-free holding area with 679 open
stories — has no rollout ordering to give them. This RFC gives that cohort an owner,
a phase order, and an end condition.

## Motivation

`0023-surfaced-deviations` is the standing catch-all for port-discovered work. Its
README is explicit that it is "the waiting room, not the ward", and that a story
which survives a sweep with a live, cited divergence "should be re-homed onto the
topical RFC that owns the surface". No such RFC existed for the adapter tree, so the
adapter findings accumulated in the bucket instead — 118 of them, the single largest
coherent group in a 1545-story RFC.

They are coherent because they answer the same question in the same voice: _a trails
adapter method does something Rails' adapter method does not._ Every one cites both
sides concretely, as 0023's intake rule requires. Representative live examples:

- `PostgreSQLAdapter#transactionStatus` synthesizes `PQTRANS_ACTIVE` from a
  hand-maintained flag rather than reading the driver's status.
- `create_schema_dumper` takes `(source, options)` where Rails takes `(options)` and
  passes `self` (`abstract_adapter.rb`).
- `Queue#clear` returns the removed elements; Rails' `Array#clear` does not
  (`connection_pool/queue.rb`).
- `buildAdapterArg`'s SQLite whitelist silently drops config keys Rails passes
  straight through to the adapter.
- `sqlite3`'s `check_constraints` regex adds a quoted-name arm and caps nesting depth
  that Rails' has no counterpart for.
- `AbstractAdapter#resetBang` drops Rails' `attempt_configure_connection`.
- `dirties_query_cache` is wired on `exec_insert`/`update`/`delete` rather than Rails'
  public `insert`/`update`/`delete`, leaving PG's non-returning arm uncleared.

Left in the catch-all these read as 118 unrelated chores. Grouped, they are one
campaign with a measurable finish line, over a tree that already has a parity gate
pointed at it.

The cohort is also **live**, not stale: every story in it was last updated in 2026-07
(47) or 2026-08 (71). A symbol-level sweep against `main` at `152b2ebe9` found six
stories whose premise had already landed or was misfiled; those six were excluded
from the carve-out and remain in 0023 for a triage pass to close.

## Design

### Membership rule

A story belongs to this RFC if the trails `file:line` it cites lives under
`packages/activerecord/src/connection-adapters/**` or `src/adapters/**` — plus the
adjacent files Rails also places in `connection_adapters/`: `database-configurations`,
`connection-handling`, `schema-cache`, and the `tasks/*-database-tasks` entry points
that construct adapters. 120 of the 124 candidates cite such a path directly; the
remainder are adapter-test stories whose only path is a `.test.ts` inside the tree.

### Composition

| area                                                                                                                                                | open stories |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| `connection-adapters/abstract/**`, `abstract-adapter.ts` — schema statements, schema definitions, database statements, quoting, transaction manager | ~50          |
| `connection-adapters/postgresql/**`, `postgresql-adapter.ts`                                                                                        | ~30          |
| `connection-adapters/sqlite3*`                                                                                                                      | ~18          |
| `abstract-mysql-adapter.ts`, `mysql2*`                                                                                                              | ~13          |
| `database-configurations`, `url-config`, `connection-handling`, `tasks/`, top-level `connection-adapters.ts`                                        | ~13          |

`connection-pool/**` (queue, reaper, execution-context) and `connection-handler`
account for 18 of the abstract-tree stories and are **in scope** — Rails places them
in `connection_adapters/abstract/` too. They share a boundary with
`0073-permanent-connection-checkout-flip`; where a story is really about that flip,
it stays there.

### Out of scope

Anything above the adapter boundary: Arel visitors that emit the SQL, `Relation` and
query-methods callers, and the type/cast layer. A story that merely _observes_
adapter behaviour from `relation.ts` is not adapter work and stays in 0023 or moves
to the RFC that owns its surface.

Also out of scope, per 0023's own named non-goals: infrastructure and tooling, and
Ruby interpreter semantics with no TS analogue.

## Rollout

Phased by dependency, because the dialects inherit from the abstract tree and a
divergence fixed in a subclass first has to be re-fixed when the base converges:

1. **Abstract adapter and its mixins** — schema statements/definitions, database
   statements, quoting. The largest phase and the one every later phase rests on.
2. **Connection pool and lifecycle** — queue, reaper, execution context, handler,
   checkout/checkin. Coordinated with `0073`.
3. **Per-dialect adapters** — PostgreSQL, MySQL, SQLite3. Parallelisable across the
   three once phase 1 has landed; each dialect's stories are independent of the
   others'.
4. **Configuration and construction** — `DatabaseConfigurations`, `UrlConfig`,
   `establish_connection`, database tasks, `dbconsole`.

Stories carry their `est-loc` from 0023; see the live index
(`pnpm tasks list --rfc <this-rfc>`) for current contents and ordering.

## End condition

The RFC closes when all three hold:

- Every `call-mismatches-exclude/` row whose source file is under the adapter tree is
  gone — converged, not rewritten with a better `reason`. The baselines are
  only-shrink, so this is directly measurable by
  `pnpm parity:api:calls` and `pnpm parity:api:calls:args`.
- `pnpm parity:api:extra --package activerecord` reports no untagged public name
  under the adapter tree; every remaining `@noRailsEquivalent` there carries a
  reviewed `PERMANENT` reason naming a real TypeScript language wall.
- Every story in this RFC is `done` by convergence or `closed` with a Rails
  `file:line` showing there was nothing to converge. Per CLAUDE.md, a
  deviation-convergence story is never closed by writing a better justification for
  the deviation, by broadening a baseline reason, or by moving it to another register.

## Alternatives considered

- **Leave the cohort in `0023-surfaced-deviations`.** Rejected — that is the status
  quo, and 0023's plan-free design is what let 118 related findings sit unordered.
  0023's README already prescribes re-homing surviving stories onto a topical RFC.
- **Supersede 0023 with `--carry`.** Rejected — `--carry` migrates _every_ open story
  of the source RFC, which would empty the catch-all the port loop still files into.
  The carve-out is partial by construction.
- **Carve by package instead of by tree** (an "ActiveRecord fidelity" RFC). Rejected —
  a package is not a subject. The equivalent ActiveSupport grouping spans
  `StringScanner`, JSON, cache stores, callbacks and `Duration`, which is exactly the
  ragged shape that made 0023 unclosable.
- **Split per dialect** (one RFC each for PG, MySQL, SQLite3). Rejected — the three
  share the abstract base, and roughly half the cohort is in that base. Splitting
  would give three RFCs that each block on the same fourth.

## Open questions

1. **Boundary with `0073-permanent-connection-checkout-flip`.** The 18 pool stories
   overlap it. Recommendation: `0073` owns the checkout-model decision; this RFC owns
   the method-body fidelity of the pool once that model is settled.
2. **Sibling carve-outs.** Associations (62 open) and type/cast (65, itself two
   campaigns) are the next candidates out of 0023. Not blocked on this RFC; noted so
   the pattern is applied deliberately rather than ad hoc.

## Changelog

- 2026-08-24: initial RFC — carve-out of 118 open connection-adapter stories from
  `0023-surfaced-deviations`, with a phase order and a gate-measurable end condition.
