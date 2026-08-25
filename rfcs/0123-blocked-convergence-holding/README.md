---
rfc: "0123-blocked-convergence-holding"
title: "Holding epic: blocked convergence work carried from RFCs 0078 / 0096 / 0106 / 0107"
status: active
created: 2026-08-25
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
  - "activemodel"
  - "arel"
  - "activesupport"
clusters:
  - "schema"
  - "api-compare"
related-rfcs:
  - "0078-sti-schema-reflection-fidelity"
  - "0096-naming-identifier-burndown"
  - "0106-wide-call-set-direct-burndown"
  - "0107-relation-ts-decomposition"
---

A **holding epic**. It owns no convergence programme of its own; it holds the
blocked stories carried out of four RFCs that had run out of pickup-able work,
so those RFCs could be retired without dropping the debt they had surfaced.

Closed and superseded into this RFC on 2026-08-25:

- `0078-sti-schema-reflection-fidelity` — 5 blocked stories carried
- `0106-wide-call-set-direct-burndown` — 3 blocked stories carried
- `0107-relation-ts-decomposition` — 4 blocked stories carried

`0096-naming-identifier-burndown` stays **active** — its claimed and in-progress
naming waves are still in flight. Only its one blocked story, `naming-gate-flip`,
moved here.

Nothing here is ratified. Every story arrived blocked with its blocker reason,
priority, deps and est-loc intact, and a story leaves this RFC by converging (or
by moving to whichever RFC owns its unblocking work once that work exists) — not
by being re-justified. See CLAUDE.md, "A documented deviation is debt, not
permission".

## Blocker themes

The 13 stories group into five blockers. Eight of the thirteen are the same one.

### 1. No synchronous schema or connection read exists in TypeScript (8 stories)

The dominant theme, and the reason three separate RFCs stalled at once. Ruby can
execute SQL synchronously, so Rails resolves schema and connections inline:
`attribute_types` / `_has_attribute?` load the schema on first touch
(`inheritance.rb:61` -> `model_schema.rb` `load_schema`), `schema_cache.columns_hash`
either answers or raises (`model_schema.rb:587-597`), `arel` acquires through a
block (`query_methods.rb:1595`), and `apply_join_dependency` executes a query
mid-build (`finder_methods.rb:457-481`). trails' reflection and
`ConnectionPool#withConnection` are `async`, so each of these ported bodies
carries either a trails-only cold-window fallback or a synchronous duplicate of
an async builder.

- `converge-new-sti-gate-drop-stienabled-disjunct` — the `new()` STI gate's
  trails-only `!stiEnabled` disjunct stands in for a cold `classHasAttribute`.
- `company-model-invents-inheritance-column-assignment` — the invented
  `Company.inheritanceColumn = "type"` line is what makes that disjunct true;
  deps on the story above.
- `delete-warm-cache-reinvalidation-in-reflect-column-names` — the warm-cache
  re-invalidation block exists only because a sync `loadSchema` can settle
  against a cold cache.
- `converge-get-primary-key-lease-free-schema-cache-reads` — gated on RFC 0073
  (permanent-connection-checkout flip), still `draft`; `getPrimaryKey` runs on
  sync paths and must not lease.
- `port-with-connection-acquisition-seam-for-the-arel-reader` — no synchronous
  `with_connection` seam to port the `@arel ||=` reader against.
- `converge-sync-eager-builders-async-to-sql` — `toSql(): string` is read by 31
  non-test and 683 test call sites; the invented sync eager builders exist to
  serve them.
- `converge-excluding-deferred-ids-marker-to-eager-materialization` — same root,
  recorded as a deps edge on the arel-seam story.
- `load-async-bypasses-exec-queries-prerequisites` — the two trails-only
  `execQueries` prerequisites (lazy schema reflection, deferred distinct-PK
  materialization) are exactly the async-ness above, leaking into ordering.

