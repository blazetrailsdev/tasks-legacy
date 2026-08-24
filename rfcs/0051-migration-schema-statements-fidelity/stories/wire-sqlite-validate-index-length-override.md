---
title: "Wire the dead SQLite validate_index_length! override onto the adapter"
status: closed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Delivered. The override is no longer dead: sqlite3/schema-statements.ts:208 exports validateIndexLengthBang (delegating via AbstractSchemaStatements.prototype.validateIndexLengthBang.call at :215), sqlite3-adapter.ts:32 imports it as sqliteValidateIndexLengthBang and installs it as override validateIndexLengthBang(tableName, newName, internal = false) at :2185 with the Rails cite at :2177. The dispatch site abstract/schema-statements.ts:1832 passes options.internal through, so SQLite now gets Rails' internal exemption."
---

## Context

Surfaced while reviewing PR #5560 (`port-migration-columns-cases`).

`validateIndexLengthBang` in
`packages/activerecord/src/connection-adapters/sqlite3/schema-statements.ts:217`
is the port of Rails'
`ActiveRecord::ConnectionAdapters::SQLite3::SchemaStatements#validate_index_length!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/schema_statements.rb:139-141`).
In Rails that override is live: `add_index_options` calls
`validate_index_length!` (`abstract/schema_statements.rb:1484`) and SQLite's
override narrows it to `super unless internal`, so internal index names skip the
length check while user-supplied ones still raise.

In trails the function is **module-private and unreferenced** — not exported, and
never assigned onto `SQLite3Adapter` the way the other SQLite schema-statement
functions are. `this.validateIndexLengthBang(...)` at
`abstract/schema-statements.ts:1996` therefore resolves to the abstract
implementation on the SQLite lane. Verified empirically during the PR review: a
probe on the sqlite3 lane printed the resolved method body (the abstract one) and
confirmed `addIndex("t", "a", { name: "z".repeat(200) })` still rejects with
`/too long/`.

Consequence today: SQLite does **not** get Rails' `internal` exemption — an
internal index with an overlong generated name raises where Rails would let it
through. The behaviour is currently stricter than Rails, not looser, which is why
nothing fails.

PR #5560 corrected the override's _body_ (it previously returned
unconditionally, dropping the check for every index rather than only internal
ones) so that wiring it up later is safe, but deliberately did not wire it —
that is a behaviour change beyond a test-porting PR.

## Acceptance criteria

- [ ] `validateIndexLengthBang` is wired onto `SQLite3Adapter` the same way the
      sibling SQLite schema-statement functions are, so `add_index_options`
      dispatches to it on the SQLite lane.
- [ ] Behaviour matches Rails: internal index names skip the length check,
      non-internal overlong names still raise `ArgumentError`.
- [ ] A regression test covers both arms and fails on the pre-wiring baseline.
- [ ] Check the same class of problem in the sibling SQLite schema-statement
      functions — confirm none of the others are dead in the same way.
- [ ] Green on all three lanes.
