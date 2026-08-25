---
rfc: "0077-quoting-binds-fidelity"
title: "Adapter quoting and bind-param fidelity"
status: active
created: 2026-07-26
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
  - "arel"
clusters:
  - "adapters"
priority: 3
---

Extracted from RFC 0023 (surfaced-deviations) triage, 2026-07-26.

Open deviations in the adapter quoting layer (quote_string/quote overrides, invented arms, dead interfaces, class-field vs prototype-method override shape, host contracts) and in bind handling (inline quoting where Rails binds, Attribute unwrapping, temporal/binary bind formatting, non-prepared inline-binds branch). These interlock - several stories name each other as sequencing hazards - so they are collected under one RFC.

Swept 2026-08-09 against origin/main: 2 of 15 stories closed as already converged (the \_qi/\_qt shorthand is gone; both SchemaQuoter object-literal assignment sites are gone). 13 remain.

## Done when

The RFC closes on a deliberate end-to-end verification of the quoting/binds
surface, not on "no open stories left" — the inventory was assembled per-file
from PR-surfaced findings, and at least one story's premise had already gone
stale on main without anyone noticing. All five must hold together:

1. `pnpm parity:api --package activerecord`, read scoped to
   `abstract/quoting.rb`, `mysql/quoting.rb`, `postgresql/quoting.rb` and
   `sqlite3/quoting.rb`: every Ruby method in those files is matched, or has a
   `SKIP_GROUPS` entry carrying a reason.
2. `pnpm parity:api:extra --package activerecord` reports no un-tagged extra name in
   those four files. A surviving `@noRailsEquivalent` needs a reviewed reason.
3. Every `call-mismatches-exclude` row under those four files is enumerated and
   is either deleted (converged) or carries a reason naming the Rails
   `file:line` it deviates from.
4. Emitted SQL is unchanged across the three adapters — the quoting,
   sanitization and schema-creation suites captured before and after.
5. Every story in this RFC is `done` or `blocked` with a specific blocker.

### Verification of 2026-08-09 (PR for `quoting-surface-close-out-verification`)

Points 1, 2 and 4 hold at that commit. Point 3's surviving rows are:

- `abstract/quoting.ts` — `quoted_date` omits `default_timezone`
  (`abstract/quoting.rb:186`); trails reads it via `defaultSqlTimezone()` in
  `abstract/sql-datetime.ts`.
- `mysql/quoting.ts` — `type_cast` omits `default_timezone`
  (`mysql/quoting.rb:100,106`); same indirection.
- `postgresql/quoting.ts` — `quote` omits `check_int_in_range` (alias-target
  artifact), `quote` call ORDER differs, `quote_string` omits
  `with_raw_connection` (`postgresql/quoting.rb:127`), `quote_table_name` omits
  `extract_schema_qualified_name` (`postgresql/quoting.rb:73`).
- `sqlite3/quoting.ts` — `quote_string` omits `quote`
  (`sqlite3/quoting.rb:66`, `::SQLite3::Database.quote`).

`mysql/quoting.ts` lost its `quote` row entirely: Rails has no MySQL `quote`
override, and the abstract `quote` now self-dispatches `quote_string`
(`abstract/quoting.rb:76`), so MySQL inherits it. The RFC stays `active` until
point 5 holds.
