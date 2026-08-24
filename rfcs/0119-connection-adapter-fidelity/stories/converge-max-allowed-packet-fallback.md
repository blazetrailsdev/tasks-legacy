---
title: "Drop the 16 MiB max_allowed_packet fallback and receiver-dispatch branch"
status: draft
updated: 2026-08-03
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`MySQL::DatabaseStatements#max_allowed_packet`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/database_statements.rb:89-90`)
is `@max_allowed_packet ||= show_variable("max_allowed_packet")` — no fallback,
because `show_variable` always answers on a live MySQL connection.

The trails port
(`packages/activerecord/src/connection-adapters/mysql/database-statements.ts:165-175`,
shipped in #5957) falls back to the 16 MiB MySQL default when `showVariable` is
absent or unparseable:

```ts
const resolved = Number.isNaN(parsed) ? 16_777_216 : parsed;
```

The fallback exists only so bare hosts (unit-test doubles that implement
`MaxAllowedPacketHost` without a connection) can call the function. On a real
adapter it is dead code that would silently mask a broken `showVariable` —
Rails would surface `nil` and fail loudly instead.

The same shape appears in the sibling `isMaxAllowedPacketReached` receiver
dispatch (`database-statements.ts:151-153`), which branches to the free function
for hosts lacking the mixed-in accessor.

## Acceptance criteria

- Decide whether the test doubles can carry a real `maxAllowedPacket`/
  `showVariable` so the fallback and the receiver-dispatch branch can both be
  deleted, converging on Rails' single `show_variable` path.
- If the fallback must stay, narrow it so a live adapter whose `showVariable`
  fails raises rather than silently using 16 MiB.
