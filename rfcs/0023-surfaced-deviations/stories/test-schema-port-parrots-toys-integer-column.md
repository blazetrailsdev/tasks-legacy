---
title: "TEST_SCHEMA omits the canonical parrots.integer / toys.integer columns"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:schema` (PR #4966) reports two UNPORTED-COLUMN findings — the
only column-level divergences across the 242 shared tables:

- `parrots.integer`
- `toys.integer`

Both come from a genuine Rails quirk, not a parser artifact.
`vendor/rails/activerecord/test/schema/schema.rb:1225` declares
`t.integer :pet_id, :integer` inside `create_table :toys` (and the same shape
appears for `parrots`). Rails' `t.integer` accepts a varargs list of column
names, so that line declares **two** columns: `pet_id` and one literally named
`integer`. It reads like a typo for `t.integer :pet_id` but Rails executes it
as written, so the canonical tables really do carry an `integer` column.

`packages/activerecord/src/test-helpers/test-schema.ts` omits both, so the
mirror is incomplete. Nothing depends on the column today, which is why this is
low priority rather than a bug — but the whole premise of TEST_SCHEMA is that
it mirrors schema.rb exactly, and the gate now measures that.

## Acceptance criteria

- TEST_SCHEMA declares the `integer` column on `parrots` and `toys`, matching
  the Rails type and options.
- `pnpm parity:schema` reports zero UNPORTED-COLUMN findings.
- Confirm no existing test asserts an exact column list for either table before
  changing them; if one does, update it to the canonical shape rather than
  skipping the port.
