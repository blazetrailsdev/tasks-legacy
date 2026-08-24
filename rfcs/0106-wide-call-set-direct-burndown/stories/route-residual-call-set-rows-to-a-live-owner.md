---
title: "Exit ledger: give each of the 42 residual call-set rows a converged call, a receipt, or a claimable story"
status: done
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps:
  [
    "converge-activesupport-residual-set-rows-to-zero",
    "converge-get-primary-key-schema-cache-call-set-rows",
  ]
deps-rfc: []
est-loc: 200
priority: 9
pr: 7007
claim: "2026-08-24T20:39:36Z"
assignee: "route-residual-call-set-rows-to-a-live-owner"
blocked-by: null
closed-reason: null
---

## Context

RFC 0106 has an unambiguous exit number: **`scripts/api-compare/call-mismatches-exclude/**`reports 0 rows with`kind: "set"`for`activerecord`, `arel`and`activesupport`. That is the number; there is no partial-credit exit.\*\*

Measured against `origin/main`'s exclude tree on **2026-08-24** (static count
over the committed baseline, no build required):

| package         | `kind: "set"` rows | files |
| --------------- | -----------------: | ----: |
| `activerecord`  |             **39** |    23 |
| `activesupport` |              **3** |     2 |
| `arel`          |              **0** |     0 |
| **in scope**    |             **42** |    25 |

Trajectory against the RFC's own table: 6,845 (2026-07-17) → 2,218 (2026-08-03)
→ 1,134 (2026-08-14) → **42 (2026-08-24)**. The waves worked. What is left is
not a wave — it is 42 individually-reasoned rows, and every one of them already
carries a per-site verified reason. The head is gone; there is no longer a
densest file (the maximum is 5 rows, in three separate files).

This story is the RFC's **exit ledger**: walk the residual row by row and give
each one a disposition that is either a converged call or a live, claimable
story somewhere. It exists because the residual has quietly become a
cross-RFC routing problem, and three failure modes are already visible in it.

### The residual, by claimed owner

Read from the `reason` strings in the exclude tree:

| claimed owner                               | rows | notes                                                                                                             |
| ------------------------------------------- | ---: | ----------------------------------------------------------------------------------------------------------------- |
| RFC 0073 (pool checkout / connection lease) |    9 | `connection-pool.ts` ×5, `quoting.ts` `quote_string`, `transactions.ts`, `reaper.ts`, `type-caster/connection.ts` |
| RFC 0094 (sqlite3 construction cluster)     |    5 | all in `connection-adapters/sqlite3-adapter.ts`                                                                   |
| RFC 0023 (surfaced deviations)              |    6 | adapter-layout `columns`/`primary_key` ×3, plus 3 with named stories (see below)                                  |
| RFC 0082 (Ruby idiom class)                 |    2 | `String#limit` byte-truncation, `Hash#fetch` with a raising block — both in `abstract/schema-statements.ts`       |
| RFC 0059 (one-schema convergence)           |    1 | `schema.ts` `define` → `with_connection`                                                                          |
| RFC 0051 (migration / schema-statements)    |    1 | `tasks/mysql-database-tasks.ts` `purge` → `recreate_database`                                                     |
| unported dependency                         |    3 | `ActionDispatch::Request` (`middleware/shard-selector.ts`), `ActiveRecord::Promise` (`statement-cache.ts` ×2)     |
| **no owner named**                          |   12 | see below                                                                                                         |

### Failure mode 1 — a named story is closed while its rows are still live

`database-configurations.ts` `default_env → call` and
`database-configurations/database-config.ts` `for_current_env? → call` both name
`port-connection-handling-default-env-proc` as their tracking story. That story
is **closed**, with reason "merged into
`converge-establish-connection-default-env-funnel`". The successor exists and is
`draft` under RFC 0023 — so the rows are still tracked, but only by a chain the
baseline reason does not name. The reason strings point at a closed story.

Same shape to check for the three `0023` stories named in reasons —
`adapter-not-found-message-should-be-built-inline`,
`pg-cidr-cast-value-should-build-an-ipaddr`,
`legacy-point-serialize-should-extract-number-for-point` — all three are `draft`
under RFC 0023, and RFC 0023 being `draft` means none of them is claimable from
the ready queue. That is exactly the delegation stall this RFC was written to
escape ("Why 0084's model stalled"): a row delegated to a `draft` RFC has nobody
retiring it.

