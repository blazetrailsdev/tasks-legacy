---
title: "TimeZone#today should answer a Date, not a {year,month,day} triple"
status: ready
updated: 2026-08-22
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`TimeZone#today` returns a bare `{ year, month, day }` object literal instead of
a `Date`, which forces `tomorrow` and `yesterday` to hand-roll date arithmetic
Rails does not have.

Surfaced by `wave-5c-tail-sweep` (PR #6882): the call-set row
`activesupport/values/time-zone.json` -> `today` -> `to_date` was deliberately
left as a baseline row rather than migrated to a `@missingRailsCall` receipt,
because the deviation is convergeable work, not a language shortcoming — the
row's own reason says "Converging is the `Date` port, not the missing call".
The `Date` port has since landed under RFC 0088 (`port-date-today-and-datetime-now`,
`port-date-sub-today-now-receiver-class` are both done, and that RFC is now closed), so the blocker is gone.

Rails — `activesupport/lib/active_support/values/time_zone.rb:520-534`:

    def today
      tzinfo.now.to_date
    end

    def tomorrow
      today + 1
    end

    def yesterday
      today - 1
    end

trails — `packages/activesupport/src/values/time-zone.ts:1107-1128`:

    today(): { year: number; month: number; day: number } {
      const n = this.now();
      return { year: n.year, month: n.month, day: n.day };
    }

    tomorrow(): { year: number; month: number; day: number } {
      const t = this.today();
      const d = new Date(Date.UTC(t.year, t.month - 1, t.day + 1));
      return { year: d.getUTCFullYear(), month: d.getUTCMonth() + 1, day: d.getUTCDate() };
    }

    yesterday(): { ... }   // same shape with `t.day - 1`

Three separate fidelity misses:

1. `today` answers a triple where Rails answers a `::Date`, and reads it off
   `now()` rather than `tzinfo.now.to_date`.
2. `tomorrow`/`yesterday` are one-liners in Rails (`today + 1` / `today - 1`).
   trails replaces each with a four-line JS `Date` UTC round-trip — invented
   logic that exists only because `today` has no `Date` to add to.
3. The JS `Date.UTC` round-trip is a second date implementation living next to
   the ported `Date` gem.

## Converged shape

    today(): Date {
      return this.tzinfo.now().toDate();
    }

    tomorrow(): Date {
      return this.today().add(1);
    }

    yesterday(): Date {
      return this.today().sub(1);
    }

using the ported `Date` from `@blazetrails/date` and whatever spelling that port
settled on for Ruby `Date#+` / `Date#-` (check the receiver-class stories under
RFC 0088 before picking a name — do not invent one).

## Acceptance criteria

- [ ] `today` returns a ported `Date`, built from `tzinfo.now` via the `to_date`
      analogue rather than read off `now()`.
- [ ] `tomorrow` and `yesterday` are the `today + 1` / `today - 1` one-liners;
      the `Date.UTC` round-trips are deleted.
- [ ] Call sites of all three updated; no `{ year, month, day }` triple survives
      on this seam.
- [ ] The `today` -> `to_date` row is deleted from
      `scripts/api-compare/call-mismatches-exclude/activesupport/values/time-zone.json`
      (only-shrink: delete the row by hand, do not reseed), and
      `pnpm parity:api:calls:tighten activesupport/values/time-zone.json` if the
      mark goes stale.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] Rails test names verbatim; SQLite, PostgreSQL and MySQL/MariaDB lanes green.
