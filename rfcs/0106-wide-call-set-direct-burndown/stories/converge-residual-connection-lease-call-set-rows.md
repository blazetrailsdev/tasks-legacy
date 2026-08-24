---
title: "converge-residual-connection-lease-call-set-rows"
status: ready
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Eight residual `kind: "set"` rows in
`scripts/api-compare/call-mismatches-exclude/activerecord/**` delegate to RFC
0073 (permanent-connection-checkout) but no 0073 story names them, so nobody is
retiring them. RFC 0106's charter is to converge any row in its scope regardless
of which RFC owns the underlying defect, so they are filed here.

| shard                                                      | ruby method                    | missing call          | Rails                            |
| ---------------------------------------------------------- | ------------------------------ | --------------------- | -------------------------------- |
| `connection-adapters/abstract/connection-pool.json`        | `checkout`                     | `acquire_connection`  | `connection_pool.rb:548`         |
| `connection-adapters/abstract/connection-pool.json`        | `checkout`                     | `lock`                | `connection_pool.rb:550`         |
| `connection-adapters/abstract/connection-pool.json`        | `clear_reloadable_connections` | `checkin`             | `connection_pool.rb:510-511`     |
| `connection-adapters/abstract/connection-pool.json`        | `disconnect`                   | `checkin`             | `connection_pool.rb:457-458`     |
| `connection-adapters/abstract/connection-pool.json`        | `pin_connection!`              | `checkout`            | `connection_pool.rb` pinned path |
| `connection-adapters/abstract/connection-pool/reaper.json` | `register_pool`                | `spawn_thread`        | `connection_pool/reaper.rb:59`   |
| `connection-adapters/postgresql/quoting.json`              | `quote_string`                 | `with_raw_connection` | `postgresql/quoting.rb`          |
| `connection-adapters/transactions.json`                    | `add_to_transaction`           | `with_connection`     | `transactions.rb`                |

Every one is the same mechanism: the Rails body scopes a lease (or a Monitor, or
a Thread) around work that trails performs synchronously, because trails'
`checkout`/`checkin`/`withConnection`/`withRawConnection` are all async and the
calling paths are not. `converge-connection-pool-checkout-lease-async` (done)
flipped `checkout` to async; the callers listed above were not carried with it.

`type-caster/connection.json` `type_for_attribute -> with_connection` is the same
class but is already owned by 0073's ready story
`typecaster-connection-drops-datasource-gate-and-with-connection`; it is out of
scope here.

## Acceptance criteria

- [ ] Each row above is either converged (the TS body calls what the Rails body
      calls) or carries an honestly classified `@missingRailsCall` receipt at the
      call site — PERMANENT only where no TypeScript spelling can exist.
      `register_pool -> spawn_thread` is the one strong PERMANENT candidate (JS
      has no threads; the Reaper drives its period from a timer).
- [ ] Converged rows are deleted from the exclude tree by hand; stale high-water
      marks are narrowed with `pnpm parity:api:calls:tighten <shard>`. No
      `--write`, no reseed, no new rows.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
