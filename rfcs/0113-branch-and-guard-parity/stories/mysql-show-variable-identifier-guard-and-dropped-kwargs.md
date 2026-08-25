---
title: "show_variable adds a /^\\w+$/ guard Rails lacks (rejecting @@global.x), drops materialize_transactions/allow_retry, and string-coerces the scalar"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced reading `abstract-mysql-adapter.ts` end-to-end for RFC 0106 wave 3b
(PR #6577). Not a call-set row — the guard is EXTRA code, and the call gate
only sees calls Rails makes that trails omits, never the reverse.

`abstract_mysql_adapter.rb:578-583`:

    def show_variable(name)
      query_value("SELECT @@#{name}", "SCHEMA", materialize_transactions: false, allow_retry: true)
    rescue ActiveRecord::StatementInvalid
      nil
    end

trails (`abstract-mysql-adapter.ts:995`) adds a guard Rails does not have and
drops two kwargs Rails does pass:

    async showVariable(name: string): Promise<string | null> {
      if (!/^\w+$/.test(name)) return null;      // <- no Rails counterpart
      try {
        const val = await this.queryValue(`SELECT @@${name}`, "SCHEMA");
        ...

Three deltas:

1. The `/^\w+$/` guard silently returns `nil` for any name with a dot, which is
   a LEGAL MySQL variable reference — `@@global.max_connections`,
   `@@session.sql_mode`. Rails would run those and return the value; trails
   returns nil as though the variable did not exist. Callers cannot distinguish
   "rejected by the guard" from "server said no".
2. `materialize_transactions: false` and `allow_retry: true` are dropped.
   `show_variable` is called during connection configuration, and Rails
   deliberately keeps it from materializing a lazy transaction; trails' call
   can.
3. `String(val)` coerces — Rails returns the raw `query_value` scalar, so a
   numeric variable comes back as an Integer in Rails and a string here.
   `maxAllowedPacket` (`mysql/database-statements.ts`) depends on the numeric
   reading.

The guard predates the RFC 0106 convergence and was left in place deliberately
rather than removed blind, since removing it changes what reaches the SQL
string.

## Converged shape

    async showVariable(name) {
      try {
        return await this.queryValue(`SELECT @@${name}`, "SCHEMA", {
          materializeTransactions: false, allowRetry: true,
        });
      } catch (e) {
        if (e instanceof StatementInvalid) return null;
        throw e;
      }
    }

Interpolating `name` unescaped is what Rails does — every caller in the tree
passes a literal. If a caller ever passes user input, the fix is at that caller,
not a silent nil here.

## Acceptance criteria

- [ ] `showVariable` mirrors rb:578-583, including both kwargs, with no
      identifier guard.
- [ ] `@@global.x` / `@@session.x` resolve rather than returning nil
      (regression test that FAILS on baseline).
- [ ] The return value is not string-coerced; `maxAllowedPacket` still reads a
      number, and `charset` / `collation` still read strings.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