### Failure mode 2 — rows with no owner at all

Twelve rows name no owning RFC and no story. Two of them are being taken by
`converge-get-primary-key-schema-cache-call-set-rows` and three by
`converge-activesupport-residual-set-rows-to-zero` (this story depends on both).
The rest still need a disposition:

- `associations/disable-joins-association-scope.ts` `scope → add_constraints` —
  reviewed (wave 4d); the body IS called, spelled `_addConstraintsDj`, because
  TS rejects a derived declaration of a base-private name (TS2415). This is a
  strong `@missingRailsCall` PERMANENT candidate — check it against
  `add-leading-underscore-call-candidate-to-conventions`, which may retire it
  mechanically.
- `connection-adapters/postgresql/database-statements.ts` ×2 —
  `last_insert_id_result → internal_exec_query` (reached via `queryValue`) and
  `returning_column_values → first` (`Array#first` is not a ported name).
- `connection-adapters/sqlite3/schema-definitions.ts` `change_column → column`.
- `delegated-type.ts` `define_delegated_type_methods → define_method`.
- `inheritance.ts` `discriminate_class_for_record → find_sti_class` (routes
  through `findStiClassForRow`, `inheritance.ts:867`).
- `querying.ts` `_load_from_sql → instantiate_instance_of`.
- `scoping/default.ts` `build_default_scope → any?` (inverted branch shape).

### Failure mode 3 — the conventions fix already landed and did NOT retire these

The RFC's seeded-reason audit found the `"_" + camel` candidate gap and filed
`add-leading-underscore-call-candidate-to-conventions`, which is **done**
(PR #6825). So the cheap mechanical retirement has already been taken. The
survivors are survivors on purpose: `_addConstraintsDj` carries a `Dj` suffix on
top of the underscore, so no bare `"_" + camel` candidate reaches it. Do not
re-litigate the conventions table for these rows — the RFC's Scope section puts
tooling changes to the ratchet out of scope, and a genuinely new conventions
need is a new RFC, not a widened candidate list bolted onto this ledger.

## Acceptance criteria

- [ ] Re-measure from a clean build (`pnpm build`, then
      `API_COMPARE_FORCE=1 pnpm parity:api --calls`) and record the current
      in-scope `kind: "set"` count in this RFC's README changelog. If it differs
      from the 42 measured here on 2026-08-24, say so and say why.
- [ ] Every remaining in-scope row has **one** of: (a) a converged call, (b) a
      `@missingRailsCall` receipt at the call site, honestly classified —
      PERMANENT only where no TypeScript spelling can exist, per CLAUDE.md's "a
      documented deviation is debt, not permission", or (c) a named story that
      is **open and claimable**, with the story slug spelled in the row's reason.
- [ ] No row's reason names a **closed** story. The two `DEFAULT_ENV` rows are
      repointed at `converge-establish-connection-default-env-funnel` (or at
      whatever succeeds it), not at the closed
      `port-connection-handling-default-env-proc`.
- [ ] For each of the six owning RFCs in the table above, confirm the owning RFC
      has a live story covering its rows. Where a row is delegated to a `draft`
      RFC with no story naming it, file the story **here** under RFC 0106 rather
      than leaving it delegated — that is this RFC's stated charter: "converges
      any row in its scope regardless of which RFC owns the underlying defect".
- [ ] For any row whose TS body calls the name under a `_`-prefixed spelling,
      confirm against the landed
      `add-leading-underscore-call-candidate-to-conventions` (done, PR #6825) why
      the existing candidate does not reach it, and record that in the receipt —
      do not widen the conventions table here.
- [ ] Open question 1 in the RFC README (`relation.ts` cross-file mixin
      attribution noise) is answered and marked resolved in the README —
      `relation.ts` is now at 0 rows, so the answer is recoverable from the wave
      history rather than re-measurable.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no
      `--write`, no reseed, no widened allowlist, no new rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] When the count reaches 0, flip RFC 0106 to `closed` with the final measured
      number in the changelog.
