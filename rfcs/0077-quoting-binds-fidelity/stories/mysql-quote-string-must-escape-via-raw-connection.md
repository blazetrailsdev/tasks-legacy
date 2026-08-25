---
title: "MySQL quote_string uses a static JS escaper instead of the driver's connection-aware escape (unsafe under NO_BACKSLASH_ESCAPES and multi-byte charsets)"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: 8
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Baselined in PR #6577 (RFC 0106 wave 3b): row
`quote_string | with_raw_connection` in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`.

`abstract_mysql_adapter.rb:694-699`:

    def quote_string(string)
      with_raw_connection(allow_retry: true, materialize_transactions: false) do |connection|
        connection.escape(string)
      end
    end

Rails delegates escaping to the DRIVER, which escapes per the CONNECTION's
current character set. trails (`abstract-mysql-adapter.ts:1145`) delegates to
`mysql/quoting.ts#quoteString`, a pure-JS backslash escaper.

This is not merely stylistic: `mysql_real_escape_string` changes behaviour
under `NO_BACKSLASH_ESCAPES` sql_mode (backslash stops being an escape
character, and `'` must be doubled instead), and multi-byte charsets such as
big5/gbk/sjis have sequences where a naive byte-wise backslash escape is
unsafe. A static escaper cannot see either condition.

**Why it was baselined:** `withRawConnection` is async; `quoteString(s: string):
string` is the synchronous `Quoting` contract that every `quote()` call site
depends on transitively.

## Converged shape

`quoteString` routes through the driver's `escape` on the checked-out
connection, as Rails does. Options worth weighing before picking one:

- Make the `Quoting` seam async where MySQL needs it (large blast radius —
  measure the caller set first).
- Cache the connection's charset and `NO_BACKSLASH_ESCAPES` flag at
  `configure_connection` time and have the synchronous escaper honour both.
  This is NOT Rails' shape but is the same OUTPUT; it would still need its own
  justification and would not retire the baseline row.

The first is the convergence; the second is a mitigation. Do not close this
story with the second without maintainer sign-off.

## Acceptance criteria

- [ ] `quoteString` mirrors rb:694-699, or the divergence is reduced to a
      documented language shortcoming with the escaping OUTPUT proven
      equivalent under `NO_BACKSLASH_ESCAPES` and a multi-byte charset.
- [ ] Regression tests that FAIL on baseline for both conditions.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Triage note (2026-08-25): the baseline shard in this body is gone

`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract-mysql-adapter.json`
**no longer exists**. The rows this story cites were retired from the baseline
into `@missingRailsCall … PERMANENT` receipts at the call sites in
`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts`
(`with_raw_connection` at :1185). Converging therefore deletes
the **JSDoc receipt**, not a baseline row — and note the receipt currently reads
`PERMANENT`, which this story disputes: it is the tag that has to go, not be
reworded. Everything else here — the Rails and trails `file:line` citations and
the described divergence — is unaffected and was re-verified live on 2026-08-25.