These unblock together, not individually. The shape that closes them is a
synchronous schema/query seam (or an async `toSql`/`arel` with every caller
converted) plus the RFC 0073 lease decision — each of which is its own RFC-sized
programme with its own three-lane adapter run, which is why none of them fit
inside 0078/0106/0107.

### 2. JS offers no receiver-bearing `class X extends Y {}` hook (1 story)

`converge-schema-invalidation-onto-push-only-eager-registration`. Ruby's
`inherited` fills `DescendantsTracker` the moment a subclass is defined, so
Rails' invalidation is push-only against one VM-maintained registry
(`descendants_tracker.rb:97-100`). `[[SetPrototypeOf]]` fires before the
subclass object exists, so trails' `registerSubclass` stays lazy and
`Inheritance#subclasses` unions two hand-filled registries. Needs an owner
decision on the second-best shape (proxied-`Base` get trap, lint/codegen-enforced
registration at every `extends`, or ratifying the union), not more measurement.

### 3. A third-party driver cannot express the Rails call (1 story)

`mysql2-execute-batch-routes-through-raw-execute`. The body converged in PR 6913; the **install** cannot land because node-mysql2 ships no command class for
`COM_SET_OPTION` (`lib/constants/commands.js:31` defines `SET_OPTION: 0x1b`,
`lib/commands/` has no `set_option.js`), so Rails' per-batch `set_server_option`
(`mysql2/database_statements.rb:41-45`) is unsendable and a combined `"a;\nb"`
fails `ER_PARSE_ERROR` on mariadb:11. Unblocks on an upstream driver change or a
decision about the alternatives the story enumerates — enabling
`multipleStatements` globally is explicitly not one of them.

### 4. The story's own premise needs a respec (1 story)

`column-coder-carries-a-class-tag-and-subclass-state`. Its acceptance criteria as
written would **diverge** from Rails: `postgresql/column.rb:50-61` and
`sqlite3/column.rb:36-44` do define `init_with`/`encode_with`, so "carry no
`encodeWith`/`initWith` override" is not Rails' shape. The residual real
deviations are the JSON dump's `class` coder key and trails holding `oid`/`fmod`
/`extra` on `Column` where Rails holds them in the adapter `TypeMetadata` — and
the second cannot move until the fixtures warm stops comparing a dump-loaded
cache against a reflected one. Rewrite against the actual Rails source before
anyone claims it.

### 5. Parity-tooling prerequisites (2 stories)

- `align-collect-calls-filter-with-ruby-extractor` — `calls` is not only the
  compared call set, it is also the edge set the same-file closure walks
  (`compare.ts:538-585`), so applying the Ruby extractor's `_`-prefix filter
  severs every `this._helper()` closure edge: 92 new rows measured, all closure
  false positives. Needs a companion design that applies the filter after the
  closure is computed, or emits closure edges separately.
- `naming-gate-flip` — carried from 0096. Gates the `naming` dimension for
  AR-closure packages once the in-closure convergeable count reaches zero.

## Note on `naming-gate-flip`'s blocker

Its recorded blocker names the wave-5 band (`wave-5-naming-activesupport`,
`-ar-model-core`, `-ar-adapters`, `-ar-associations`, `-ar-relation`, `-tail`) as
the precondition. **All six are now `done`**, so the blocker reads stale on its
face. It was deliberately NOT unblocked during the 2026-08-25 sweep: the
precondition is a _measured_ count ("the in-closure convergeable count reaches
zero"), not a dep-list checkbox, and the count has not been re-measured since the
2026-08-21 artifact. Re-run
`pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls && pnpm parity:api:calls:args:report`
before touching its status; if the AR-closure `burndown` +
`module-mixin-receiver` count is zero, `pnpm tasks status-set naming-gate-flip ready`.

Two other stories carry partially-superseded blockers worth reading before
claiming:
`converge-schema-invalidation-onto-push-only-eager-registration` has two of its
five ACs already checked off by PR 6809, and its AC4 may already be satisfied;
`mysql2-execute-batch-routes-through-raw-execute` has its body converged and only
the install outstanding.
