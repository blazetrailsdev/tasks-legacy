---
title: "sqlite3 translate_exception mirrors Rails' six arms (missing BusyException, extra ValueTooLong)"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-order
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while landing `converge-translate-exception-cause-kwarg` (PR #6375),
which converged the argument lists in this function but deliberately did not
touch its branch structure.

`SQLite3Adapter#translate_exception`
(`activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:692-710`)
has exactly six arms, in this order:

```ruby
if    exception.message.match?(/(column(s)? .* (is|are) not unique|UNIQUE constraint failed: .*)/i)  # RecordNotUnique
elsif exception.message.match?(/(.* may not be NULL|NOT NULL constraint failed: .*)/i)               # NotNullViolation
elsif exception.message.match?(/FOREIGN KEY constraint failed/i)                                     # InvalidForeignKey
elsif exception.message.match?(/called on a closed database/i)                                       # ConnectionNotEstablished
elsif exception.is_a?(::SQLite3::BusyException)                                                      # StatementTimeout
else  super
end
```

trails' module-level `translateException`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:~3178`)
diverges on three counts:

1. **A `ValueTooLong` arm Rails does not have** —
   `if (msg.includes("String or BLOB exceeded size limit"))`. No counterpart at
   `sqlite3_adapter.rb:692-710`; Rails lets that fall through to `super`.
2. **The `SQLite3::BusyException` → `StatementTimeout` arm is missing**
   (`:705-706`). A busy/locked database therefore surfaces as a bare
   `StatementInvalid` rather than `StatementTimeout`, so callers that
   discriminate on the timeout class cannot.
3. **Each message regex is OR'd with a driver `code?.includes("CONSTRAINT_*")`
   check** that Rails does not perform. Rails matches on the message only.

The `else super` arm is also inlined as a direct `StatementInvalid`
construction rather than dispatching to the inherited translator.

## Converged shape

`translateException` has Rails' six arms, in Rails' order, matching on the
message the way Rails does, with the `BusyException` arm restored (against
whatever better-sqlite3 surfaces for `SQLITE_BUSY`) and the `ValueTooLong` arm
either removed or justified at the call site with a cite showing the driver
cannot reach the Rails arm. The final arm dispatches to the inherited
translator rather than re-constructing `StatementInvalid` inline.

## Acceptance criteria

1. The branch set, branch order and match conditions mirror
   `sqlite3_adapter.rb:692-710`.
2. A busy/locked SQLite database yields `StatementTimeout` (`:705-706`).
3. Dropping the `code?.includes(...)` disjuncts does not regress the adapter
   suites — if a driver code arm is genuinely required because better-sqlite3
   phrases a message differently than the C sqlite3 gem, it stays with a
   one-line call-site justification naming the message it covers.
4. `pnpm parity:api:calls` / `:args` green; SQLite suites green.

## Absorbed: `sqlite-translate-exception-branch-set-parity`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Converge SQLite translateException branch set to Rails"

### Context

Surfaced while converging `isNoDatabaseError` away (PR #5883).
`translateException` in
`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts` diverges from
Rails' `SQLite3Adapter#translate_exception`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:692`)
in two ways:

- trails has a `String or BLOB exceeded size limit` -> `ValueTooLong` branch that
  Rails does not have.
- trails is missing Rails' `exception.is_a?(::SQLite3::BusyException)` ->
  `StatementTimeout` branch (`sqlite3_adapter.rb:706`).

Rails' branch list, in order: unique, not-null, FK, closed-database, busy, else
`super`. trails' order also differs (closed-database sits after the extra
`ValueTooLong` arm).

### Acceptance criteria

- trails' `translateException` branch set and order match
  `sqlite3_adapter.rb:692`-`:709` exactly.
- The `ValueTooLong` arm is removed, or justified at the call site with the
  driver evidence that makes it unavoidable.
- A busy/locked driver error maps to `StatementTimeout`.
- No test name renamed.

## Absorbed: `sqlite3-translate-exception-missing-arms`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "sqlite3 translate_exception: restore the BusyException arm, drop the invented one, delegate the else to super"

### Context

Surfaced while converging the `translate_exception` argument lists in PR #6370.

Rails' `SQLite3Adapter#translate_exception`
(`activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:692-710`)
has six arms, in order: `RecordNotUnique`, `NotNullViolation`,
`InvalidForeignKey`, `ConnectionNotEstablished`, then

```ruby
elsif exception.is_a?(::SQLite3::BusyException)
  StatementTimeout.new(message, sql: sql, binds: binds, connection_pool: @pool)
else
  super
end
```

trails' `translateException`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`, bottom of
file) diverges on both tails:

- **Missing arm.** No `BusyException` → `StatementTimeout` branch, so a lock
  timeout surfaces as a bare `StatementInvalid`. (`sqlite-retries-busy-handler-unported`
  under 0023 is closed and covers the retry/busy-handler machinery, not this
  classification arm.)
- **Extra arm.** A `ValueTooLong` branch on `/String or BLOB exceeded size limit/`
  that `sqlite3_adapter.rb` does not have — Rails reaches `ValueTooLong` only
  through the abstract translator, if at all.
- **No `super`.** The `else` returns `StatementInvalid` directly instead of
  delegating to `AbstractAdapter#translate_exception`
  (`connection_adapters/abstract_adapter.rb`), whose own `case` returns the
  exception unchanged for a `RuntimeError` / `ActiveRecordError` — so a trails
  `ActiveRecordError` raised from the driver callback gets re-wrapped where Rails
  passes it through.

### Converged shape

Six arms, in Rails' order, with the `BusyException` arm restored, the invented
`ValueTooLong` arm removed, and the `else` delegating to the abstract
translator so its `RuntimeError` / `ActiveRecordError` passthrough applies.

### Acceptance criteria

1. `translateException` mirrors `sqlite3_adapter.rb:692-710` arm for arm, in
   order.
2. A SQLite busy/lock error classifies as `StatementTimeout`, covered by a test
   that fails on the pre-change implementation.
3. An `ActiveRecordError` thrown from the driver callback comes back unchanged
   rather than wrapped in `StatementInvalid`.
4. `pnpm parity:api:calls` and the SQLite suites green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
